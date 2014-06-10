---
layout: post
title: Appserver - A proxy middleware for front-end development
tags: [ox, tools, appserver]
---
Multiple times, I mentioned *appserver*, a tool developed and used by the front-end team at Open-Xchange.
Earlier this year, we decided to publish it to the npm registry as a node module, so other developers can
easily install it.
We also put it on [github](https://github.com/Open-Xchange-Frontend/appserver), to eventually allow more
easy contributions.
The functionality is tightly coupled to Open-Xchange Appsuite UI development, but it might provide a good
starting point for some similar project.
This post should give you some insights about what this tool can do and how it works.
Even if you are familiar with *appserver*, this might still be worth reading.

### Serving files locally

The most common use-case is probably that of serving all files belonging to an UI module.
During development, our [shared grunt configuration](https://github.com/Open-Xchange-Frontend/shared-grunt-config)
can be used to run certain tasks transforming all source files into a shippable form.
The first step is to organize all JavaScript, images, CSS, translations and other data within the `build/` directory.
This directory contains the basic structure of the servable module.
The *appserver* can pick up everything in this directory and make it available to the browser.

When developing for Appsuite, there is a special way to load code or static text data.
There is a servlet (`apps/load`), that can be used to dynamically concatenate files during runtime.
On the client side (requirejs in the browser) all requests that occur during a specific time frame are collected
and only one request is sent to the `apps/load` servlet.
This large response is then separated into individual files again and handed over to requirejs, to be loaded in the
browser.
Only one HTTP request is needed, instead of one for each file being loaded.
*Appserver* also implements the `apps/load` interface but serves files from the local directories, if found.
Resources that are not loaded via requirejs are requested directly by the browser and are served as well.

### Proxy calls to some remote server

Of course we don't depend on a local installation of the complete front-end code in order to develop locally.
It is possible to configure the *appserver* with a remote server and all requests, that can not be handled
locally, will be forwarded to this server.
The response is sent back to the browser.
This works for the local implementation of `apps/load`, loading of files, and every other API call to the backend.

#### Manipulating server responses

In some cases it is necessary to change the response returned by the remote server before sending it back
to the browser.
For example, a timestamp appended to the version string is used internally to test if the caches for requirejs
need to be invalidated.
The timestamp can be added during build time, but the file containing the version string is only built
by the core UI.
For external modules, this file is loaded through the proxy and has to be changed by *appserver* on-the-fly.

Another case would be manifest files, that would need to be installed on the server, but are shipped by the
UI modules them self.
In *appserver*, there is a way to inject those local manifest files into the server configuration.
All changes to local manifest files will be available in your browser after the next reload.

### Workflow using grunt

Our [shared grunt configuration](https://github.com/Open-Xchange-Frontend/shared-grunt-config) contains a
pre-configured version of [grunt-contrib-connect](https://github.com/gruntjs/grunt-contrib-connect).
This gets loaded when running `grunt connect watch` or `grunt dev`.
The task is named `connect`, you will find it in grunt's output.
By default, [grunt-contrib-connect](https://github.com/gruntjs/grunt-contrib-connect) has support for
[connect-livereload](https://github.com/intesso/connect-livereload), a middleware to add the livereload script to the
response.
In combination with [grunt-contrib-watch](https://github.com/gruntjs/grunt-contrib-watch#optionslivereload), which is
able to start a *livereload* server, automatic reloading of all connected browsers is supported.
The `connect` task is configured to load all the *appserver* middleware modules, so everything mentioned above can be
used in this setup, too.

All this enables you to do the following for an Appsuite UI module:

1. run `grunt dev` to start
    * `connect` server
    * [karma](https://karma-runner.github.io/) based test server
    * the `watch` task
2. connect to `http://localhost:8337/appsuite/` in a browser of your choice
    * if not already done, login
3. open any file from your `apps/` directory in your favorite editor
4. make a change and hit the `save` button

Now the change is detected by the `watch` task and the `build` task is run to generate a recent version in
the `build/` directory.
After `build` is complete, the `send_livereload` task is run to trigger an update on the *livereload* server.
The *livereload* server will send an update event to the connected browser(s) which will then reload with the
latest version.
Last but not least, the `testrun` task will tell the karma runner to run all the tests defined within a
`spec/` directory.

So what we have is a tight feedback loop for the code you are writing.
A few moments after hitting the `save` button in your editor, your browser window will have an up-to-date version of
your code running.
If you have written tests (you really should! ;)), you will see the result in your terminal window shortly after that.
This really reduces development to: changing code and inspecting the result.
No manual tasks in between.

### Conclusion

All in all, *appserver* is a MITM (man-in-the-middle) proxy server with some abilities to manipulate requests
and responses to/from a _real_ Open-Xchange backend.
It's implemented using a few [connect](http://www.senchalabs.org/connect/) based middleware modules, that provide
support for serving local files, remote files, remote API calls, and the injection of local manifest files.
In combination with the [shared grunt configuration](https://github.com/Open-Xchange-Frontend/shared-grunt-config),
it's possible to work in a tight feedback loop: change code and inspect the result.

The dynamic injection of the latest build timestamp to core files, has landed recently, which results in a large
improvement for non-core developers.
No more work-arounds like the `touch-appsuite` script or `&cacheBusting=true` URL parameter are needed any longer.

Of course, there is still room for improvement.
For now, it's still a little complicated to develop multiple modules at once.
The list of prefixes (think of it as the set of `build/` directories to be served by *appserver*) is hard-coded
(though easily configurable) in one file.
Our goal is to run `grunt dev` in each of the modules you want to serve and be fine.
There should be no more configuration.
This can be achieved by registering new modules with an already running *appserver*.
A running instance of *appserver* can then dynamically add new modules to the list of prefixes and start serving
these, too.

Another helpful feature would be to extend the ability to manipulate responses not only to manifests, but
server configuration in general.
Most prominent use-case would be to arbitrary change `capabilities` that are configured on the server.
This is of course possible at the moment, using the `&cap=[capability,list]` URL parameter, so there is no real need to
implement that.
