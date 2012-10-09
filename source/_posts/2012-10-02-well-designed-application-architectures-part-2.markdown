---
layout: post
title: "Well Designed Application Architectures - Part 2 - Business Logic"
date: 2012-10-02 06:29
comments: true
categories: [Architecture, Design Patterns]
---

[Last time](/blog/2012/09/25/well-designed-application-architectures-part-1/) we talked about what MVC really is, and how to use it properly. It didn't really mention much of anything about how/where to put the code that makes your site tick. This code is called your *application*. It's the business logic that, you know, actually does the real work on the site.

If you recall from last time, I said that this code should live far, far away from your delivery mechanism (your MVC layer). That's pretty abstract. Where should it *really* go? How do you write it?

A lot of people look at me funny when I say "do not write your business logic with a framework". They think "how on earth will I get anything done?". It's actually very easy, and not time consuming at all if you do it right.

## Language

I'm going to be using PHP for my code examples here. I prefer Ruby, but it seems that it's "harder" to do this in PHP, so I want to show that it's actually not.

## File Organization

First off, put your business logic into it's own repository. This will make the separtion clear and force you to not muck the separation of concerns together. You'll also be able to recognize your application's purpose from it's file structure, as described in the Uncle Bob talk from last time.

People in the PHP world seem to like the [PSR-0](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-0.md) standard for file and class naming so we'll use that, but not the silly PSR-1 and PSR-2 "standards".

# Components

Below are the components that we'll be using to design our application.

## Use Cases

Back in school you learned about a [Use Case](http://en.wikipedia.org/wiki/Use_case), right? Well, they aren't useless after all. Use cases are the backbone of how your application works. We'll be using a use case driven architecture from now on.

Why? They let us define our classes in a structured way, and let any reader of our code know exactly where to go when they need to look at some functionality. Need to know about how a user registers? Simply go to the Register_User class, all the code is there.

### DCI vs Interactors

I prefer to use the [DCI](http://en.wikipedia.org/wiki/Data,_context_and_interaction) pattern for describing my use cases in code. However, it's not clear how to implement DCI in many languages. Because of this, I'll just be describing *interactors*, which are a similar but simpler way to write use cases.

DCI makes *a lot* more sense overall, but in some languages (especially php), it's just impossible to implement properly. You basically would need the ability to assign traits to objects at runtime (at the minimum), and php is certainly not capable of that.

An interactor is just a class that describes your use case. It has the following properties:

  - It has a single public entry point, `execute()`
  - All other methods are protected or private
  - It raises exceptions when an error occurs
  - It returns normalized data (like an array) for a query operation, and returns nothing for a write operation
  - It injects all it's dependancies. I like to use factory methods for these, so I'll do that in my examples later on

## Database Access

So where does your database access come into play in all this? A lot of frameworks make you think about the database entirely too early. The database is just a detail about where/how your data gets persisted. Don't worry about it too much yet. Just worry about defining your Data Objects first. We'll use a *repository pattern* to send those data objects to be persisted, and it doesn't really matter to anyone but the repository class on how that's done.

## Data Objects

Your data objects classes that model your business objects. They don't extend anything, they don't connect to a database. In php, they are just POPOs (Plain old php objects). Here's a good example:

{% codeblock %}
<?php

class User
{
  public $id;
  public $email;
  public $password;
  public $first_name;
  public $last_name;

  public function __construct($data)
  {
    $this->id = $data['id'];
    $this->email = $data['email'];
    $this->password = $data['password'];
    $this->first_name = $data['first_name'];
    $this->last_name = $data['last_name'];
  }
}
{% endcodeblock %}

Easy.

## Repository Classes

A repository class is an object that persists and retrieves data objects. That's all it does. Here's an example:

{% codeblock %}
<?php

class User_Repository_Database implements User_Repository
{
  protected $_db;

  public function __construct(Database $db)
  {
    $this->_db = $db;
  }

  public function create(User $user)
  {
    $this->_db->insert('users')->values($user->as_array());
  }

  public function find_by_id($id)
  {
    return $this->_db->from('users)->where('id', '=', $id)->as_object('User');
  }
}
{% endcodeblock %}

The repository class should take data objects in, and spit data objects back out. That's it.

Next time we'll start building an actual interactor.
