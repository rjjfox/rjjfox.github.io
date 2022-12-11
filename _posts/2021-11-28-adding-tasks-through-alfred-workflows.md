---
layout: post
title: "Adding tasks quickly through Notion"
categories: general
location: London, UK
location-link: london
image: https://images.ctfassets.net/lzny33ho1g45/alfred-p-img/71663bbc6930f55fab6b7967399d53fb/file.png?w=1520&fm=jpg&q=30&fit=thumb&h=760
---

One of my favourite apps for productivity on the Mac is [Alfred](https://www.alfredapp.com/). It's a suped up version of Spotlight or for the windows comparison it's like the Windows search function. I use Alfred around close to 100 times a day to launch apps, search for files, do Google searches, grab items off the clipboard history and more.

<!--description-->

Another one of my favourite tools is [Notion](https://www.notion.so/). Notion is an all-in-one Notetaking workspace. I use it to capture my learning, track projects, jot down notes and a thousand other things.

To keep track of tasks at work and prioritise accordingly, I'm currently trialling the '[Getting things done](https://en.wikipedia.org/wiki/Getting_Things_Done)' framework. A key feature to the GTD approach is that you offload tasks from your mind into an 'Inbox' which can be sorted later. It's important to do this while you remember the tasks, which is often not very long.

Notion is the tool I use to capture the tasks, however, speed can be an issue with Notion. Opening Notion and navigating to the right area can take a little time and a few clicks. This is where Notion and its amazing workflows comes in. Workflows are only available if you pay for a 'Powerpack' but it opens up an amazing amount of customisation.

![Demo of the workflow]({{site.baseurl}}/assets/img/alfred-workflow-demo.gif)

Check out the workflow on [Github](https://github.com/rjjfox/notion-new-task-alfred).

## A note on doing this with Python

I did initially set out to do this through Python, before realising that I can simplify the process using a bash script instead. Bash means that the workflow is much more transportable than if you add Python as a dependency. Not everyone has Python installed on their machine whereas anyone using Mac OS will be able to run the bash script.

For those interested, here's a peak at the Python required.

```python

import requests, json, sys

# This is the token provided by the integration API
# https://developers.notion.com/docs/getting-started#step-1-create-an-integration
token = "[KEY]"

# Found in the URL on the database page
# https://developers.notion.com/docs/getting-started#step-2-share-a-database-with-your-integration
databaseId = "[ID]"

headers = {
    "Authorization": "Bearer " + token,
    "Content-Type": "application/json",
    "Notion-Version": "2021-05-13",
}


def addTask(databaseId, headers, name):

    createUrl = "https://api.notion.com/v1/pages"

    newPageData = {
        "parent": {"database_id": databaseId},
        "properties": {"Task Name": {"title": [{"text": {"content": name}}]}},
    }

    data = json.dumps(newPageData)
    # print(str(uploadData))

    res = requests.request("POST", createUrl, headers=headers, data=data)


```
