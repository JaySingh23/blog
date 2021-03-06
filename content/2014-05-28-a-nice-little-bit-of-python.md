title: A Nice Little Bit of Python
date: 2014-05-28 15:09
categories: python iyp

The blog has been relatively quiet recently due to my many irons and fires and
all that, but I did want to take a second and post a simple bit of code which
really appeals to me. It solves a common problem in a reasonably elegant way and
is straightforward enough for Python programmers of any level to use.

<!--more-->
The best way to show it is to show an example of the type of problem it solves.
Say we have a string and want to make sure it ends with one of four sub-strings.
Here is the usual method of checking this:

    #!py
    if needle.endswith('ly') or needle.endswith('ed') or 
        needle.endswith('ing') or needle.endswith('ers'):
        print('Is valid')
    else:
        print('Invalid')

Ugly, right? If we were just checking if the string `needle` *was equal to* one
of those values we could use this common idiom:

    #!py
    if needle in ('ly', 'ed', 'ing', 'ers'):
        print('Is valid')
    else:
        print('Invalid')

Alas, we can't use `in` when a function call like `endswith` is involved. When
we say exactly what we want to check aloud, however, the more elegant solution
becomes obvious:

    I want to check if the string ends with any of the given sub-strings

The key word in there is `any`, which happens to be a Python built-in. Instead
of the overly clunky method of checking `endswith` used above, why not take a
page from the idiomatic set-membership check:

    #!py
    if any([needle.endswith(e) for e in ('ly', 'ed', 'ing', 'ers')]):
        print('Is valid')
    else:
        print('Invalid')

Now, some readers might be disappointed here, having expected a far more
earth-shattering revelation. And that's OK. I'm sure a large portion of you
already use a method similar to this when faced with the same situation. The
interesting bit was that I was able to *reason it out* given my knowledge of
existing idioms.

I didn't read it in someone else's blog post or while reading some library's
code, I reasoned it out on my own. Now of course it existed before I discovered
it and is old hat for many, but the point is that I discovered it without being
shown it. And I was able to do so because of my knowledge of Python idioms.

So that's that. Stay tuned for a number of posts I have in the works (some more
controversial than others) in addition to the videos for the Kickstarter
campaign. I'm still giving away free books for women in STEM, so email me if
that fits your description and you'd like a copy of *Writing Idiomatic Python*.
