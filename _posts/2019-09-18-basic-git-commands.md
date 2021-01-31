---
layout: post
title: "7 basic git commands to get you started"
date: 2019-09-18 18:00:00 +0700
categories: general
location: Huaraz, Peru
location-link: Huaraz
image: https://tr3.cbsistatic.com/hub/i/r/2017/10/31/af72d5e4-2f4c-48b5-954c-e4fa24fb0a97/resize/1200x/9f5c03620b98aa0a8d1a3caedded38fe/git-logo.jpg
---

![Git logo >](https://miro.medium.com/max/910/1*-l3Qrum9oPMiiM0yW7v_Ng.png){: height="100px"}

Getting started in git takes 10 minutes and provides a quick and simple way to ensure your scripts are saved online and gain version control. It's also makes you feel like a pro using the command line interface (CLI). In this post are the 7 commands you need to manage a simple repository.

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

This is it. All you'll need to get you started if you're simply trying to maintain a basic respository. Keep reading for more detail on what these functions do.

{% include toc.html %}

## Downloading and using git

To **download git** to your machine, go [here](https://git-scm.com/downloads).

**Using git** is done from your command line or CLI. Most likely you can open a terminal from your text editor as well. Both Jupyter Labs/Notebook and Visual Studio code have a CLI.

You can check that git has installed correctly by typing in `git --version` into the command line. This should return the version number installed.

## Getting started

There are two ways first make the connection through git between Github and your computer

1. You create a folder on Github and clone it to your computer
2. You create the folder on your computer or use an existing one

For both of these steps, the first thing you'll need to do is navigate your folder directory via the command line. This is essentially pointing the CLI at your desired location. Learning how to do this is a key skill for any would-be programmer.

### Option One: Creating the folder/repository on Github

For those who have created a repository already on Github, you'll need to **clone** the folder to your computer. First, navigate to the parent folder where you want your repository to appear.

For example, if you named your repository 'test_repo' on Github and you want it to appear in 'C:\Users\ryanf\Documents>' then you would make this your directory in the CLI using `cd C:\Users\ryanf\Documents>`.

Now you'll need to grab the clone URL from your repository on Github. Then use the following command:

```dcl
git clone https://github.com/rfoxoptimisation/rfoxoptimisation.github.io.git
```

You should see your repository appear as a folder in your files.

### Option Two: You already have a folder on your computer

In this case, you'll simply need to walk the directory tree until the directory is the folder you want to link to Github. For example, if I want to link my 'test_repo' folder then I should have `C:\Users\ryanf\Documents>\test_repo` in the command line before I start using git commands.

Once you've navigated to here, use:

```dcl
git init
```

Now, to finish the link to Github, create a repository on [Github](https://github.com/) and follow the instructions for 'Push an existing repository…'.

These getting started steps you'll only need to do this the one time. When I first set this up on laptop, I wondered whether I'd have to set up the link again the next time I turned on my computer. But it will still be there and you can use `git status` mentioned below to check.

## Everyday git

Now you've got your folder on Github and your computer, here are the everyday commands you'll need to keep your repository up-to-date.

### Status check

```dcl
git status
```

This is your sanity check and harmless command. You can't break anything with this command and it doesn't do anything other than give you a status update. This will tell you what has been modified, deleted or added. I will usually run this in the CLI to before doing any commits or pushes as this will tell you what's to be updated

### Pushing vs pulling

For me, I'm almost solely pushing files to Github and rarely pulling them. What this means is that I am more regularly creating files in the linked folder on my laptop and sending them to Github rather than doing it the other way round.

An example: I usually write posts for this blog using VSCode. I save these into a linked folder on my laptop and then use the below commands to send this to Github.

The only case usually where I might make edits in Github are if I get prompted to add a readme or I make a small copy change. In this case I'll then need to 'pull' the content from Github when I'm working from the command line again. You'll be prompted from git if you need to make a pull request the next time you try to push.

### Add, commit and push

These, along with `git status` are the three commands you'll find yourself using every single time you use git.

Once you've made changes to your project and you are happy to add this to the repository formally, use the following.

```dcl
git add .
```

Using the full stop at the end adds all the changes you've made to any files in the folder. If you use `git status` before and after adding the files, you'll see that the files move from 'Changes not staged for commit' to 'Changes to be commited'. This is the next step.

```dcl
git commit -m "Message here"
```

This is the second step in pushing your change to Github. Write a commit message so you can see what changes you made to your files. Warning, if you forget the `-m "[message]"` then you're going to find yourself in a weird Vim situation. [Here](https://medium.com/rightindem-design-team/how-to-never-get-stuck-in-vim-again-edaee4d632bd) is an article I enjoyed about how to get out of this.

```dcl
git push
```

This final command will push your changes to Github. All in all a very satifying process.

### Pull

This is for those times mentioned above when the repository you have on your computer is behind the latest commits.

```dcl
git pull
```

Running this command will bring you up to date and if you're working on a larger project, you will find yourself needing this often along with many other commands out of scope for this post.

## Conclusion

That's it! These commands will take you far and you'll find them coming to you very easily after a few commits and pushes. Actually using the command line for a legitimate reason felt like a big step into the programming world and I hope this will help anyone going forward as well.

You now know how to:

- Initialise and clone a repository to Github
- Check the status of your repository
- Keep your repository updated using add, commit and push
- Pull the latest version of the repository from Github

There's no need to use Github desktop when it's this simple and it looks far more professional to use git from the CLI.
