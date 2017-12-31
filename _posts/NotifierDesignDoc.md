---
layout: post
title: Solving the Category Notification Problem
date: 2017-12-31
---

# Category Notifier Facebook Bot [Draft]
This is a draft and will likely get updated constantly.

## Background/Research
Frequently, my friends and I go to trivia at a local bar in Seattle. This bar posts it's trivia categories weekly, and then I copy and paste the categories into a group chat I have with my friends.  Ocasionally I forget to do this, or it takes a while. This begs to be automated.

### Getting the Categories
This is super easy - Facebook has a [Graph API](https://developers.facebook.com/tools/explorer?method=GET&path=cha.seattle%2Ffeed&version=v2.11) which lets you easily get the posts, as well as a [webhook](https://developers.facebook.com/docs/graph-api/webhooks/) which goes back to a URL I can control. Then I can do whatever (like call a post API).   

### Sending the Message
This is unfortunately not super easy - the only APIs Facebook has for sending messages are through it's [Messanger Platform](#https://developers.facebook.com/docs/messenger-platform), which only let's you send messages to people who subscribe. It would be nice if it just had an API to post a message to an arbitrary conversation - that's all I really want. 

## Approach

I'm going to make a Facebook bot that allows a user to subscribe to get a message anytime a page posts an update that meets a particular regex. They'll be able to specify the page and the regex in the API. I'll have a page where my friends can just click to get the already prepoulated messanger sending it to them. 

I'll create a web app, which will have one page, where you can click to sign up to recieve messages from a specific regex for a specific page. I'll start (and likely end) with only supporting Cha Cha, so I will create the web hook subscription by hand. I'll start with hard coding my regex in the web app, but the backend will have an argument for that. 

When the user signs up, they will use the [checkbox plugin](https://developers.facebook.com/docs/messenger-platform/discovery/checkbox-plugin) to authenticate. In the database, I'll have a `users` table with a GUID primary key, which I will pass to facebook as the `ref` field. In that table, I'll have a one to many relationship to the `regexes` table. Each regex will be identified by a GUID primary key, and will have a many to one relationship to the `pages` table. The `pages` table will use the facebook `page-id` as the primary key. In my MVP, I'll only need the users table, since the regex and the page will be defined by me, to send the trivia categories to my firneds.      

The webhook callback will hit an endpoint `/pages/[page-id]/post`, with the content of the post. Each page will have a list of regexes, and each regex will have a list of user accounts. I'll then send a push notification using the [bot send api](https://developers.facebook.com/docs/messenger-platform/send-messages#send_api_basics) using the `NON_PROMOTIONAL_SUBSCRIPTION` message type, to the `user_ref` we store when the users signs up and authenticates. 
