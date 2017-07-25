---
title: "Beginner's Guide to Go Installation"
date: 2017-07-21T14:27:44-04:00
draft: true
---

I chose Go as my first language because it was designed with some higher-level computing concepts in mind. Sure, Ruby may be 'optimized for programmer happiness,' but leads at the Golang project want you to understand that, say, <a href="https://blog.golang.org/concurrency-is-not-parallelism" target="blank">concurrency is not parallelism</a>. That's pretty much Schroedinger's-catnip to my philosophy major brain.

There are, of course, a few more challenges for the complete beginner when wading into a language fitted for more advanced uses. And I hit them *really* early on. I assumed my experience would be:

<ul>
  <li>Step 1: Install Go</li>
  <li>Step 2: Get into all sorts of of yak-shaving scrapes trying to write Go</li>
</ul>

Instead it was:

<ul>
  <li>Step 1: Get into all sorts of yak-shaving scrapes trying to install Go</li>
</ul>

I'd therefore like to run through the knowledge gaps I hit while trying to make Go run on my machine. I think the Go <a href="https://golang.org/doc/install" target="blank">Getting Started</a> page assumes a little bit of basic programming experience, so if you too are a beginner, hopefully this will illuminate a few useful concepts and help you get to a working Go setup.

<b>Mysterious PATHs</b>

I use a Mac, so I was following the quick two-step instructions for the Mac OSX package installer. Let's talk about comfort levels with each piece of this guide:

"Download the package file, open it, and follow the prompts to install the Go tools."
Jufisch comfort level: 100%. Bring it on, GUI button-clicky installation. I'm great at this!

"The package installs the Go distribution to /usr/local/go."
Jufisch comfort level: 72%. Sure. Directories. I...know what directories are. Glad we're talking about this for some reason. This is fine.

"The package should put the /usr/local/go/bin directory in your PATH environment variable. You may need to restart any open Terminal sessions for the change to take effect."
Jufisch comfort level: 37%. ::sips coffee:: Oh, we're talking about the Terminal now? I thought we were GUI-ing this, Go.

"Check that Go is installed correctly by setting up a workspace and building a simple program, as follows. Create your workspace directory, $HOME/go. (If you'd like to use a different directory, you will need to set the GOPATH environment variable; see How to Write Go Code for details.)"
Jufsich comfort level: 0.29% and falling.

In a mere 30 seconds, I have transformed from a happy little Gopher to a melted pile of doggo. Because what even is a PATH, exactly?

<b>What Even Is a PATH Exactly</b>

The Linux Information Project <a href="http://www.linfo.org/path_env_var.html target="blank">describes PATH</a> as "an environment variable in Linux and other Unix-like operating systems that tells the shell which directories to search for executable files (i.e., ready-to-run programs) in response to commands issued by a user".

We'll need to involve the PATH variable when using Go because Go is a compiled language, and compiled languages take your code and compile them into executable files to be run. In order to run executable files on your machine, your machine needs to be 'aware' of the location of those executable files, and that's what PATH does. The Path environment variable is a set to be equal to a list of directory paths, one for any location you plan to run executable files.

I found it helpful to unpack this into a few more ideas.

<b>Running Programs: Compiled vs. Interpreted</b>

I won't delve too deeply into compiled vs. interpreted languages, but I liked <a href="http://www.programmerinterview.com/index.php/general-miscellaneous/whats-the-difference-between-a-compiled-and-an-interpreted-language/" target="blank">this description</a> over at ProgrammerInterview.com.

Let's talk about *running* code you've written in a compiled language like Go versus an interpreted language like Ruby.

With Ruby, you write a script foo.rb, and run it with the command "ruby foo.rb." This calls on the ruby interpreter (the "ruby" part of the command) to translate the text foo.rb into an intermediate representation that the Ruby Virtual Machine (which lives 'inside' the Ruby interpreter) can then run.

Not so with Go. With Go, you write a program called bar.go. When you're ready to execute that code, you run the command "go build" in your working directory. This instructs the Go compiler to take your bar.go file and  and then package them into a new executable file called 'foo.' To execute your code, enter the command "./foo."

As an executable file, ./foo needs to be in a directory that is saved in your PATH variable. Therefore, it matters where you're writing your Go program, and that's why Go has <a href="https://golang.org/doc/code.html#Workspaces" target="blank">a recommended Workspace configuration</a>.

<b>Go Workspace Setup</b>

 Go recommends you create a basic directory structure on your machine to serve as your Go workspace for all Go programs you intend to write. Within a parent "go" directory, you'll want three sub-directories:

 <ul>
   <li>pkg</li>
   <li>src</li>
   <li>bin</li>
</ul>

"Src" is for source code, your foobar.go files. When you run "go build" in any project directory (ie src/project/) you will end up with foobar executable files there.

To make sure
