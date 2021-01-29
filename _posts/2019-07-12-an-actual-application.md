---
layout: post
title: "A Basic Command Line Application"
date: 2019-07-12 16:00:00 +0600
categories: python
location: Santa Cruz de la Sierra, Bolivia
location-link: Santa+Cruz+de+la+Sierra
image: /assets/img/ColourLookup.gif
---

Continuing with Al Sweigert's 'Automate the boring stuff with Python' book and course, I completed the first section 'Python Programming Basics'. I found this section slow but, naturally, quite important. At the end of it came the first truly exciting part which is covered in the Apendix in the book. It came in the first 'Chapter Project' of the book, where you are given a project that will demonstrate the concepts learnt in the chapter. The example given is a (insecure) password locker.

<!--description-->

The idea is that you could call a password locker from the command line which will spit out the password for a specific account and paste this onto your clipboard, ready to be pasted. This is extremely simple, but very satisfying to see it work from the command line and I was quite pleased with myself afterwards.

From this, I tried to find an alternative use and decided to create a colour lookup tool for my partner. She often creates illustrations and has a set a colours she often uses for her designs. Currently, she stores the hex codes on a text file that's easy enough to pull up and I decided to pull this information into my program so that I can pat myself on the back, which I'll do regardless of whether she uses it or not.

The code required for this is incredibly simple and yet being able to do this makes me feel like I've ascended to the next level of computer nerdiness as all command line/power shell commands will do.

Here is a short gif showing the lookup in action (by me, she hasn't used it yet :\|).

![Colour lookup ><]({{site.baseurl}}/assets/img/ColourLookup.gif)

It also works from the run application.

![Colour lookup from the run line ><]({{site.baseurl}}/assets/img/ColourLookup_Run.gif)

The code is barely 15 lines not including the dictionary which can be any length. Required is a dictionary, some if statements and some amazing pyperclip functions. To make it run from the command line or Run application requires a Batch file and potentially editing the PATH variable. As I said, it's simple yet very satisfying to see it work and I'll be looking forward to further applications.

<script src="https://gist.github.com/rfoxoptimisation/84c64356f4f95a0e94290b548ab182da.js"></script>

Now whether this is an actual application is debatable as I don't know if it will be used but the book is certainly now getting into the applicable and more applications should be forthcoming.
