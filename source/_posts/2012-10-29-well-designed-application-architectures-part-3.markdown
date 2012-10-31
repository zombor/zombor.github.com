---
layout: post
title: "well-designed-application-architectures-part-3"
date: 2012-10-29 07:37
comments: true
categories: 
---

[Last time](/blog/2012/10/02/well-designed-application-architectures-part-2/) we talked about how your business logic should look. It was a high level overview of the architecture. This time we'll actually put that into practice and write some code. We'll be using BDD (Behavior-Driven-Development) to design the application.

We're going to be using [Behat](http://behat.org/) to run these features. So let's start a new project!

    mkdir sample-app; cd sample-app
    git init

We'll use composer to install Behat:

{% codeblock composer.json %}
{
  "require": {
    "behat/behat": "2.4.*@stable"
  },
  "config": {
    "bin-dir": "bin/"
  }
}
{% endcodeblock %}

And let's install it:

{% codeblock %}
curl http://getcomposer.org/installer | php
php composer.phar install --prefer-source

bin/behat --init
{% endcodeblock %}

So where do we start? BDD says we should start from the outside and move our way in. So, we'll start with some Gherkin. I'll make up some requirements that you might receive from a product manager:

{% codeblock features/register_account\.feature %}
Feature: User Registration

  Scenario: User Registers With Valid Data
    Given I am an unregistered user
    When I register with the following information:
      | first_name | foo         |
      | last_name  | bar         |
      | email      | foo@bar.com |
      | password   | foobar      |
    Then I should be registered
    And I should receive a welcome registration email

  Scenario: User Registers With Invalid Data
    Given I am an unregistered user
    When I register with the following information:
      | first_name | foo         |
      | last_name  | bar         |
      | email      | foo@bar.com |
      | password   |             |
    Then I should not be registered
    And I should see the following errors:
    """
    password is required
    """
{% endcodeblock %}

Now let's run it:

{% codeblock %}
bin/behat
{% endcodeblock %}

You'll see a bunch of undefined steps with some snippet code. Let's paste these into our context file (features/bootstrap/FeatureContext.php) and rerun behat. Now we'll see some TODOs! Time to actually write code! We'll start from the top:

{% codeblock %}
Given I am an unregistered user
{% endcodeblock %}

This is going to be a no-op. We don't need to do anything here, so let's just remove the PendingException in the step defintion:

{% codeblock %}
/**
 *  @Given /^I am an unregistered user$/
 */
public function iAmAnUnregisteredUser()
{
}
{% endcodeblock %}

Now that step should be green when we re-run the behat steps. We'll move on to the next step. Here's where the real meat will start. Since we are writing our business logic, we'll start with defining the interface we want to work with, and write the code we *wish* we had.

{% codeblock %}
/**
 * @When /^I register with the following information:$/
 */
public function iRegisterWithTheFollowingInformation(TableNode $table)
{
  $parsed_data = [];
  foreach ($table->getNumeratedRows() as $row)
  {
    $parsed_data[$row[0]] = $row[1];
  }

  $register = new Account_Registration;
  $result = $register->execute($parsed_data);
}
{% endcodeblock %}

This might be something we would want. Instantiate a new use case class (Account_Registration), and call it's execute() method with our registration data.

Let's run this in behat:

{% codeblock %}
PHP Fatal error:  Class 'Account_Registration' not found
{% endcodeblock %}

Now we are blocked, we have an error in our tests. So let's make this thing pass:

{% codeblock classes/Account/Registration\.php %}
<?php

class Account_Registration
{
  public function execute($params = [])
  {

  }
}
{% endcodeblock %}

Also make sure you require this file in your FeatureContext class.

{% codeblock %}
require 'classes/Account/Registration.php';
{% endcodeblock %}

Now if you run behat, we'll have the next step green in both scenarios.

Let's move on to the next step. What do we do here? How do we assert the account was created? Simple. We want to use Dependancy Injection, and inject a *repository object* into the use case class. The use case will then use that repository to persist the data. We will use a mock object to keep track of this. We don't want to use the real thing because we don't care about the database right now \(we will later, and we'll replace this code later\). We'll need to modify our last step definition:

{% codeblock %}
$this->account_repo = new Mock_Account_Repository;
$register = new Account_Registration($this->account_repo);
{% endcodeblock %}

Now when we run behat, we'll get a fatal error about this Mock class not existing. So let's just create it at the end of the file.

{% codeblock %}
class Mock_Account_Repository implements Account_Repository
{
  public $create_called = FALSE;
  public $create_data = NULL;

  public function create($data)
  {
    $this->create_called = TRUE;
    $this->create_data = $data;
  }
}
{% endcodeblock %}

And we'll need to define that interface. Let's do that in our classes folder:

{% codeblock classes/Account/Repository\.php %}
<?php
interface Account_Repository {
  public function create(array $data);
}
{% endcodeblock %}

Make sure you require that file right above the Mock_Account_Repository class defined above. Now we need to assert that our class does what we want. So in our next step definition, we'll inspect that mock object:

{% codeblock %}
/**
 *  @Then /^I should be registered$/
 */
public function iShouldBeRegistered()
{
  Assertion::true($this->account_repo->create_called);
  Assertion::same($this->data, $this->account_repo->create_data);
}
{% endcodeblock %}

Now when you run this, it should fail. So let's make it pass.

{% codeblock classes/Account/Registration\.php %}
<?php

class Account_Registration
{
  protected $_account_repository;

  public function __construct(Account_Repository $account_repository)
  {
    $this->_account_repository = $account_repository;
  }

  public function execute($params = [])
  {
    $this->_account_repository->create($params);
  }
}
{% endcodeblock %}

On to the next step! We want to send an email out to the user after they register. We need to abstract out the email handling, so let's make a mock class for that too.

{% codeblock %}
class Mock_Email_Sender implements Email_Sender
{
  protected $sent = array();

  public function send_mail($to, $from, $subject, $body)
  {
    $this->sent[] = array('to' => $to, 'from' => $from, 'subject' => $subject, 'body' => $body);
  }
}
{% endcodeblock %}

And the interface for this class:

{% codeblock classes/Email/Sender\.php %}
<?php

interface Email_Sender
{
  public function send_mail($to, $from, $subject, $body);
}
{% endcodeblock %}

And let's add this to the step definition:

{% codeblock %}
/**
 *  @Given /^I should receive a welcome registration email$/
 */
public function iShouldReceiveAWelcomeRegistrationEmail()
{
  Assertion::same($this->email, $this->email_sender->sent[0]['to']);
}
{% endcodeblock %}

You'll need to modify some previous steps for this. When you are done, you should get an error like:

{% codeblock %}
Notice: Undefined offset: 0 in features/bootstrap/FeatureContext.php line 81
{% endcodeblock %}

So we just need to make this pass. Easy:

{% codeblock classes/Account/Registration\.php %}
  public function execute($params = [])
  {
    $this->_account_repository->create($params);
    $this->_email_sender->send_mail($params['email'], 'donotreply@foo.com', 'Welcome!', 'The body');
  }
{% endcodeblock %}

Sweet! Our first scenario passes! You should be able to implement the rest of the second scenario fairly easily.

This concludes our tutorial on creating a basic Interactor class. It doesn't do much right now, but next time we'll add in some validation, and do some other fun stuff with phpspec.
