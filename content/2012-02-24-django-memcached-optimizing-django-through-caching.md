title: Django Memcached: Optimizing Django Through Caching
date: 2012-02-24 09:00
categories: django optimization memcached python linkrdr


Caching is a subject near and dear to the heart of many
performance-minded programmers. For those coming to web programming
without other programming experience, caching may be a new topic. For
programmers new to the web, using an external cache may be an approach
not yet considered. In this post, I'll describe how, through the use of Django's caching support, I was able to __reduce [linkrdr's](http://www.linkrdr.com) page load time from over 3.5 seconds to 0.01 seconds.__
<!--more-->

What is Caching?
-----------------

Caching is a word that changes meaning a bit depending on the context in which it's used, but in the general sense, _caching is the process by which the result of previous computation is saved and reused without re-performing the computation._ In the Django specific sense, there are three different types of caching:

1.    __The per-site cache__: Saves the result of requests to all URLs 
      for reuse when a request for the same URL is later made.

2.    __The per-view cache__: Saves the result of requests resolving to
      specified views for reuse when a request resolving to the same
      view is made.

3.    __The low-level cache API__: Used by the developer to set, retrieve, and maintain objects in the cache manually.

While the first two methods are certainly useful for some sites, the third is the most interesting. Caching requires some back-end cache (obviously), and Django ships with a few out of the box. It can keep the cache in memory (not particularly useful when new threads are spawned for HTTP requests by the HTTP server), in the database (fine but slow), or on the filesystem (ditto).

Then there's [Memcached](http://memcached.org). Memcached is pretty well known in the web development community as an extremely flexible, scalable, and resilient external caching solution. The fact that Django ships with a "low-level" API for interacting with Memcached is great, although it does require the installation of a Python/Memcached interface package, of which there are two: python-memcached and pylibmc. Let's investigate using Memcached via a case study.

Case Study: linkrdr.com Page Load Time Optimization
--------------------------------------------------

I've talked about [linkrdr's](http://www.linkrdr.com) optimization
techniques before. The main view shows links from entries in a
user's subscribed feeds. These links have been aggregated and sorted according to a ranking algorithm. For example, if a user subscribes to 10 RSS feeds and follows a few Twitter users and 4 of those sources mention a particular link, that link should appear higher in your list of links to read than a link from a single feed entry.

Retrieving, analyzing, aggregating, and sorting all of the links from all of the entries from all of a user's subscribed feeds is computationally expensive. The calculation of a link's "importance" was optimized by [rewriting the calculation in C++ and calling it from the view](http://www.jeffknupp.com/blog/2012/02/15/optimizing-django-views-with-c-plus-plus/). The database query [was similarly optimized](http://www.jeffknupp.com/blog/2012/02/14/profiling-django-applications/). These proved not to be enough, however, in the face of a flood of new users (and new data) to linkrdr.

3.5 seconds. That was how long it took to generate the data within the
main view function. Previously, it had been under a second, but after a
sudden flood of users and data, things had gotten out of control. I knew that I needed to implement caching to cut down page load time, but there were a few wrinkles. First, the data sets linkrdr returns are rather large, even with pagination reducing the items shown to 100 per page. Second, a link had two real scores: one that could be calculated just by inspecting the link's properties in isolation and one that could only be calculated with respect to __all__ the other links in a user's feed. These issues combined led to an interesting optimization problem.

The first goal was to cache all the user's sorted links whenever a
request came in, to be used in subsequent requests. I originally tried
to cache the entire result set like so:

    #!py
    def view(request):
        results = cache.get(request.user.id)
        if not results:
            results = do_a_ton_of_work()
            cache.set(request.user.id, results)
        # ...

Each time I tried this the results weren't added to the cache. After about 15 minutes of digging, I realized that the data set was larger than memcached's limit for record sizes. When that happens, the call to `cache.set()` (maddeningly) fails silently. 
 
So I couldn't cache the whole thing, but this turned out to be a useful
exercise. I realized I didn't need to cache the entire set as one big value. I could chunk the data set in the same size chunks as were being paginated (currently 100 links). So now, the code looks like this:

    #!py
    def view(request):
        page = get_page(request) # for pagination

        results = cache.get(request.user.id+str(page-1*100))
        if not results:
            results = do_a_ton_of_work()
            for index in range(0, len(results), 100)
                cache.set(request.user.id+str(index), results[index:index+99])
        # ...

Cache Eviction
-------------------

One thing to remember is that caches are not some magical never-ending source of storage. Memcached stores items in memory, of which it has a limited supply (limited by you, that is). When a cache is using the full amount of memory allotted and gets a request to add something new, a choice has to be made. This choice is known as the cache's _eviction policy_, since a record is about to be 'evicted' from the cache. Memcached uses a variant of LRU, or Least Recently Used, eviction. Glossing over the topic of page allocation and sizing, which you can read about on Memcached's wiki, the cache looks for the least recently used item and overwrites it with the new value. This approach has good _temporal locality_, since something that was used recently (especially in web programming), is likely to be used again soon. Things not used for a while are less likely to be used again soon, and thus are good candidates to remove from the cache. For linkrdr, this has a few useful side effects. The view's cached items are likely to be used soon after they are created (as a user browses their links) and then not used at all (when they leave the site), which perfectly matches the cache eviction policy. Also, using small chunks instead of the whole data set allows eviction with more granularity, so a user's entire cache isn't lost all at once.

Which is Nice, Except...
-------------------------

But there are some problems with this approach. For one, the first (uncached) request still takes the full 3.5 seconds. The first request being the most important one, this is not ideal. Second, we have to deal with updates to the data set that would require a recalculating of the scores of each of the links (for example, when a user adds a new feed or linkrdr is updating feed items). Using some additional features of Memcached, we can overcome these obstacles as well.  

Let's deal with updates to the data set first. Django supports "versioning" of cache records. If you specify a version number in your `cache.set(key, value, version=my_version)` call, then a corresponding `cache.get(key, version=some_other_version)` call will not return any data. Using versioning, we can change things around and store the user's current 'version' in the cache. When we want to get the cached data set, we specify the user's cached version number. In this way, we are able to _invalidate_ old cache entries without searching through the cache for all of a user's cached items. An example will help clarify:

    #!py
    def view(request):
        version = cache.get(request.user.id)
        if not version:
            version = 1
            cache.set(request.user.id, version)
        page = get_page(request) # for pagination

        results = cache.get(request.user.id+str(page-1*100), version=version)
        if not results:
            results = do_a_ton_of_work()
            for index in range(0, len(results), 100)
                cache.set(request.user.id+str(index), results[index:index+99], version=version)
        # ...

Now, when a user adds a feed and the link scores need to be recalculated, we can simply increment his or her version so that the next `cache.get()` will be a cache miss.

But wait, I just said we _want_ a cache miss. That can't be right. What we'd _really_ like is for the links to be recalculated _and cached_ and the old values invalidated. To accomplish this without interrupting the user (remember recalculating takes a couple seconds), we fire off a Celery task to asynchronously recalculate the results, update the cache, and update the user's version. This way, the user can continue using the site and, if they take longer than 3-4 seconds between adding a feed and looking at their new links, we'll have the results _prefetched_. The idea of prefetching is a powerful one, and it solves the other problem (the first visit to a page results in a cache miss) as well. When linkrdr updates the feeds, it also caches the results for frequent users of the site (which it determines using yet more cached data). This way, the people who use the site the most experience instant page loads. Infrequent users will experience at most one slow page load.

It would be nice if we could prefetch every user's data after every update, but linkrdr has neither the memory nor computing power to do this. In a way, this is a good thing, because throwing more hardware at a problem doesn't require much thought. Many of the interesting technical challenges linkrdr faces come about because of the constraints on memory and CPU availability. Since I'm running on a single VPS, I have to get a bit creative with some of my solutions. If I didn't have to consider these things, the site would likely be worse for it and would certainly be less interesting to work on. The current caching solution can scale with the site without straining resources too much.

The preceding is [linkrdr](http://www.linkrdr.com)'s current approach to caching. One interesting side note: I'm currently investigating running memcached in a distributed manner using the other VPS I use to host [IllestRhyme](http://www.illestrhyme.com). Since the latter site has far more spare CPU cycles, computation can be partially offloaded to the second machine and the results stored in a distributed cache (which Memcached and Django helpfully support out of the box). Additionally, sprinkling in other types of caching (like file based caching) may make it possible to prefetch all data for all users.

Questions or comments on _Django Memcached: Optimizing Django Through Caching_? Let me know in the comments below. Also, [follow me on Twitter](http://www.twitter.com/jeffknupp) to see all of my blog posts and updates.
