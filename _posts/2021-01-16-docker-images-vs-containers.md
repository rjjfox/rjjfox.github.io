---
layout: post
title: "Docker images vs. containers - how I explained it to myself"
categories: general
location: London, UK
location-link: london
---

I used Docker and containers recently as a way to simplify deploying web apps on cloud platforms such as Google Cloud Platform. I kind of hacked my way through it, cutting and pasting bits and pieces from Dockerfiles I'd seen lying around on the web. I got it to work but it didn't feel great.

I've since dedicated time to really understand what I'm doing with Docker and understanding the terminology. I want to take you through part of the learning process with me and I'll focus this post on the terminology of Docker containers and images. What are they really and how do we use them?

![Docker logo ><](https://pbs.twimg.com/profile_images/1273307847103635465/lfVWBmiW_400x400.png)

<!--description-->

{% include toc.html %}

## Images vs. Containers

My process for really trying to understand a new topic tends to be to Google the question I want answered, open a bunch of articles in multiple tabs, go through some of the articles, open more tabs for things I don't understand. Normally I'll also watch a video or two of someone giving a brief overview of the topic.

Doing this to understand simply the difference between images and containers, on one website I read that "an image is an instance of a container" which made limited sense to me at first but I let it go. I then read on another site, "a container is an instance of an image" 😕. I had to go back and check that I didn't read it wrong in the first article. I wasn't mistaken. I was taking notes while looking into all this and I'd just written the reciprocal statement. And it is true.

Now to explain how it can be true. Doing so should explain what the difference is between images and containers.

### Images are Immutable - They Can't Be Modified

This is important for understanding why the above two statements are true. For most of us, images are created from Dockerfiles. With a Dockerfile, we run

```docker
docker build [path-to-Dockerfile]
```

This builds an image. This image cannot be edited. We can delete it or duplicate it but we can't change an image once it's built. If we edit the Dockerfile and run `docker build`, we will create a new image, not edit the original.

### Run an Image to Create a Container

To create a container, we run an image using

```docker
docker run [image-name]
```

This process alone is useful to remember, a **Dockerfile** → _build_ → **Image** → _run →_ **Container**.

![Docker process]({{site.baseurl}}/assets/img/docker-process.png)

_Full image credit goes to [Jake Wright](https://www.youtube.com/user/jaketvee) and his great Youtube video referenced below._

### Layers

Images are made up of layers. Many many layers.

When we write a Dockerfile, it may look something like this

```docker
FROM python:3.9-slim

WORKDIR /usr/src/app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "coviddashboard:app"]
```

This is an example Dockerfile that will build an image for a Flask app that I built.

The line I want to point out here is `FROM python:3.9-slim`. This here is called the base image/layer. `python:3.9-slim` is an image that we are using as our base. It is also called the base layer because when we build an image from this Dockerfile, Docker will run through each line in the file and execute each as a new layer. They are built upon the base image/layer which will be the `FROM` line **which must always be the first line in a Dockerfile**.

Once it has been through each line, we have an image. However, it's a useful exercise to dive down into the base layer. If we go to [Docker Hub](https://hub.docker.com/_/python), and look at the description for the Python image, we can take a look at the Dockerfile for the `python:3.9-slim` image. This Dockerfile contains a lot of stuff but starts with the line:

```docker
FROM debian:buster-slim
```

It's using a `Debian` image for its base. We can take a look at the Dockerfile for this image too and here we finally get to the bottom of the base layers and see:

```docker
FROM scratch
```

The Debian image was built from scratch.

### Why is a container an instance of an image and vice versa?

For me, the idea that a container is an instance of an image is more readily understandable. We run an image to get a container. This is why the container is an instance of an image.

To understand why an image is an instance of a container, we need to remember that images can not be modified. We also need to remember that when we run an image, we get a container.

Containers can be modified. We can use a container to open a terminal and run commands, we can add files or spin up servers. We can do what we want with containers and that's why we love Docker. When we create a container from an image, to be able to make changes, a new modifiable or writeable layer is added. To this layer we can do what we want. Once we've made our changes, if we then want to share what we've done, we can create an image or snapshot of our container and share it. This is why an image is an instance of a container. It is a snapshot of the container we have built that we want to share.

This could be the perfect working conditions to work with Python/R using favoured versions and the right OS. It could be your web app running on a server with all the right dependencies installed. We create an image from our container and share it with whomever.

## My analogy for understanding

After all the articles I went through, the way I summed it up to myself was this. I saw the "snapshot" word used often in articles when talking about Docker and took this into further into a photo analogy.

> **Images** are snapshots, **containers** are like printouts of these snapshots that we can then edit by writing over them with a pen. We can then take a snapshot of our edits to make it into an image that we can share with our friends.

This probably isn't much help without the long explanation before it but this is how I finally explained it to myself.

## Articles and videos that helped me further my understanding

I hope to have helped you in your journey understanding Docker. I haven't spent much time going over why you should use Docker, although I've alluded to it, since I believe there are many articles trying to explain this. Here are a few resources to help you further your understanding that I found useful.

- [Learn Docker in 12 Minutes](https://www.youtube.com/watch?v=YFl2mCHdv24&ab_channel=JakeWright) - a great starting point/refresher. I found this useful even after creating one or two Dockerfiles. This explained the workflow of Dockerfile to Image to Container and got me going down the base layer rabbit hole. I watched this while in the shower ¯\*(ツ)\*/¯.
- [What is a Docker Image and How is it Used?](https://searchitoperations.techtarget.com/definition/Docker-image) - Image is an instance of a container article
- [Docker Image vs Container](https://stackify.com/docker-image-vs-container-everything-you-need-to-know/) - Container is an instance of an image article
- [The Docker Book](https://dockerbook.com/) - I haven't gone through too much of this yet but it seems a great way to get more familiar with everything. I have found chapter 4 most useful so far.
