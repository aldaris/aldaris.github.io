---
layout: post
title:  "My hunt for a secure web framework"
date:   2019-01-05 19:00:00 +0000
categories: dev rails
---

About 5 months ago I started to work on and off on a pet project of mine that hopefully will simplify my wife's life. As
my wife is a sole trader, she's been busy issuing invoices to her clients, however her existing process is not really
that optimal, and requires various manual steps. Hopefully this will all change once I complete this web application
that will allow her to easily file invoices without much of an administrative work.

Currently the sole trading business is very small, and it doesn't really bring in a whole lot of money, so using a
SaaS service or anything similar is pretty much out of question (as it does not make financial sense).

### Decisions on the technology

One thing was clear to me: I wanted to write an application using up to date and secure web technologies leveraging
server side rendering. By secure I specifically wanted to have:
* support for same-site cookies
* potentially built-in support for XSRF tokens (just in case same-site cookies aren't supported by the browser)
* No unsafe-eval/unsafe-inline requirements in CSP

With this very simple goal in mind, I've started to look for a Java based web framework that would qualify the list
above. I soon had to realize that server side web frameworks in Java land are mostly out of date when it comes to
security practices. Let me explain why..

#### Component based web frameworks

I've worked with Apache Wicket before, and I just loved it, a component based web framework that is relatively easy to
work with, so it was clear that I should start my investigations there... It turned out that the main issue with
component based frameworks is that they usually ship with their own magical JavaScript utilities (to implement Ajax
updates) that is usually based on a well known JavaScript library (jQuery, etc.). While these home grown JavaScript
utils usually get the job done, they also tend to be a great source of technical debt, and as such are out of date when
it comes to security.

Here's a couple of example issues with regards to CSP support:
* [Primefaces issue]
* [Vaadin issue]
* [Wicket issue]

If one wants to use out of the box available Ajax components then chances are unsafe-eval and/or unsafe-inline is going
to be a requirement in your CSP with these frameworks.

N.B.: I've been also told that Wicket is essentially in active maintenance mode and it isn't really receiving new
features lately (I guess everyone is writing JavaScript based single page applications).

#### Plain old Java EE

With good old servlets and JSPs I would have definitely covered the CSP issues, but in that case the process of
development becomes very much cumbersome and will require lots of boilerplate code. Other than that I would have had to
come up with my own CSRF token implementation, which wasn't idillic either.

The last nail in the coffin was when I realized that the servlet API just can't keep up with modern technologies. The
JavaEE API for Cookies still does not support same-site cookies, which means that the containers most likely won't
support the flag for the session cookies either (custom session management and manual set-cookie header generation is
not a fun way to overcome this).

#### Other candidates

Amongst the other framework candidates (Grails, Play!, Spring MVC + Thymeleaf), Play! looked the best from security
point of view, but in the end I couldn't really get accustomed to the Scala language, and various StackOverflow articles
have been suggesting that the Java version of it isn't as powerful. All in all I've been stuffed, I had no idea what to
actually use for my application, so I started to look for something else than Java.

Back in the good old days, I did a semester at the uni on Ruby on Rails, and had a relatively productive time. Working
with Rails just felt much more rewarding because of the quicker pace of development. When working with Java based
frameworks it can take ages to figure out how to do hot-deployment based development (it's also dependent on which
application server you use), and even then it usually feels a bit clunky (or requires usage of specific tools).

As I started to look into Rails's approach to security I was just speechless:
* Same-site cookies and CSP was supported out of the box.
* There is a whole [section of documentation] that just details the different security issue types and how one can
 easily address each one of them. The document is just so vast (and *actually* useful), I still go back to it at times.

As an added plus there are many plugins out there for Rails, which should help with the development process a bit (for
example installing the [devise] plugin should make authentication/2fa/self registration a lot simpler than rolling my
own).

#### The outcome

In the end I've made a very much biased decision on using Ruby on Rails for my next application, and I've been having
a pretty good time with it so far. Of course I had a few difficulties on the road (most to do with my inexperience with
both Ruby and Rails), but I hadn't really hit a complete brick wall just yet.

[Vaadin issue]: https://github.com/vaadin/framework/issues/5266
[Wicket issue]: https://issues.apache.org/jira/browse/WICKET-5406
[Primefaces issue]: https://github.com/primefaces/primefaces/issues/3505
[section of documentation]: https://guides.rubyonrails.org/security.html
[devise]: https://github.com/plataformatec/devise
