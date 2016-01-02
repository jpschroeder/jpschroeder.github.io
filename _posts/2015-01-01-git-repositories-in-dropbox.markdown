---
layout: post
title:  "Git Repositories in Dropbox"
date:   2016-01-01 22:10:00 -0500
categories: git
---
I sometimes find it useful to push git repositories into a shared Dropbox folder instead of pushing them to an actual git server like Github.  Note that the instructions below are not specific to Dropbox.  I will describe the basic process for pushing to a local folder that can be shared.

First you need to initialize a bare git repository.  This is basically the contents of the .git folder from a normal repo without the actual working copy.  This will be created in your Dropbox folder.  

{% highlight sh %}
~/Dropbox/code % git init --bare myproject.git
{% endhighlight %}

Next you need a repo to sync.  This can be preexisting, or you can create a new one like below.  This will be in a folder outside of Dropbox.

{% highlight sh %}
~/code/project % git init
~/code/project % git add .
~/code/project % git commit -m "init"
{% endhighlight %}

Now you need to add a remote that points to the Dropbox directory.  You can call it whatever you like, I generally just call it dropbox.

{% highlight sh %}
~/code/project % git remote add dropbox ~/Dropbox/code/myproject.git
{% endhighlight %}

Now you can push to the remote.

{% highlight sh %}
~/code/project % git push -u dropbox master
{% endhighlight %}

After the files have been synced to another machine you can clone the repo.

{% highlight sh %}
~/anotherservercode % git clone ~/Dropbox/code/myproject.git ./myproject
{% endhighlight %}

I'm sure there are many other explanations like this across the internet, but I hope you find it useful anyway.  I know I do.  
