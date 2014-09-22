---
layout: post
title: "Behavior-Driven Development With Behat"
date: 2014-09-22 13:38:03 -0700
comments: true
categories: 
---
<h2>Behavior-Driven Development With Behat and Mink</h2>

Unit testing is not the only practice software engineers should use to ensure a project is operating correctly. Behavior-driven development has been growing in popularity since 2008. In short, BDD starts with a “story.” Whoever is in charge of designing the software’s flow writes use-cases for features in human-readable language. Tests are then run using headless browsers and other automation tools to ensure the story can be execute properly.<br />

A short story in Gherkin, Behat's markup language, might look like this:

{% codeblock lang:Cucumber %}
Scenario: Search for red hat
    Given I am on "/search"
    And I fill in "search_input" with "red hat"
    And press "submit"
    Then I should see "red hat - The best clothing item you will ever buy."
{% endcodeblock %}

This story looks eerily similar to a functional requirements document because, well, that's what it is. Typically someone doing quality assurance would have to manually follow this scenario every time a new feature or bug fix has been implemented in order to properly regression test. But Behat can automatically test these scenario in a variety of browsers, headless or not. This is great for testing complex features like those involving AJAX requests. Mink defines how Behat should translate the Gherkin steps into browser commands in order to test the site. We can also define our own custom steps here.<br />

Let's take a look at how we can implement Behat to test an e-commerce site.

<h2>Set Up</h2>

Clone the simple e-commerce site and checkout the basic-shop tag. This post assumes you are working in either OSX or Linux.
{% codeblock lang:bash %}
git clone https://github.com/nater1067/Shop-Top.git
cd Shop-Top
git checkout -b Shop-Top
curl -sS https://getcomposer.org/installer | php
php composer.phar install
vagrant up
{% endcodeblock %}

Press enter after the composer install for each field to keep default values. Now you should be able to navigate to http://127.0.0.1:8080/search to see a working ecommerce site. Try buying a red hat to see how the site works. (Note: You will not be charged for the any red hats on this site.)

<h2>Including Behat and Mink Libraries</h2>

We need to add a few dependencies to our project using composer. This took a little time to figure out, and at the time of writing this post works. However be warned - you may need to adjust the version numbers in order to get this working in the future.
{% codeblock lang:json %}
# composer.json
{
    # When you see "..." it means there may be code here.
    # Don't delete anything unless it's mentioned in a post.
    # ...
    "require": {
        # ...
        "behat/symfony2-extension": "~1.1",
        "behat/mink-extension": "~1.3",
        "behat/mink-selenium2-driver": "~1.1",
        "behat/mink-goutte-driver": "~1.0"
    }
}
{% endcodeblock %}

Run composer update to download the the Behat and Mink libraries. Then create the necessary behat files.

{% codeblock lang:bash %}
php composer.phar update
bin/behat --init
{% endcodeblock %}

This creates a new directory "features" which holds our Gherkin and Behat FeatureContext. Modify the FeatureContext to extend MinkContext.

{% codeblock lang:php %}

use Behat\MinkExtension\Context\MinkContext;

class FeatureContext extends MinkContext
{
    /* ... */
}
{% endcodeblock %}

We should also create a new file in the project root directory - behat.yml.

{% codeblock lang:yaml %}
# behat.yml
default:
    extensions:
        Behat\MinkExtension\Extension:
            base_url: http://127.0.0.1:8080
            goutte: ~
            selenium2: ~
{% endcodeblock %}

<h2>Creating the Gherkin Story</h2>

Time to write our features in human terms. Create two new files: features/search.feature and features/buy.feature.

{% codeblock lang:Cucumber %}
# features/search.yml
Feature: Clothes Search
  In order to search for clothes
  As a shopper
  I need to enter a search query and press submit

  Scenario: Search for red hat
    Given I am on "/search"
    And I fill in "search_input" with "red hat"
    And press "submit"
    Then I should see "red hat - The best clothing item you will ever buy."

  Scenario: Search for empty string
    Given I am on "/search"
    And press "submit"
    Then I should not see "Results"
{% endcodeblock %}

{% codeblock lang:Cucumber %}
# features/buy.yml
Feature: Buy Clothing Item
  In order to buy a clothing item
  As a shopper
  I need to press buy next to an item

  Scenario: Buy red hat
    Given I am on "/search"
    And I fill in "search_input" with "red hat"
    And I press "submit"
    Then I press "buy_redhat"
    Then I should see "You purchased 1 red hat!"
{% endcodeblock %}

These files describe, in fairly simple terms, what the application should do. We should be able to search for a red hat, and see the description in the results.

<h2>Running Behat</h2>

Now that we have created a couple of Gherkin stories, we can run Behat to test our site's behavior in a headless browser.

{% codeblock lang:bash %}
bin/behat
{% endcodeblock %}

We should see our stories outputted to the terminal. If everything works we should see a lot of green, and a summary of successful scenarios and steps.