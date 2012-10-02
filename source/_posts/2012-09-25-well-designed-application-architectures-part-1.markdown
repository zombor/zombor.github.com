---
layout: post
title: "Well Designed Application Architectures - Part 1 - MVC"
date: 2012-09-25 16:06
comments: true
categories: [Architecture, Design Patterns]
---

I've been dumping my brain about this topic in IRC for about 3-4 months now, and I figure it would just be easiest to do a dump here and link people to it. This series will probably be a few parts, I'm just going to do parts of the dump for a few days.

Before I begin, you should watch this (I'm basically going to be rehashing it and adding my own notes): <iframe width="640" height="360" src="http://www.youtube.com/embed/WpkDN78P884?feature=player_detailpage" frameborder="0" allowfullscreen></iframe>

## MVC

So, lets start with MVC. Web-based MVC (yes, not the "real" kind). "What's wrong with MVC?", you might ask? Well, nothing, as long as you use it for it's strengths: as a delivery mechanism pattern. We'll get into what a delivery mechanism is in a bit. So, how do you use MVC effectively? Well, lets start with the different parts of the acronym:

### Controller

Your controllers are thin, right? Do they contain *only* http logic? If they contain any business logic at all, they are not thin. Just because it's not database calls doesn't mean you can throw it in the controller. The controller shouldn't know anything about your domain logic. It should just know:

  - How to parse incoming request data
  - How to send that data to your business classes
  - How to send the output from the business classes to the view

That's *it*! What do you gain from doing this?

  - Your controllers are even thinner
  - Your controllers are probably testable in isolation, so you can test your http interactions quickly, and without a real web server or framework loading up

### Views

When I say "view", does "class" also come into mind? Are you doing all your presentation logic in your templates? Stop it! You can't test that. Use something like [Mustache](http://mustache.github.com/). What do you gain by using Mustache?

  - Logicless templates
  - Testable views
  - Portable templates (you can reuse your same templates in both your server and client).

Logicless templates let you start your views with tests (You do TDD, right?). It keeps your views more maintainable and lets you be more confident that the logic contained in them actually *works*.

When you write your view classes, they should only accept input from the outside world. They should not be reaching into the database, they should not be instantiating any classes (especially not models!). All they should be doing is accepting input from the controller and formatting that input for display. This display can be anything: html, json, xml, etc. This is another advantage of using mustache: it lets you easily adjust your output template, and not change your logic that parses the data. Reuse people!

### Model

Wow, what a big area. Basically anything that isn't a controller or view goes here. That is A LOT. This is the first problem with using MVC 100% to design your application. If you subscribe to the "thin controller, fat model" approach (as just about all web frameworks out there teach you), you *will* end up with an unmaintainable mess of an application.

Why? Well, first off, you'll be combining your domain logic (since your controllers are thin, right?) and your persistance logic (just about every framework tells you to put queries in a model) in the same layer. This is very bad for many reasons:

  - Hard to test
  - Slow to test
  - Tests will do too much

A lot of the topics in this series will revolve around testing. How can you make your application easier to test? How can you *start* with tests?

So what can we do? Easy: Don't use models. Just don't. Models don't belong in your MVC delivery mechanism. In this sense, I suppose I should be calling it CVT (Controller-View-Template).

## Delivery Mechanism

What's this "delivery mechanism" thing, anyway? A delivery mechanism is:

*A framework of classes that easily let you redistribute your business logic (your application) in a format specific way, over a specific medium.*

Your application is not made up of controllers and views. That stuff is all part of your delivery mechanism. So from here on out, I'll be refering to your *business logic* as your *application*.

You *do* need these things. You don't want to be writing http request and response classes. That crap is already done, and it's a lot of work.

A delivery mechanism's sole purpose is to call your application, and send it's output to a specific format. It could be an html website, a JSON based REST API, or a console based binary. All three of these could be delivering the exact same business logic. All three of these could be different frameworks, because your application is completely decoupled from your delivery mechanism.

Your delivery mechanism should not know anything about your business rules, except how to call the application. It *consumes* your application, and spits out the data.

Ever cry when your favorite framework released an upgrade, and it broke the old API? Separate your application from your delivery mechanism, and upgrading to a new version (or a completely different) framework becomes much easier. The framework code no longer has it's slimy tentacles embeded inside *your* code. That means less code to change.

This brings me to a related topic: You should try like hell to never have your business logic classes extend a third party class. Don't let someone else taint your class's API with theirs. You'll just end up in a similar situation to the framework problem above. I'll go into more details on this in a future post.

## Business Logic

"If we can't use models, then were the hell does the business logic go?" You might be asking. Easy:

Far, far, far, far away from your MVC layer. We'll get into the details on this next time.
