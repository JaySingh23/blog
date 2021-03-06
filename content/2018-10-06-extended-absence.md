title: Extended Absence
date: 2018-10-06 18:35
categories: python braze ruby rails

For those who have been keeping score, I've been on somewhat of an extended absence, especially since October of last year. Much of it was the result of events in my personal life which I'll not discuss here. Regardless, consider this my "comeback" announcement. I plan to return to writing regularly, hoping to match the pace of 2014 or so, which already seems so long ago. Heck, [Writing Idiomatic Python](https://jeffknupp.com/writing-idiomatic-python-ebook) is three months older than my older daughter and she just started Kindergarten this a few weeks ago. This blog is older than most startups, but it's been a while since I've been producing (I think) useful or interesting content at a reliable pace. I'm going to change that.
<!--more-->

## Slightly Less Personal News

### Braze

At the end of July of this year I joined [Braze (formerly Appboy)](https://www.braze.com) as a Senior Engineer in my return to being an IC (Individual Contributor in management-ese). Management was fun but somewhat thrust upon me. I wanted to take a more measured approach.

In Braze, I couldn't have found a better match. Braze counts some of the biggest brands in the world (Gap, Microsoft, Domino's, ABC News, NASCAR, Lyft, Venmo, HBO, and Burger King are a few, to give you a sense of breadth of industry they work width) as customers. Those brands use Braze's platform to manage their mobile marketing strategies. And though you probably haven't heard of Braze (formerly Appboy) (which is *actually* how the company, for now, refers to itself), they are no small player: [#21 On Deloitte's Annual Fast 500 (the 500 fastest growing technology companies in America)](https://www2.deloitte.com/us/en/pages/technology-media-and-telecommunications/articles/fast500-winners.html), [A "Leader" in the 2018 Gartner Magic Quadrant Survey of Mobile Marketing Platforms (and that's from a field of companies that included Salesforce, IBM, and Oracle)](https://www.braze.com/resources/library/report/gartner-magic-quadrant-2018/), and profiled by publications like Fortune and Inc. They've built an incredible company full of nice, thoughtful people who genuinely care about the product they work on and services they offer.

So what does it do? And what do I now do? I'm glad you asked.

Braze's product is a B2B SaaS product, which is a fancy way of saying a cloud-hosted, web-based app sold directly to businesses and not individuals. Companies that purchase the product have marketing departments with more money and employees than most startups. Of course, they have a number of mediums through which to advertise: TV, outdoor ad space, radio, print media, etc. And, oh yeah, **digital**. You know, the primary way you interact every day with almost any major brand? Suffice it to say the digital marketing is a **big** business.

But how does one go about creating, managing, and implementing their digital marketing strategy? Once they've determined that they should run an email marketing campaign to announce the newest changes to their product, where do they go to actually write the email, choose the users to receive it, send the emails, and measure and analyze engagement data? And wait, a large brand may have a dozen email-based marketing campaigns going at one time with new ones starting all the time. And they also have campaigns they want to run for users of their mobile app using push notifications and in-app messages. And web push/browser campaigns for visitors to their website. They need a platform that allows them to 

A customer can use Braze's platform to create, manage, and actually *run* (i.e. send the emails, push notifications, etc. to the actual end user's device) all of those campaigns. And what about all that data that gets generated, the kind capable of answering questions like "who bought something (and what did they buy) within 3 days of receiving this notification on their phone?" or "what's the most effective messaging channel to reach my female customers from Canada who are over 35 and have used my mobile app at least three times in the past week"? All of that is collected, analyzed, presented, and ultimately fed back into the product itself, as well as to the company itself.

Adding features to that web application and owning the backend platform, which is responsible for actually sending all those messages and managing the life-cycles of automated marketing campaigns, is the responsibility of my team. And it's all in Ruby on Rails and in JavaScript. No, I'm not joking. I'm working on an RoR app (including frontend development), having never used Ruby, as a guy who's known as a "Python Guy". **And I couldn't be happier.**

What the engineering team at Braze has been able to accomplish using RoR, mongo, redis, memcached, and [sidekiq](https://sidekiq.org/) (a distributed task execution framework along the same lines as Celery for you Python folk, which Braze co-founder and CTO [Jon Hyman](https://github.com/jonhyman) happens to be one of the main contributors to), is nothing short of remarkable. Aside from a few Go microservices acting as the last mile before sending *billions of messages a month* and some Java for data processing and integrations, the entire platform is a single Rails app. And they've been able to scale that app in ways that would make John Carmack smile. In terms of sheer amount of storage and compute power used, it's the largest distributed system I've ever worked on (algorithmic trading systems, which I started my career building at Goldman Sachs in 2005, were *somewhat* distributed back then but more often just sharded instances of the same application that didn't share any data).

But then, it's also 2018.

Microservices developed and deployed at the speed and scale we see today simply weren't feasible until fairly recently. There was a sticky problem: orchestration. Before the orchestration tools we have today existed, the increase in maintenance and support effort to deploy a new service was quite high in most organizations. But almost no effort is required to do so with today's orchestration tools. At whiteboards around the world, engineers are discussing ways to break up their monolithic application into a series of disparate services with focused responsibilities and well-defined interfaces. And Braze is no different.

As an engineer who loves working on distributed systems, there are two really fun times to work on them: when they are first being built or when they've hit an inflection point and need to scale in ways that require rethinking *everything*. And "scale" here doesn't just mean "process more data". It also means "support concurrent development by dozens of developers" and "decrease our maintenance and support burden". Those are the opportunities I live for, and that's why I'm so excited to be at Braze.

### Ever Onward

Yes, yes, I know. "The Python guy is working on a Ruby on Rails app?". To be completely honest, it's only been in the past few years have I used it professionally. So in terms of my earlier "comeback" announcement, the fact that I'm not writing Python at work doesn't mean I'm not writing it, and I do and will have plenty to of topics to discuss on all things Python. Because Python is reaching an inflection point as well, and learning Python concepts and how to think about the writing code has never been a hotter (nor more important) topic.

So yeah, my life has changed a lot. And even calling myself "The Python guy" is a stretch at this point. I haven't produced a lot in the last few years. But I miss that. And I aim to change it. For those of you who emailed or tweeted me while the site was down last month, thank you. It's up now and staying up. And you're going to be finding yourself on it more often now.

Time to get to work...
