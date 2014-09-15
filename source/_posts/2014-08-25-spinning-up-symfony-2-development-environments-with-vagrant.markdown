---
layout: post
title: "Spinning up Symfony 2 Development Environments with Vagrant"
date: 2014-08-25 14:59:49 -0700
comments: true
categories: 
---

<h2>How it’s gonna go</h2>
<ol>
    <li>Go over basics of Puppet and Vagrant</li>
    <li>Use Composer to grab symfony-standard-edition</li>
    <li>Create the VagrantFile</li>
    <li>Provisioning a Symfony 2 environment using Puppet</li>
    <li>Vagrant Up</li>
</ol>

<h2>Vagrant + Puppet Basics</h2>
<p>
For the purpose of this tutorial there are only a few things you need to know about Vagrant and Puppet. Vagrant creates virtual machines using a number of different providers. We’ll be using Virtualbox. Vagrant runs Puppet for you on the newly created virtual machine. Puppet handles the installation of various dependencies. (Think Apache, mysql, curl, etc.) This process is known as provisioning.
</p>

<h2>Why encapsulate development environments with Vagrant?</h2>
<p>
When we use Vagrant to create new virtual development environments we avoid the very real possibility that we could mess up our personal development machines. People have used virtual machines for development for years. The drawbacks of passing around USB sticks with Virtualbox images are easy to spot. For one, without a complicated and slow shared folder setup, you need to use and IDE or text editor inside the VM to modify code. You won’t be able to run this environment headless to free up clock cycles on your host machine. Managing installed applications across a teams VMs is a pain. Why not just include a Vagrantfile and a few Puppet manifests instead? Instead of passing around a virtual machine a few gigabytes in size, just include your Vagrant and Puppet in a project's source control. That’s it. In future tutorials we will be using the environment we create here to start a new virtual machine running Symfony 2 with the above command. See vagrantup.com for more reasons why Vagrant is awesome.
<p>

<h2>Create a Symfony Project with Composer</h2>
<p>
Let’s have Composer install a Symfony 2 project in a directory, let’s call the directory symfony2. Open up a terminal window and enter the following commands:
</p>

``` bash
curl -s https://getcomposer.org/installer | php
php composer.phar create-project symfony/framework-standard-edition symfony2 'dev-master'
cd symfony2
```
<p>
At the end of the composer create-project command, you should be prompted with “Some parameters are missing. Please provide them.”<br />

Let’s install the AcmeDemoBundle so we can see the welcome page when we're finished. When prompted for “Would you like to install Acme demo bundle? [y/N]” type y and press enter.<br />

Go ahead and press enter for each of the remaining fields to leave the default values.<br />

Because we are running a development environment let’s modify two lines in the web/.htaccess file. Replace occurrences of app with app_dev in these two location:<br />
</p>
``` apacheconf
# /web/.htaccess
RewriteRule ^app_dev\.php(/(.*)|$) %{ENV:BASE}/$2 [R=301,L]
RewriteRule .? %{ENV:BASE}/app_dev.php [L]
```

While we are in the new Symfony project, let's disable some security checks in the app_dev.php file. Comment out the following lines:
``` php
# /web/app_dev.php
// header('HTTP/1.0 403 Forbidden');
// exit('You are not allowed to access this file. Check '.basename(__FILE__).' for more information.');
```

We have Symfony 2, but we’re not going to be able to develop in this awesome framework until it’s running on Apache. Let’s have Vagrant do the heavy-lifting for this.

<h2>Setting up Vagrant</h2>
<p>
Vagrant is powerful out of the box, and there’s not a whole lot of configuration we have to do ourselves. Make sure you have the latest version of Vagrant installed. See http://www.vagrantup.com/downloads.html for installation packages. Open the project directory in another terminal.
</p>
``` bash
vagrant init hashicorp/precise32
```

<p>
Then open up the newly created Vagrantfile in your favorite text editor (use PHPStorm!) and enable Puppet provisioning. Also, ensure port 8080 is forwarded to 80 on the VM. Vagrant is now ready to go.
</p>

``` ruby
    # /Vagrantfile
    VAGRANTFILE_API_VERSION = "2"

    Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
      config.vm.box = "hashicorp/precise32"
      config.vm.network "forwarded_port", guest: 80, host: 8080

      # Enable provisioning with Puppet stand alone.  Puppet manifests
      # are contained in a directory path relative to this Vagrantfile.
      # You will need to create the manifests directory and a manifest in
      # the file default.pp in the manifests_path directory.
      #
      config.vm.provision "puppet" do |puppet|
        puppet.manifests_path = "manifests"
        puppet.manifest_file  = "site.pp"
      end

    end
```

<h2>Provisioning with Puppet / Symfony’s Requirements</h2>
<p>
Let’s face it.. Symfony 2 is amazing and all powerful, but not the fastest framework to get started with. Let’s eliminate the need for manual intervention when setting up a new symfony application. In order to get our symfony application running we need to ensure 3 things happen:
</p>

<h3>Install our dependencies</h3>
<p>
Since we are keeping this Puppet script very basic for now, all we need to to install Apache, php, modrewrite and mysql. Create a folder called manifests in our project and a file named sites.pp in the new folder.
</p>

``` puppet
# /manifests/site.pp
exec {"apt-get update":
  path => "/usr/bin",
}

package {"apache2":
  ensure => present,
  require => Exec["apt-get update"],
}

service { "apache2":
  ensure => "running",
  require => Package["apache2"]
}

package {['mysql-server', 'mysql-client']:
  ensure => installed,
  require => Exec["apt-get update"]
}

service { 'mysql':
  ensure  => running,
  require => Package['mysql-server'],
}

package { ["php5-common", "libapache2-mod-php5", "php5-cli", "php-apc", "php5-mysql"]:
  ensure => installed,
  notify => Service["apache2"],
  require => [Exec["apt-get update"], Package['mysql-client'], Package['apache2']],
}

exec { "/usr/sbin/a2enmod rewrite" :
  unless => "/bin/readlink -e /etc/apache2/mods-enabled/rewrite.load",
  notify => Service[apache2],
  require => Package['apache2']
}
```

<h3>Set up a new VirtualHost</h3>
First we need to point /var/www to our /vagrant directory.
``` puppet
# /manifests/site.pp
file {"/var/www":
  ensure => "link",
  target => "/vagrant",
  require => Package["apache2"],
  notify => Service["apache2"],
  replace => yes,
  force => true,
}
```

Let’s set up a simple vhost pointing to our symfony project’s web directory. We should include the vhost in our git repository. Let’s place a file named vhost.conf in the manifests/assets folder.
``` apacheconf
# /manifests/assets/vhost.conf
<VirtualHost *:80>
    ServerName symfony.dev
    DocumentRoot /vagrant/web
    <Directory /vagrant/web>
        # enable the .htaccess rewrites
        AllowOverride All
        Order allow,deny
        Allow from All
    </Directory>

    ErrorLog /var/log/apache2/error.log
    CustomLog /var/log/apache2/access.log combined
</VirtualHost>
```

Now we just have to tell Puppet to create a symlink to that file in the Apache sites-available directory.

``` puppet
# /manifests/site.pp
file { "/etc/apache2/sites-available/default":
  ensure => "link",
  target => "/vagrant/manifests/assets/vhost.conf",
  require => Package["apache2"],
  notify => Service["apache2"],
  replace => yes,
  force => true,
}
```

<h3>Set Apache to run as the Vagrant user</h3>
Because this is a development-only environment we can use a little hack I stumbled upon on Jeremy Kendall’s blog (http://jeremykendall.net/2013/08/09/vagrant-synced-folders-permissions/).
All we need to do is change the Apache user and group to use Vagrant’s user and group. This way Apache can write to cache and log directories with no problem!
``` puppet
# /manifests/site.pp
exec { "ApacheUserChange" :
  command => "/bin/sed -i 's/APACHE_RUN_USER=www-data/APACHE_RUN_USER=vagrant/' /etc/apache2/envvars",
  onlyif  => "/bin/grep -c 'APACHE_RUN_USER=www-data' /etc/apache2/envvars",
  require => Package["apache2"],
  notify  => Service["apache2"],
}

exec { "ApacheGroupChange" :
  command => "/bin/sed -i 's/APACHE_RUN_GROUP=www-data/APACHE_RUN_GROUP=vagrant/' /etc/apache2/envvars",
  onlyif  => "/bin/grep -c 'APACHE_RUN_GROUP=www-data' /etc/apache2/envvars",
  require => Package["apache2"],
  notify  => Service["apache2"],
}

exec { "apache_lockfile_permissions" :
  command => "/bin/chown -R vagrant:www-data /var/lock/apache2",
  require => Package["apache2"],
  notify  => Service["apache2"],
}
```

Now run “vagrant up” from your project directory and watch as your Symfony 2 application comes to life! Navigate to http://127.0.0.1:8080 to see a Symfony 2 welcome page.

<h2>Conclusion</h2>
Now we have a working Symfony 2 project running on a virtual machine we can develop on immediately. To see the real beauty of having Vagrant and Puppet do the work for us, copy this project to another device, run "composer install" and then "vagrant up." You should be able to develop on the project from any host running OSX, Windows or Linux and see your changes at http://127.0.0.1:8080 .

The source can be found on my Github account here: https://github.com/nateVegas/Vagrant-Symfony-2 .

Do you know of any ways to improve the project we just created? If so please leave a comment so I can update this post based off your feedback.