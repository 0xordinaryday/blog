---
layout: post
title:  "Github Pages"
date:   2020-06-21 00:00:00 +1000
category: meta
---

#### Moving the hosting
So this site is made in Jekyll, posts written either in the terminal or in Joplin. Then I was copying the content to my Fastmail 'files' which can be used to serve a static website. I was doing the transfers from my PC to Fastmail with lftp.

#### What's wrong with that?
The main thing I *didn't* like about it was that lftp wasn't doing what I think of as syncing - it was replacing every file even if it hadn't changed. It worked just fine, but it wasn't very efficient. What if I want to add photos or something? I don't want to be copying stuff around all the time. I did briefly toy with the idea of writing a bash script to diff the files and only copy the ones I needed, but decided it wasn't the best use of my time.

#### Current version
So now I've moved over to Github Pages, which was intended for use with Jekyll anyway. I'm still using my own domain name, but the content is hosted in a GitHub repo and I can update the site just by doing a git push. Which is convenient, for sure. And it only copies across things that have changed.

#### Potential issues
Unfortunately you can only link to a Github Pages site with a public repo using a free account, which kind of mildly annoys me. The site is public anyway so it *shouldn't* bother me, but it kind of does.

Also, over time the repo will build up but it's only the current state of the blog that actually matters. So I don't really know that I want to keep a bunch of old commits in the repo, it's not like I'm going to want to revert the blog to some state it had 6 months earlier or whatever. I might look into whether it's possible to trim them or something. This wouldn't be an issue with how I was doing things earlier.
