## Setup and Installing Hobo

### Pre-requisites

The main dependency for Hobo is, of course, Ruby on Rails. Hobo needs at least Rails 2.0, but it's best to get the latest version (2.0.2 at the time of writing). Rails has a few dependencies itself of course, but rather than duplicating the instructions here, just follow the instructions on the main Rails site:

 * [rubyonrails.org: installing Rails](http://www.rubyonrails.org/down)
 
You will also need subversion in order to obtain the needed `will_paginate` plugin:

 * [tigris.org: installing subversion](http://subversion.tigris.org/project_packages.html)
 
(Tip: Try a web search for a better guide to installing subversion on your particular platform)

Hobo also requires a database. Before Rails 2.0.2 the default was MySQL. It just changed to SQLite3. Either of those are a fine for working through this tutorial. If you're on a Mac, SQLite3 is pre-installed since 10.4

 * [mysql.com: download MySQL 5](http://dev.mysql.com/downloads/mysql/5.0.html#downloads)
 * [sqlite.org: download SQLite](http://www.sqlite.org/download.html)


### Installing Hobo

In the process of getting Rails up and running you will install RubyGems, the package-manager for Ruby. That means installing Hobo is as easy as:

    $ gem install hobo
    
That will give you the `hobo` command, which can be used to create a new Hobo application.

Next: [Creating A Blank Application](/pod-tutorial/1-2-blank-app)