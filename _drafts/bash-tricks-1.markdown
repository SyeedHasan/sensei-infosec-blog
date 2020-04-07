---
layout: post
title:  "Bash Tricks: Killing Processes!"
date:   2020-04-06 12:00:19 +0500
categories: bash tricks 1
---

Hi there, I hope you're all doing great! 

This blog is a short one - but we'll be discussing a cool trick I learned a while ago.

**What are we up against?**

Let's say you've written a bash script to run some tasks on a machine. Unfortunately, the script's taking way too long for you to get through it. It's Friday, you've decided to sit in late. It's still taking too long - you've decided to quit it this time round and run it back on Monday. 

Fortunately, you've got two days to decide whether you're going to run it as-is, hoping it works out this time. OR, you can implement a _solution that's bound to kill the process (the script) after a specified time._
 
Now I know, what purpose does it serve if I simply quit a script? Well, it's technically going to complete it's intended tasks to whatever extent it can and only then quit - leaving it's previously done work intact. 

Again, if you've landed here by mistake - this might or might not help you out. But, let's get to it and see if it matches your needs. Here's a function I've written (by several hit and trials!) that does exactly what we've talked about:

_Don't worry I'll paste a link to the Gist at the end of the blog!_

![alt text](/assets/bash-tricks-1/code.png)

