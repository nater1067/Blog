---
layout: post
title: "Sexy URLS For Vagrant With FoxyProxy"
date: 2014-09-08 13:09:54 -0700
comments: true
categories: 
---
<h2>Use Foxy Proxy to make your life easier (Mac OSX/ Linux)</h2>
If you’re like me and you don’t like seeing an ugly IP address during development, you can use FoxyProxy to easily change the http://127.0.0.1:8080 to vagrant.dev or any url of your choice.<br />
<br/>
Start off with either a Vagrant project of your own, or this basic Symfony 2 Vagrant project here: https://github.com/nateVegas/Vagrant-Symfony-2 . Clone the project and follow the installation instructions. Run "vagrant up".<br/>
<br />
Now install FoxyProxy.
Firefox Users: https://addons.mozilla.org/en-US/firefox/addon/foxyproxy-standard/
Chrome Users: https://chrome.google.com/webstore/detail/foxyproxy-standard/gcknhkkoolaabfmlnjonogaaifnjlfnp?hl=en
<br />

After installing you should see a little fox somewhere on your browser. Click that fox.
Make sure “Use proxies based on their pre-defined patterns and priorities” is selected.

{% img /images/spinning-up-symfony-2-dev/enable-foxy.png 700 700 'image' 'images' %}

Click on the new fox and select "Options". Add a new proxy for 127.0.0.1 port 8080.

{% img /images/spinning-up-symfony-2-dev/create-proxy.png 700 700 'image' 'images' %}

Then add a new url pattern. Set “*symfony.dev*” as the url pattern, optionally set a pattern name, make sure whitelist urls and wildcards are selected. Press save, exit foxy proxy and navigate to http://symfony.dev.

{% img /images/spinning-up-symfony-2-dev/whitelist-url.png 700 700 'image' 'images' %}