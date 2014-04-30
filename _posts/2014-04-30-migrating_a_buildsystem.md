---
layout: post
title: Migrating your JavaScript build system
tags: [grunt, ox]
---
A few months ago, we decided to—in a way—throw away our old, jake based build system and implement
a new one using grunt.
Taking such a step is not easy and it is debateable, if it’s worth the effort.
This post should shed some light on the motivation behind this project, our goals, and problems we are facing.
Maybe, this can be of help if you are in a similar position and need to decide whether to take on
such a project or try to somehow fix your current workflow and not write everything from scratch.

So, let’s start at the very beginning, the structure of our project and our ideas about how to provide
a build system to simplify all those stupid tasks you repetively need to do as a developer.
After that, the problems we saw with the old system will be discussed, as well as the solution our team
implemented after working hard for about two months.
This should give you some hints on what to expect when starting such a project.
Finally, a few points, that we have learned during this process and our plans for the future will conclude this
post.

### Our project

The project we are working on, provides infrastructure for a complete ecosystem of web apps, that can be
composed and run together.
All those apps are driven by a backend (others might call it middleware) that won't be discussed, here.
The focus will be on the frontend part, written entirely in JavaScript.
We call it the core UI, or simply “Core”.
The Core provides a basic set of applications, a default theme, some libraries to build own apps, and
the part all this should be about, i.e. a build system or SDK to help external developers to write and
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
This middleware proxy topic should be subject of a later post, that will be published through this blog.
We solved some interesting issues with this approach and those might be useful for others.

So basically, what we had, was a quite large library of self-written scripts that made up a complete build
system.
If some developer needed a new feature, the core team implemented it and distributed it through the SDK package.
If there were bugs, core team fixed them and distributed the fixes through the SDK package.

### Problems with the old system

One benefit of the old solution is the possibility to base your work on best practice
and learn from others.
So in our case, there are quite a few power users you could ask to solve any problem you might encounter.
This is definitely something, we wanted to keep.
However, the centralized approach was what I disliked most about the old system.
Every team is different and I think, it should be the decision of the team, which tools to rely on.
Furthermore, different projects may demand different approaches, so the system should be customizable.
Of course, this has been guaranteed with the usage of JavaScript and it was possible to patch everything you wanted, but this
should be avoided.
In the end you might end up writing everything on your own, because you did some incompatible change.

Another problem is the structure of the code.
Most tasks have been coded into one large file, except for some really complex functionality.
Anyway, the Jakefile was needed to glue all this together.
IMHO this was hardly maintainable.
I had a hard time reading through the code to understand, how it works.
Extending the system, like installing a new library, was quite hard.
Lots of copy & paste has been involved and there was a huge section that mostly looked the same all over,
making it really hard to spot the differences.

There was also nearly no separation between the users of this build system.
Every project used the same code base, additions (even specific ones) had to be done in the Core.
I'm not even sure if it was possible to overwrite existing functionality in own projects.
I have never used this feature and I am not aware others did.

Then there was the point about dependency management.
This is not part of a build system, I know, but worth to be mentioned, at this point, I believe.
We had a `lib` directory that contained the sources of all libraries, we depended on.
There were internal libraries, like for the build system itself, some extensions to thirt-party libraries, and so on.
Then there were third-party libraries, that we used, like Twitter Bootstrap, jQuery, Backbone and others.
Some of those libraries contained patches, that were never integrated upstream.
Those libraries have been put there by hand, so no management tool has been used.
It was hard to keep track of the versions of the libraries.
We had to maintain a list somewhere.
Updates have been rare.
Sounds scary? It was.
Somehow, we managed not to constantly break everything over a long time.

Last but not least, there is the point about adoption.
Most JavaScript projects have been using grunt as a build system, for quite a while.
When our jake-based system was started, jake might have been the best player on the ground, but nowadays
at least you want to look, what others have done meanwhile.
Of course this is no reason to throw away a working system, but it might start discussions, if switching
could improve the situation.

When I started working for OX in January 2013, I have never seen nor worked on a larger project written in
JavaScript.
I had some experience with Ruby-On-Rails, where I learned how cool Behaviour-Driven-Development is and I've been
practicing it on some projects, before.
Our project had tests, back when I started, but I was never shown, how to run them.
So I basically considered them dead, when I found out about them.
But the fact, that there were tests, made me to think about this topic and that I wanted to do something about
automated testing in our project.
Testing JavaScript is out of the scope of this article, but I promise, I will write up on this topic, sooner or later.
I ended up playing around with Testacular, nowadays better known as [karma](http://karma-runner.github.io/),
a test runner for JavaScript tests.
Really nice tool, I suggest to check it out, if you haven't heard of it.
I started to integrate it into our jake system, it kind of worked, but it felt wrong.
Everything had to be done manually.
The documentation said, just use this grunt plugin and you are done.
It took me several days (okay, I was pretty new to the JavaScript world, back then) to get this up and running.
Anyway, I wasn't thinking about switching the jake based system, but I felt, we could do better.

### Our solution

At some point, we (so at this time, it was not only me any longer) started to think, that we had to do something about this.
We had a closer look into grunt and learned how grunt works and what it can and can not do.
After playing around with it, we decided to drop the old system and rewrite everything from scratch using grunt.

Many common tasks were very easy to implement, because there are already a lot of [grunt plugins](https://gruntjs.com/plugins)
available.
I will cover the interesting tasks in more depth with separate posts, but to give you an idea, what we are basically doing,
take a look at this overview:

* `build`
    * `jshint` all source files
    * `jsonlint` all json files
    * `copy` all source files to a `build/` directory
    * `less` compile all less files from the source directory
    * `concat` special manifest.json files (those define "apps" within our "appsuite") into one large .json file
    * compile i18n (po) files into require modules (more on that, later)
* `watch` source files for changes
    * run `build` if any source file changes
    * run tests after new files have been built and testserver is running
    * reload any open browser pages connected to a running server
    * run other tasks depending on the type of the file that changed
* start a man-in-the-middle proxy server to develop the frontend using a "real" backend
    * inject built code if locally available
    * dynamically register new apps served locally
    * more on this technique in a different post
* start a test server to run tests from different browsers
* run tests on all connected browsers
* extract i18n strings from JavaScript files (more on that, later)
* `dist` - create a distribution ready version of the software
    * remove previously generated files
    * check that all dependencies are satisfied
    * `build` the software
    * run JavaScript files through uglify
    * copy all other files into `dist/` directory
* `install` built files or distribution ready files to an arbitrary location (useful for packaging)
    * provide special location for files that should be installed on the Webserver (static files)
    * provide special location for files that should be installed on the "backend"/middleware server (dynamic files)
    * provide special location for i18n related files (independently for each language)
* `bump` the version of the project

This list contains the tasks, we implemented for the grunt based system.
The jake based one only did a subset of these tasks but of course it would have been possible to just extend our Jakefile.
However, what we built was a generic grunt configuration to develop new UI modules for the OX Appsuite product.
Our initial idea was to distribute this configuration to everybody interested in writing UI modules for Appsuite.

So what we did was creating a generator for [yo](http://yeoman.io) that can be used to generate a UI module project.
We stored the basic configuration within yo templates and ported the first projects (including the core UI) to this approach.
This made the core UI a module like everything else.
So there is no difference between the core UI and any other module that is serving as a client for the OX backend.
Sounds decoupled, does it?
So this was pretty much a success.
We still have a shared configuration for the build system, but every part of the ecosystem is equal.
It is also possible to completely diverge from the shared system and overwrite everything directly for one
project.

### Evaluation

All this went live a few weeks ago.
For quite some time, there has been nearly no feedback.
We tested the conversion from the jake based system to grunt with a project, we consider our largest “external Project”.
In fact, this project belongs to the OX family, but is developed by a completely different team in a different city.
Everything worked fine, there have been a few issues with the configuration, we fixed those and regenerated the configuration
for each of our projects using yo.

Then there is the custom development team, maintaining many small plugins for individual customers.
We started to port the first projects to the grunt based system and everything worked as expected.
The developers also found a few issues in the standard configuration that have been fixed quickly
in the templates.
At this point we recognized, that this approach doesn't scale.
It's pretty much work to update every single module, if some configuration changes.
The approach using yo to manage the configuration is not really the way to go.
This might be different for other projects, though.

I had some sleepless nights and thought about this issue and I'm really glad, I found a solution.
Looking back, it is pretty simple, but a few weeks ago, we didn't think about it.
During development, I stumbled upon an issue that was introduced by a grunt plugin.
The [assemble-less](https://github.com/assemble/assemble-less) grunt plugin had an
[issue](https://github.com/assemble/assemble-less/pull/27), I fixed when working on all this stuff.
This is exactly, how open source works.
You take a look at other peoples solutions and use it in your own projects.
If you think, “upstream” does something wrong, you open a pull-request and hope to get it fixed
upstream.
Never the less, this bug brought me to the solution, that was needed in our case.
The problem was, that assemble-less manipulated the global grunt configuration object and
unintentionally removed parts of our configuration.
So the solution to our problem was to provide an npm module that will bring a large object
containing our shared configuration that can be loaded from any `Gruntfile.js`.
The shared configuration also extends grunt to allow extending the grunt configuration easily.
Eventually, this can also go upstream into grunt, one time.
So what we have now is an [npm module](https://github.com/Open-Xchange-Frontend/shared-grunt-config)
that can be added as a dependency to any project.
It will extend the grunt configuration with everything that is needed to build an OX Appsuite UI module.
After loading it within any `Gruntfile.js`, this configuration can be overwritten as you like.
A complete documentation of all tasks will be provided in the
[readme](https://github.com/Open-Xchange-Frontend/shared-grunt-config) file of the
[shared-grunt-config repository](https://github.com/Open-Xchange-Frontend/shared-grunt-config).

This latest approach will serve as good as the first, yo based one, to manage a central shared configuration.
It is scalable to update many projects at once.
We will stick to [semantic versioning](http://semver.org) when adding new features or changing configuration.
So the only thing you need is to add the npm module to your `package.json` and `npm install` will bring
you everything, you need.
Of course, this change has been added to [generator-ox-ui-module](https://github.com/Open-Xchange-Frontend/generator-ox-ui-module),
the yo generator that can be used to get started with new Appsuite UI modules.

### Conclusion

All of this has been very exciting during the past months.
I'm looking forward to publish many of our design decisions through this blog in the next time.
After collecting all this feedback, I'm pretty sure this project will turn out well.
All in all I consider this a success.
I will try to explain some problems, we encountered during this mission.

First of all, we call ourselfes a team following an agile workflow.
This means, projects will take at most the time of one sprint (using scrum wording, here).
What we did, took more than just the time of one sprint.
We started to work independently on that project, but when we merged back our changes,
it took way more than one sprint to work everything out.
Maybe, this could have been managed in a different way.
All in all, grunt is a so called `task runner`.
We could have started with using grunt to run our basic jake tasks, calling jake directly.
This would have been less intrusive and we could have collected feedback early.
From this point, it would have been possible to start changing things and provide more
and more configuration.
When we started working on this project, this option was never considered.
All we wanted, was to get rid of this unflexible thing we had before and get something new and usable into
our workflow.
I guess, we managed to do that, but it has been quite some work.
Maybe, with some more planning (round about the time we needed to implement all this … ;P) it would have
been possible to get this project working more smoothly.

Due to all of the feedback, we gathered during the last few weeks, we were able to fix most of the issues
and deliver something that will work in most of the cases we have thought about.
To handle all the other cases, we are now flexible enough to handle these cases and collect even more
feedback for future versions.
We are not as much dependend on the core UI any longer, as has been the case until now.
All you need to get started is the [generator-ox-ui-module](https://github.com/Open-Xchange-Frontend/generator-ox-ui-module).
If you are already using npm, you could just add our [shared-grunt-config](https://github.com/Open-Xchange-Frontend/shared-grunt-config)
to your dependencies, load them in your `Gruntfile.js` and be happy.

What has only been partly solved with all this is the topic of packaging.
I will cover that topic soon and provide more information on how it is possible to provide native packages for your module for the
Linux distribution of your choice.
It is possible to do that in a nice way, we have implemented it successfully during the past days.

There are still some interesting tasks to be solved.
We will continue to collect feedback from our fellow developers and release new versions of the
[shared-grunt-config](https://github.com/Open-Xchange-Frontend/shared-grunt-config) from time to time.
This post only gives a rough overview of the things we need to solve during day to day development.
Please stay tuned, I will go into more detail with one of the tasks, our build system is solving for you as a developer, in one of
my next posts.
If there are some interesting changes, I will make sure to also post them here, so the collection of articles should give a complete
overview about this topic.

### References

For now, there are no follow up posts to this one.
Once I have written more on this topic, I will link those articles here, to provide an easy way
to navigate to further readings on this topic.
If you have interesting links to share, please send them to me, I will think of a way to include them.

---

PS: this article got a bit longer than expected, I promise, the next posts will be shorter ;D
