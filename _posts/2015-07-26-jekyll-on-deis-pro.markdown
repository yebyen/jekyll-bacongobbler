---
layout: post
title:  "Set Up a Jekyll Blog on Deis PRO"
date:   2015-07-26 17:00:10
categories: deis pro tutorial
---

Deis PRO is a new product from EngineYard, which makes it easy to spin up a new Deis cluster on Amazon AWS and allows you to leverage the excellent support provided by the Deis team at EngineYard.  Jekyll is a separate project for building websites, perhaps not coincidentally, a tool that is used by the Deis team to provide supporting architecture and scaffolding for the blog and documentation of Deis itself.  As far as hosting your blog goes, Deis PRO is quite possibly the sledgehammer version of the basic flyswatter design that most people would use for a problem as minor as static content hosting.

If you are running a business, and you need your company blog (perhaps among some other discrete web services that you consume) to exhibit some demonstrable guarantees of scalability and fault-tolerance; then this post is for you.  To build a hosting system for your blog that will not crumble under stress, one that could potentially survive a Reddit mob or a "Slashdotting" from the unforgiving hordes of tech-savvy readers, one that could be scaled horizontally in order to better handle an increased volume of load when it is needed most... for these, Deis PRO is an excellent option which one can undertake to deploy with just a credit card, an Amazon account, and little to no prior experience.

It is assumed of the reader that you have already determined your own need for something like Deis.  By following this guide you will gain a fully functional Deis cluster, with plenty of room to grow.  Hosting your blog on Deis is incidental.  You should not need additional resources, separate from your Deis PRO cluster, just to host your blog!  It is a perfect example of a most trivial application that you might want to have in production.

This tutorial will take you through the motions of easily bringing up a cluster on AWS using Deis PRO, then installing and performing some basic configuration of a Jekyll blog on it.  Prepare for a crash-course in Deis PRO and Jekyll.  In case you are interested in only one of either setting up Deis PRO or deploying Jekyll (to Deis or Heroku, or any other Heroku-like PaaS that uses Heroku-style buildpacks if you do know of some other), the guide is separated into two sections, [Getting Started with Deis PRO](#getting-started) and [Setting Up Jekyll](#setting-up-jekyll).

### Getting Started

* You will need: an Amazon AWS account
* Log into the AWS management console
* Locate the Identity and Access Management service under Administration & Security
* You will need to create an IAM user during the guided setup instructions of Deis PRO.
* Visit [deis.com](http://deis.com) and follow the Try It Now guide points until you have a Deis cluster on AWS.  (Thanks, EngineYard!)
* Create a user for your Deis cluster and note the credentials for the next section.

### Setting Up Jekyll

There is a repository for [heroku-buildpack-jekyll.git](https://github.com/bacongobbler/heroku-buildpack-jekyll) that is kept by one of the core developers of Deis, `@bacongobbler`; but when I first went to try and use it myself I found that too much had changed since it was last updated in 2014, and the steps in the README.md no longer worked on either Heroku or Deis.

The word from the man `bacongobbler` himself is that NodeJS, at least, was removed from the cedar stack in cedar-14, and this is probably what caused the buildpack to stop working.  I liked the idea of using a buildpack made just for jekyll, and though I am far from expert, I was able to update it with about a day of poking and prodding to be a (hopefully) decent, serviceable imitation of the original.

I put my own forked version of the buildpack up [on github](https://github.com/yebyen/heroku-buildpack-jekyll) which is in need of further documentation.

As of today, between this article and [my jekyll-bacongobbler demo deployment](https://github.com/yebyen/jekyll-bacongobbler) (which you could clone, and push to Deis in lieu of following the rest of this guide), this what you have here is the best documentation about how to use the updated buildpack.

Start on your local machine by [installing ruby](https://rvm.io/rvm/install) if necessary, and also `bundler`, then `jekyll`:

    $ gem install bundler
    $ gem install jekyll

This part may not work on CoreOS, if the newly provisioned nodes of your cluster are the only computers that you have available: ...

I haven't tested if you can just use CoreOS, but `rvm` supports most common desktop Linux distributions including RedHat, Debian, Ubuntu, and many others.  I am almost sure it won't work on a CoreOS node, because you don't have compilers.  So, even if a binary distribution of Ruby is available, you won't be able to compile native extensions when you install gems (and you need that.)

I have been pleasantly surprised before, but I don't think this is one of those times.  If that's your situation, I suggest that you should create an Ubuntu container to try the rest of the work on.  If you are using a Windows or MacOS machine, well... I don't, so you will be more qualified than I am to say how Ruby is meant to be installed there.  Jekyll is built with ruby.  You may or may not be able to use these instructions without actually installing jekyll.  You will need the `deis` cli.

You should now be able to use `jekyll` to create a blog into an empty directory from the default template.  I called mine `blogosite` here:

    $ jekyll new blogosite
    $ cd blogosite/
    $ jekyll serve
    Configuration file: /home/yebyen/Downloads/jekyll-bacongobbler/_config.yml
                Source: /home/yebyen/Downloads/jekyll-bacongobbler
           Destination: /home/yebyen/Downloads/jekyll-bacongobbler/_site
          Generating... 
                        done.
     Auto-regeneration: enabled for '/home/yebyen/Downloads/jekyll-bacongobbler'
    Configuration file: /home/yebyen/Downloads/jekyll-bacongobbler/_config.yml
        Server address: http://0.0.0.0:4000/
      Server running... press ctrl-c to stop.

If that worked, visiting [127.0.0.1:4000](http://127.0.0.1:4000) will let you preview the blogosite, hosted on your local machine.

My buildpack update has some outside dependencies that [the multi-buildpack](https://github.com/ddollar/heroku-buildpack-multi) will help you to pull down.  Invoke it by creating a file `.buildpacks`, with these contents:

    git://github.com/heroku/heroku-buildpack-nodejs.git
    git://github.com/heroku/heroku-buildpack-ruby.git
    git://github.com/yebyen/heroku-buildpack-jekyll.git

Make a `git` repository from your jekyll default template:

    $ git init
    $ git add -A
    $ git commit -m"initial blogosite commit"

Be logged in as a user on your Deis cluster, and push your `blogosite` up to the cluster:

    deis login deis.cluster.your.domain
    deis keys:add
    deis create
    git push deis master

Now you wait, and after a brief moment (unless fate smiles, and this article is older than the buildpack) you find that the push has resulted in a state of failure.  See something like this:

    $ git push deis master
    Warning: Permanently added the ECDSA host key for IP address '[45.55.208.23]:2222' to the list of known hosts.
    Counting objects: 24, done.
    Delta compression using up to 2 threads.
    Compressing objects: 100% (22/22), done.
    Writing objects: 100% (24/24), 8.64 KiB | 0 bytes/s, done.
    Total 24 (delta 1), reused 0 (delta 0)
    -----> Multipack app detected
           =====> Downloading Buildpack: git://github.com/heroku/heroku-buildpack-nodejs.git
    To ssh://git@deis.deis.moomboo.space:2222/yeoman-odometer.git
     ! [remote rejected] master -> master (pre-receive hook declined)
    error: failed to push some refs to 'ssh://git@deis.deis.moomboo.space:2222/yeoman-odometer.git'

There are some [necessary adjustments](https://github.com/yebyen/jekyll-bacongobbler/commit/a97a48774627cbf8c06c38bc527445aeec184a15) that you must make to your `blogosite` for the new buildpack in order to make it deployable.  It needs:

a `Gemfile`:

    ruby '2.2.2'
    source 'https://rubygems.org'
    gem 'jekyll'
    gem 'pygments.rb'
    gem 'kramdown'

a `package.json`:

    {
      "engines": {
        "node": "0.12.x"
      }
    }

and a one-liner added to end of the `_config.yml` already generated by `jekyll new`:

    exclude: [vendor]

Add a domain name to your new app with the deis cli, point the DNS at your load balancer, and reap the rewards:

    deis domains:add blogosite.io

The author of this tutorial notes as an aside that, ...

If it was not made perfectly clear at the outset already, a Blog-only Deis cluster deployment is almost definitely a tremendous waste of resources

Sure, your blog will probably never go down, but if you want to do it cheaper, there is almost surely a less expensive way, and if you followed this guide from start to finish, you may have just started the clock ticking on a new recurring expense.  I hope you need more from Deis than a blog, but even if that is all you need, rest assured that the EngineYard and Deis team will be there to support you with Deis PRO.

If you read this far and you are not sure why you need Deis, there is a cheaper way to host your blog.  In fact, there is a cheaper way I know, and will share, but it involves doing un-supported things.  A Deis cluster, like almost any after-market consumable product, has [the recommended system requirements](http://docs.deis.io/en/latest/installing_deis/system-requirements/), and then there are [the requirements](/cheapest-fault-tolerant-deis-cluster).  So long for now!

