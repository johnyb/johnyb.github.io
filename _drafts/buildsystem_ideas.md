---
layout: post
title: What should a buildsystem do?
tags: [grunt ox]
---
A few months ago, we decided to kind of throw away our old, jake based build system and implement
a new one using grunt.
Taking such a step is not easy and it is arguably, if it’s worth the effort.
I want to discuss a bit the motivation behind this project, our goals and problems we are facing.
May be, this can be of help if you are in a similar position and need to decide wether to take on
such a project or try to somehow fix your current workflow and not write everything from scratch.

So let’s start at the very beginning, the structure of our project and our ideas about how to provide
a build system to simplify all those stupid tasks you repitavely need to do as a developer.
After that I want to discuss the problems I saw with the old system and first ideas what I wanted
to do about them.
A description of the current situation, after about two months, when two developers have worked hard
to implement a new buildsystem, should give you some hints on what to expect when starting such a
project.
I want to conclude with things what we have learned and our plans for the future.

### Our project

The project, I'm working on, provides infrastructure for a complete ecosystem of web apps, that can be
composed and run together.
All those apps are driven by a backend (others might call it middleware) that won't be discussed, here.
I will only talk about the frontend part, written entirely in JavaScript.
The core provides a basic set of applications, a default theme, some libraries to build own apps and
(the part all this should be about) a build system or SDK, to help external developers to write and
deploy own modules to extend core functionality.

All this is pretty complex and quite hard to get right, but when I joined this project over a year ago,
I was able to work with tools that really made my life as a developer easy.
The old system was based on [Jake](https://github.com/mde/jake) and provided all the tasks needed for
development.
In order to use the build system, it was mandatory to install the SDK package, that provided a copy of
the main Jakefile and a few libraries to implement larger tasks.
This package also provided a middleware proxy, the [appserver](#references), that was a JavaScript implementation of
an API provided by our backend, a proxy to direct requests towards a real backend and some extensions
you want to have for development.
This middleware proxy topic should be subject of a later post, I will publish through this blog, since
we solved some interesting issues with this approach and those might be useful for others.

So basically, what we had, was a quite large library of self-written scripts that made up a complete build
system.
If some developer needed a new feature, the core team implemented it and distributed it through the SDK package.
If there were bugs, core team fixed them and distributed the fixes through the SDK package.

### Problems with the old system

### Our solution

### Conclusion

### References
