---
layout: post
title:  "7 basic git commands to get you started"
date: 2019-09-18 18:00:00 +0700
categories: general
location: Huaraz, Peru
location-link: Huaraz
---

![Git logo >](https://miro.medium.com/max/910/1*-l3Qrum9oPMiiM0yW7v_Ng.png){: height="100px"}

Getting started in git is surprisingly easy and provides a quick and simple way to ensure your scripts are saved online. It's also a quick way to feel like a pro using the command line interface (CLI). In this post are the 7 commands you need to manage a simple repository.

<!--description-->

In short they are the following:

```dcl
git clone
git init [url]
git status
git add
git commit -m "[message]"
git push
git pull
```

This is it. All you'll need to get you started if you're simply trying to maintain a basic respository.

## Downloading and using git

First you'll need to **download git** to your machine to be able to use it. You can do that [here](https://git-scm.com/downloads).

**Using git** is done from your command line or CLI. Normally opened by simply searching 'cmd'. Most likely you can open a terminal from your text editor as well. Both Jupyter Labs/Notebook and Visual Studio code have a CLI.

You can check that git has installed correctly by typing in `git --version` into the command line. You should see something like `git version 2.18.0.windows.1`

## Getting started

There are two ways to get started.

1. You create a folder on Github and clone it to your computer
2. You already have a folder on your computer and want to link it to Github or will create a new one from your computer

For both of these steps, the first thing you'll need to do is navigate your folder directory via the command line. This is essentially pointing the CLI at your desired location. Learning how to do this is a key skill for any would-be programmer.

### Creating the folder/repository on Github

For those who have created a repository already on Github, you will need to navigate to the parent folder where you want your repository to appear.

For example, if you named your repository 'test_repo' on Github and you want it to appear in 'C:\Users\ryanf\Documents>' then you would make this your directory in the CLI (using `cd C:\Users\ryanf\Documents>`, cd stands for change directory) and the folder 'test_repo' would appear in here with the below commands.

To have the folder appear, you simple need to grab the clone URL from your repository on Github. Then use the following command.

```dcl
git clone https://github.com/rfoxoptimisation/rfoxoptimisation.github.io.git
```

You'll see the folder appear in your files and you can access and keep them updated using the instructions in the next section. If you make a mistake, you can always delete this folder from your computer and it will not delete the files from your Github account.

### You already have a folder on your computer

In this case, you'll simply need to walk the directory tree until the directory is the folder you want to link to Github. For example, if I want to link my 'test_repo' folder then I should have `C:\Users\ryanf\Documents>\test_repo` in the command line before I start using git commands.

Once you've done this, you use the following to create this as a repository in your Github account.

```dcl
git init
```

You'll only need to do this the one time. I remember wondering when I set it up whether I'd have to set up the link again the next time I turned on my computer. How could the link still be there? But it is there. And you can use 'git status' mentioned below to quickly check if you're unsure.

## Pushing and pulling

For me, I'm almost solely pushing files to Github and rarely pulling them. What this means is that I am more regularly creating files in the linked folder on my laptop and sending them to Github rather than doing it the other way round.

An example: I usually write posts for this blog using VSCode. I save these into a linked folder on my laptop and then use the below commands to send this to Github. 

The only case usually where I might make edits in Github are if I maybe get prompted to add a readme or if I make a small copy change there. In this case I'll then need to 'pull' the content from Github when I'm working from the command line again. You'll likely get a prompt from git telling you that you are one commit behind if you haven't made a pull request the next time you try to push.

Without further ado, here are those commands that will get you using git and feeling like a pro.

```dcl
git status
```

This is your sanity check and harmless command. You can't break anything with this command and it doesn't do anything other than give you a status update.

```dcl
git add .
git commit -m "Message here"
git push
```

```dcl
git pull
```
