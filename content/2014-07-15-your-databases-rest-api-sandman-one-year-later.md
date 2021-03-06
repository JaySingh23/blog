title: Your Database's REST API: Sandman One Year Later
date: 2014-07-15 13:22
categories: python sandman rest

A year ago yesterday, I made my first commit to what would eventually become
[sandman](http://www.github.com/jeffknupp/sandman) ( [official homepage](http://www.sandman.io) ).
It remains my most popular Python project by a wide margin. Why? Because it solves
a *real* problem that I've faced many times. July 14, 2013 was just the day I
decided to do something about it. And so, as I sit in my hotel at the University
of Pennsylvania writing this post, preparing to give a 90 minute talk on Python,
REST APIs, and `sandman`, I'm struck by how far the project has come in a year
and the effect it has had on me.
<!--more-->

## Sand*what*?

If you've never heard of `sandman` before, here's what it does in a nutshell: it
provides a binary that you point at a legacy database, hit enter, and browse
your new RESTful API service. And I do mean "browse". In addition to the normal
JSON over HTTP interface, there's a full, Django-style admin page as well,
allowing you to create, edit, delete, search, and sort your data. Remember, this
is all without needing to write *a single line of code.*

Additionally, though, it provides an extensible API that allows you to override
the default behavior of just about everything. It supports authentication,
custom validation and handling requests, and just about everything else I could
think of.

Here's a look at the admin page:

<img src="/static/images/admin_tracks_improved.jpg">

And here's how I got there:

`$ sandmanctl -g mysql+mysqlconnector://localhost/chinook`

That's it. That command creates the admin page and literally opens it in a browser
tab for you. It really doesn't get much easier than that.

## So What?

Although automatic generation of a RESTful API service from a legacy database
sounds like something that *must have* existed before a year ago, it didn't.
Outside of a very small, private company doing something related but different,
there was no way to turn give your legacy database a REST API without writing a
ton of boilerplate ORM code.

But I'm lazy, and it makes me angry when I have to write boilerplate code. So,
after discovering no solution already existed, I wrote one. I tossed it up on
GitHub and blogged about it and watched as it shot up in popularity on GitHub.
It remains one of the top 100 Python projects on GitHub and I have numerous
screenshots of it being the most popular Python repo for a day/week.

## Lasting Benefits

So while it may not sound like much, it definitely scratched an itch that I, and
many other developers had. People took to it quickly, and I'm happy to say that
development continues actively on the project. But the project gave me more than
my 15 minutes of fame. It was the basis for my novel-length (joking...kinda)
post entitled ["Open Sourcing A Python Program the Right Way"](http://www.jeffknupp.com/blog/2013/08/16/open-sourcing-a-python-project-the-right-way/),
which remains a very popular post.

It also fueled my interest in REST APIs, HATEOAS, and the semantic gap (all of
which were the subject of other posts). It also got me familiar with maintaining
an open source project. But most importantly, it got me thinking about programming in
a new way.

Throughout `sandman` (and [`Flask-Sandboy`](http://www.github.com/jeffknupp/flask_sandboy), it's little brother),
the notion of introspection and dynamic class creation abounds. I suddenly began
seeing programming problems in a whole new way, one in which problem
is solved in an extremely general manner once, individual instances of the
problem are handled by dynamically generated solutions. 

This got me thinking more about web frameworks and the impedance mismatch between most web frameworks
and RESTful API creation. That, in turn, got me working on things like
[whizbang](http://www.github.com/jeffknupp/whizbang), where I explore alternate
ways of presenting web frameworks to the user.

In short, it sent me on a wild ride I never could have imagined. As I sit here
in my hotel room, hours away from giving a talk on `sandman`, I'm still amazed
at how much of an impact it has had on me and my career.
