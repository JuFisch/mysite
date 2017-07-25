---
title: "Go Installation for the Complete n00b"
date: 2017-07-21T14:27:44-04:00
draft: true
---

As a n00b, I picked Go as my first language because it was created with some higher-level, extremely non-n00b concepts in mind. Learning Go meant learning a lot about how code works, not just learning how to code. (You had me at <a href="https://www.youtube.com/watch?v=cN_DpYBzKso" target="_blank">"Concurrency is not Parallelism,"</a> Rob Pike.)

This is awesome and extremely motivating, but for a n00b programmer, the road to Go programming is paved with - oh god, help, I tried to turn off onto Go Road and hit a huge pothole, my tire exploded, I cannot possibly drive down this road.

The issue is that Go documentation assumes a certain degree of general programming experience. Not much, but some. As I've tried to pick it up, I've found very few Go guides aimed at people new to Go *and* new to programming. 

Here I'll highlight some of the knowledge gaps I hit trying to get Go running on my machine. I hope it'll be useful for any other Go newbies out there. 

<b>Package Installers</b>

I use a Mac, so I followed the simple instructions for the Mac OSX package installer. The first thing the Go Docs tell you is: 

"Download the package file, open it, and follow the prompts to install the Go tools. The package installs the Go distribution to /usr/local/go.

The package should put the /usr/local/go/bin directory in your PATH environment variable. You may need to restart any open Terminal sessions for the change to take effect."

To quote a <a href="https://www.destroyallsoftware.com/talks/wat" target="_blank">really wonderful programming talk</a>:

![This is an image](/img/watcat.jpg)

I recognize /usr/local/go as a directory, but why do I need to do know that, Go Docs? In addition, what the $%#& is a PATH environment variable? Confusion rising... 

Then the docs say:

"Check that Go is installed correctly by setting up a workspace and building a simple program, as follows.

Create your workspace directory, $HOME/go. (If you'd like to use a different directory, you will need to set the GOPATH environment variable; see How to Write Go Code for details.)"

OH right, so I just set the GOPATH environment variable, that'll be easy I'll just:

![This is an image](/img/halpcat.jpg)

If the above two pictures apply to you, then perhaps you, like me, had no idea what a PATH was! 

<b>What the hell is a PATH?</b>

Let's just step back for a second. 
draft: Note -- Path is for EXECutABle files! is that why I have to use them for go and not things like ruby?

 
