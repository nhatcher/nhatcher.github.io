---
title: "How to create a blog with Hugo and Github"
date: 2021-02-14T12:28:38+01:00
Description: "Let's create a blog!"
Tags: []
Categories: []
DisableComments: false
Draft: true
---

        Forgive me pretty baby but I always take the long way home.
        Tome Waits

Do you want to create a webpage with some static content like a blog or some info page?
Are you familiar with git, github, markdown and a code editor?
You ok with paying 10 bucks a year for a domain name?

If you answered yes to those questions then you are in the right place.

You can go ahead and skip the caveats section where I present you with a rainbow of other possibilities.
As Steven Weinberg usually says, "Sections marked with an asterisk are somewhat out of the book's main line of development and may be omitted in a first reading"

Some caveats (*)
----------------

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

Even if you answer yes to all those questions, Hugo is not the only player in town. There are lots of others. Maybe the most famous one is [Jekyll](https://jekyllrb.com/).
It is not worse or better than Hugo, it's just another one.

You do not need to use Github, you can as well in a very similar fashion use [Gitlab](https://about.gitlab.com/). A yet different place would be [Netlify](https://www.netlify.com/).

To finalize, we will be using _automated deployments_ that, in our case, came with the name of _GitHub actions_. This means that we will setup things in a way that you can make a change in your blog and the blog will update automatically. But you don't need to do that. You can just as well build the site on your machine and then push the built bundle.

Setting everything up
---------------------

You will need a couple of things, but with a bit of luck you already have all of those iun hand

* You will need to own a domain name. Like `https://example.com`. If you don't have one, you can get one for 10€ a year at [namecheap](https://www.namecheap.com).
* Aside from that you will need a [Github](https://www.github.com) account. Go ahead, get one, it's free!
* Make sure you have `git` installed and that you can push and pull from Github.
* Next thing you need to do is install [Hugo](https://gohugo.io/).
* Also, please make sure you have a code editor. If you don't have a preference install [Visual Studio Code](https://code.visualstudio.com/) from Microsoft, but any other will do.
* Finally, and this is the most important part, think about some content. Something to get you started. It can be just a description about yourself and a brief post you want to make. You can just email you that content or write it in a word document, it doesn't matter.

You can now jump to an introduction to Hugo or go and learn about GitHub and GitHub actions.

An introduction to Hugo, the static site generator
--------------------------------------------------

So what is Hugo?. It's _a static site generator_. You give it some content in the form of markdown (mostly, but you will be able to upload images, JavaScript, CSS,...), make some choices and it produces a bunch of HTML, CSS and JavaScript. If you are into programing, this is the _compilation stage_.

Hugo, following the go programming language spirit, is a single binary. In less than 50Mb of a single file we have all the power we need. You can't double click on it. It's a CLI (command line interface) tool. But it has hundreds of options.

Let me walk you through it.

Open a terminal in your computer and write (you can follow [Hugo quick start guide](https://gohugo.io/getting-started/quick-start/) for comparison)

`hugo new site <my-site-name>`

Where `<my-site-name>` is anything you want, it doesn't matter. It will be the name of the folder where all your code is sitting. Say you did:

`hugo new site mysite`

Then you will have to hop into that folder:

`cd mysite`

If you do an `ls` (or `dir` if you are on Windows), you will see the whole directory structure. There are a few folders, but they are mostly empty.
Only one file in the root folder that we care about `config.toml`


## Add a theme



Awesome GitHub
--------------

In this section we are going to crete a webpage served by GitHub.

I'm going to share a secret with you, GitHub is freaking awesome. And what they do for people is awesome.
In this section we are going to learn how to host a webpage in GitHub. This is totally unrelated to Hugo.

There are two slightly different ways to host a webpage in GitHub.

* You can host a page for a project of yours in `https://<username>.github.io/<project-name>`
* You can host a page for the whole thing at `https://<username>.github.io/`

The process is exactly the same, so I will just show you how to do the second way.

First thing you need to do is to create a new repository with this very special name: `<username>.github.io`. For example, here is mine:

[Nicolás 'root' repo](https://github.com/nhatcher/nhatcher.github.io)

This is your first exercise, ready? Create the repo and add an `index.html` file. If you don't know what to write just copy/paste this:

```html
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My First GitHub hosted page</title>
    <h1>Brilliant!</h1>
    <div>I feel this blog is going <span style="color: red">places</span>!</div>
</head>
<body>
    
</body>
</html>
```

This is just code

You can use GitHub to host the source code of a project of yours. For instance I have a project [here](https://github.com/nhatcher/ariana-lua) that allows you to plot functions using the [Lua](http://www.lua.org/) programming language!

GitHub Actions (*)
------------------

In this sections we will create the content in Markdown and use GitHub actions to generate the HTML for us.

Custom domain
-------------

In this section we will lear how to use our custom domain `https://www.example.com` instead of the GitHub URL.
Visitors of your blog will not be aware that all the content is in GitHub!


Favicon (*)
-----------

Your website probably needs a [favicon](https://en.wikipedia.org/wiki/Favicon). The easiest way is to go to one of the many [favicon generators](https://favicon.io/favicon-generator/).
If you don't know what to do just get your initials, some colors and good to go! If it works for [Bill Gates](https://www.gatesnotes.com/) works for you.





