---
layout: post
title:  "Reduce your Heroku slug size"
categories: general
location: Paris, France
location-link: paris
---

Finally built an app on Heroku but it takes forever to load when you share it? It doesn't give the best first impression when people have to wait a while before the page even appears to start loading.

![Slugsize breakdown >](https://blog.behrouze.com/wp-content/uploads/2018/10/heroku-logotype-horizontal-purple.png){: width="50%"}

This was my experience with my first Heroku-hosted app, which [calculates sample sizes for AB tests](https://abtestsamplesize.herokuapp.com/). The app was built through Python using the Streamlit library but before anyone could see the app I'd spent so long creating, testing on local servers, they had to sit through extra page load times unnecessarily.

The problem was the slug size of my Heroku app.

<!--description-->

> Slugs on Heroku consist of your application and all its dependencies

The recommended Slug size for apps is 300MB (with the max. 500MB) and somehow I exceeded this despite my app consisting of only a few python scripts.

*Note: In this post I am not talking about the load time after your Heroku app sleeps. This happens when you have the free Heroku subscription. Under this subscription, your app will sleep after 30 minutes of inactivity and take 5 or so seconds to wake up initially.*

## Check which files are taking up space

*Note: this article assumes you have the Heroku CLI installed. If you dont, follow the details on the [Heroku devcenter](https://devcenter.heroku.com/articles/heroku-cli).*

The first and most simple step in understanding why your slugsize is so large is to check which files are taking up space. We use Heroku bash for this.

```dcl
heroku run bash -a <appname>
```

This opens bash on the Heroku directory for the app specified in `<appname>`.

```bash
du -sh * | sort -hr
```

`du` stands for disk usage.

`-sh *` tells bash to return the disk usage for files and directories within the directory we have open.

![Slugsize breakdown ><]({{site.baseurl}}/assets/img/heroku-slugsize.jpg){: width="90%"}

Instantly, I can see that the virtual environment folder is adding 418MB to the slugsize of the app. The venv folder is not needed at all in my case and was actually added to my .gitignore file but only after it had already been pushed to git. This takes us onto steps to reduce your slug size.

## Ways to reduce slug size

### Remove cached files

For me, the main issue I had after running the above bash commands was that the virtual environment files were still uploading to Github and therefore being included as part of the app. Heroku doesn't need any of this to run my app. It uses a 'requirements.txt' file to know which libraries to install.

After ensuring that `venv/` was added to my .gitignore file, I removed the folder from the git cache using

```git
git rm -r --cached venv`
```

Remember to include `--cached` as this tells git to remove it from the cache only and not your local files. The above command triggered a satisfying avalanche of `deleted` file messages in the terminal, all unnecessarily weighing down my app.

Alternatively, if you want to clear your cache completely and start again, you can use `git rm -r --cached .` and re-add everything from your local directory. If your .gitignore is up-to-date, this will flush out any left over files hanging around the cache that may have slipped in before your .gitignore was fully fleshed out.

After my purge of the venv folder, all that was left in my Slug on Heroku were files of 8KB or less. After this had been deployed, the load time dropped to next to nothing and the app is small and nimble once again.

![Slugsize breakdown ><]({{site.baseurl}}/assets/img/heroku-slugsize-reduced.jpg)

### Create a .slugignore file

The same way you can create a .gitignore file, you can create one for your Heroku app.

For a different Heroku app, I created a REAMME.md for the Github repository where the app files were held. In this, I had included an image to help explain the purpose of the repository. It's also quite common to use a gif for this purpose. These can be big files.

Heroku doesn't need these files but Github does. Therefore you wouldn't want to ignore these in your .gitignore file but this is where you can use your *.slugignore*. The .slugignore works in exactly the same way as your .gitignore, exclude any files from your Heroku slug that are not needed in the app.

### Store assets somewhere else

Move image, video or audio files to a hosting service such as Amazon S3 or Dropbox.

## Summary

Speeding up the loading of your app is as important as anything else. There's no point spending loads of time making your app easy-to-use and great looking if people will be put off before it's had a chance to load.