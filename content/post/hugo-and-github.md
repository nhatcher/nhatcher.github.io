---
title: "How to create a blog with Hugo and Github"
date: 2021-02-14T12:28:38+01:00
Description: "Let's create a blog!"
Tags: []
Categories: []
DisableComments: false
Draft: true
---

Do you want to create a webpage with some static content like a blog or some info page?
Are you familiar with git, github, markdown and a code editor?
You ok with paying 10 bucks a year for a domain name?

If you answered yes to those questions then you are in the right place.

Some caveats
------------

There are a million ways to create content for the Internet these days. Many of of them far easier than the one I am about to describe.
But they all come with different trade offs.

For instance you could create an account with [blogger](https://www.blogger.com/), [wordpress](https://wordpress.com/), [wix](https://www.wix.com/), [squarespace](https://www.squarespace.com/) and a long list.
Those tools are fine and they give you lot's of stuff for free. First of all you just need to create an account and start writing content.
You will have a full platform to help you create content in a way very similar to how you would create a power point or a word document.
They will help you right out of the bat with things like Google analytics, comments, adwords and much more that you can get just by mindlessly clicking away.

So, what's the catch? Well, remember the old adagio, _"If you are not paying for it, you are the product"_.
Those places might put ads, trackers and cookies. I'm not sure you really own the contents you put it.
Even though they might have a free tier, you will probably end up having to pay monthly if you need some extra feature.
They are as flexible as the tailored tools allow. They are bloated software full of unused features.
And they do have a learning curve. You will have to learn the nuts bolts of that platform of an un-transferable knowledge.
Content in any of those platforms will not be easy to move to another if one day you decide to change.
You might not be able to get your own domain name without paying or advertisement.
But if you can live with some of those gotchas go for it, they are excellent.
I have used them all! My favorite living mathematician, Terrence Tao, has a blog in [wordpress](https://terrytao.wordpress.com/). If you just want a blog, and don want to pay, blogger is fantastic. If you need a landing page for a business use wix or squarespace. and pay for it.

So, if you want to do it the free way, with not bloated software, owning your content, learning tools with transferable knowledge and are comfortable in the command line continue reading!

NOTE: even if you answer yes to all those questions, Hugo is not the only player in town. There are lots of others. Maybe the most famous one is [Jekyll](https://jekyllrb.com/).
It is not worse or better than Hugo, it's just another one.

Requirements
------------

You will need a couple of things, but with a bit of luck you already have all of those iun hand

* You will need to own a domain name. Like `https://example.com`. If you don't have one, you can get one for 10€ a year at [namecheap](https://www.namecheap.com).
* Aside from that you will need a [Github](https://www.github.com) account. Go ahead, get one, it's free!
* Make sure you have `git` installed and that you can push and pull from Github.
* Next thing you need to do is install [Hugo](https://gohugo.io/).
* Also, please make sure you have a code editor. If you don't have a preference install [Visual Studio Code](https://code.visualstudio.com/) from Microsoft, but any other will do.
* Finally, ans this is the most important part, think about some content. Something to get you started. It can be just a description about yourself and a brief post you want to make. You can just email you that content or write it in a word document, it doesn't matter.

An introduction to Hugo, the static site generator
--------------------------------------------------

So what is Hugo?. It's _a static site generator_. You give it some content in the form of markdown (mostly, but you will be able to upload images, JavaScript, CSS,...), make some choices and it produces a bunch of HTML, CSS and JavaScript. If you are into programing, this is the _compilation stage_.

Hugo, follwoing the go programming language spirit, is a single binary. In less than 50Mb of a single file we have all the power we need. You can't double click on it. It's a CLI (command line interface) tool. But it has hundreds of options.

Let me walk you through it.

Open a terminal in your computer and write (you can follow [Hugo quick start guide](https://gohugo.io/getting-started/quick-start/) for comparison)

`hugo new site <my-site-name>`

Where `<my-site-name>` is anything you want, it doesn't matter. It will be the name of the folder where all your code is sitting. Say you did:

`hugo new site mysite`

Then you will have to hop into that folder:

`cd mysite`

If you do an `ls` (or `dir` if you are on Windows), you will see the whole directory structure. There are a few folders, but they are mostly empty.
Only one file in the root folder that we care about `config.tolm`



Add a theme
-----------


CNAME
-----


Favicon
-------



