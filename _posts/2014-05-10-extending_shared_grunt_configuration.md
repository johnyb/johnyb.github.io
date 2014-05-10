---
layout: post
title: Extensible Shared Grunt Configuration
tags: [grunt, ox]
---
In my last [post](/2014/04/30/migrating_a_buildsystem.html), I wrote about a new build system we
implemented for Open-Xchange Appsuite UI modules using grunt.
We provide a [shared grunt configuration](https://github.com/Open-Xchange-Frontend/shared-grunt-config)
for all UI modules in the Appsuite ecosystem.
This post should provide some insight on how this shared configuration can be extended to solve problems
specific to one project.
This might also help a team of developers to work around possible configuration issues, that have not
yet been fixed in the shared configuration.

### Using External Libraries

One common case, that requires a project to extend the shared configuration, are third party libraries.
Those should be separated from the rest of the source code.
In our case, we don't even ship them in our repository, but are using [bower](http://bower.io/)  and
[npm](https://www.npmjs.org/) to manage external dependencies.
A dedicated grunt task is used to pick up all needed files and copy them together with the sources into the
`build/` directory.
This way, we can maintain our individual coding style, enforced by tools like [jshint](http://jshint.com/)
for our code in the `apps/` directory.

In a module that should make use of the [epoxy.js](http://epoxyjs.org/) library, the code can be put into
a `lib/` directory or [bower](http://bower.io/) can be used to provide the sources for the library.
To add this dependency to the `bower.json` file, in the root of the module directory we run:

    bower install backbone.epoxy -S

This will install all needed files in the `bower_components/` directory, which has been added to `.gitignore` by
default.
In order to have it copied into the `build/` directory, a configuration file in `grunt/tasks/` is needed to provide
custom configuration.
As an example, the file `grunt/tasks/epoxy.js` might have this content:

~~~JavaScript
'use strict';
module.exports = function (grunt) {
    grunt.config.extend('copy', {
        epoxy: {
            files: [{
                expand: true,
                src: ['backbone.epoxy.js'],
                cwd: 'bower_components/backbone.epoxy/',
                dest: 'build/apps/3rd.party/epoxy/'
            }]
        }
    });
};
~~~

We currently don't have any nice way to directly extend the `copy_build` task, which is used to copy
everything needed to the `build/` directory of the module.
The `grunt --help` command can be used to obtain a complete list of all tasks `copy_build` is an alias for.
In the case of the Core UI, this looks like:

    jb@wiggum ~/code/appsuite/web/ui (git)-[release-7.6.0] % grunt --help
    Grunt: The JavaScript Task Runner (v0.4.4)
    [… irrelevant output removed]
    Available tasks
    [… irrelevant output removed]
                  copy_build  Alias for "newer:copy:static", "newer:copy:apps",
                                        "newer:copy:dateData", "newer:copy:themes",
                                        "newer:copy:tinymce", "newer:copy:thirdparty",
                                        "newer:copy:specs" tasks.
    [… irrelevant output removed]

Now we override the complete `copy_build` task and add the new task to the bottom:

~~~JavaScript
grunt.registerTask('copy_build', [
    'newer:copy:static',
    'newer:copy:apps',
    'newer:copy:dateData',
    'newer:copy:themes',
    'newer:copy:tinymce',
    'newer:copy:thirdparty',
    'newer:copy:specs',
    'newer:copy:epoxy'
]);
~~~

In the future (v0.3.0 of the [shared-grunt-config](https://github.com/Open-Xchange-Frontend/shared-grunt-config/releases))
there will be a better way to make it even more easy to ship own third party libraries.
For the copy part of the `dist` task, we have found a more elegant way to write this down,
but since `copy_build` is much older, it is not implemented, yet.
The solution will be explained [later](#solution), so at least you get an idea about how it will work.

In order to integrate this with Appsuite, some glue code in the `apps/` directory is needed.
Since [epoxy](http://epoxyjs.org/) needs `underscore` and `Backbone` to be defined as `require`
modules and this is not the case within Appsuite, this needs to be done by hand.
We do this by writing a module:
`apps/3rd.party/epoxy/main.js`

~~~JavaScript
define('underscore', function () {
    //return global _ object
    return _;
});
define('backbone', function () {
    //return global Backbone
    return Backbone;
});

define('3rd.party/epoxy/main', ['3rd.party/epoxy/backbone.epoxy'], function (epoxy) {
    'use strict';

    return epoxy;
});
~~~

This does nothing special but providing the modules in the Appsuite ecosystem.
It's now possible to add `'3rd.party/epoxy/main'` to the dependency list of any other module within your plugin and use it.

### Watching more files

Another change you might want to do is to extend the files watched by the grunt `watch` task.
For example, we didn't add the po files to the list of watched files by default.
This is, because watching files is “expensive”, so we wanted to limit the amount of watched files.
If you decide, this would help your project, those files can be watched by creating the file
`grunt/tasks/watch_po.js` and adding:

~~~JavaScript
'use strict';
module.exports = function (grunt) {
    grunt.config.extend('watch', {
        files: ['i18n/*.po'],
        tasks: ['compile_po', 'send_livereload']
    });
};
~~~

This will watch all po files in `i18n/` and run `compile_po` to create the translation modules and after that
send a `livereload` command to all browsers connected to your connect middleware (if running).
It's as easy as this.

### Solution

The technical solution for this is pretty easy.
We have written a small helper method that allows the global grunt config object to be extended.
The code is straight forward:

~~~JavaScript
grunt.config.extend = function (key, value) {
    grunt.config(key, require('underscore').extend({}, grunt.config(key), value));
};
~~~

It relies on the `_.extend` method, but could of course have been written in plain JavaScript.
Since we already have `underscore` installed, we made use of it.

While writing this post, I remembered, we added another way to extend certain tasks more easily.
This would have been really helpful for the [copy_build](#using-external-libraries) task, but
it is not yet implemented like this. The idea is to prefix all subtasks that belong together and
have a simple way to run all subtasks with a certain prefix.

~~~JavaScript
grunt.util.runPrefixedSubtasksFor = function (main_task, prefix) {
    return function () {
        var list = [];

        for (var key in grunt.config(main_task)) {
            if (key.substr(0, prefix.length) === prefix) {
                list.push(key);
            }
        }
        list = list.map(function (name) {
            return main_task + ':' + name;
        });

        grunt.task.run(list);
    };
};
~~~

This can now be used with the `dist` prefix:

~~~JavaScript
grunt.registerTask('copy_dist', grunt.util.runPrefixedSubtasksFor('copy', 'dist'));
~~~

So in order to have a copy subtask run during `copy_dist`, a configuration with the `dist` prefix will do:

~~~JavaScript
grunt.config.extend('copy', {
    dist_custom: {
        files: [{
            expand: true,
            src: ['share/**/*'],
            cwd: 'build/',
            dest: 'dist/appsuite/'
        }]
    }
});
~~~

As with version 0.3.0 of the [shared-grunt-config](https://github.com/Open-Xchange-Frontend/shared-grunt-config/releases)
something similar will be available for the `copy_build` task.

### Conclusion

As you have learned, all Appsuite related code is located in the `apps/` directory and checked using
[jshint](http://jshint.com/).
An external library can be installed in a separated directory and copied into the `build/` directory during build time.
It can have a complete different coding style and won't interfere with your module.
Some glue code can make it work in the Appsuite environment, so your module can make use of it.

It's also easily possible to extend the files watched by the `watch` task and provide custom tasks that are run if one
of the files changes.
So with a little knowledge of the [grunt](http://gruntjs.com/) system, you can do everything you like.

The shared configuration has still some room for improvements, like a more easy way to extend the `copy_build` task
that holds information on which copy tasks to run during `build`.
This has already been solved and implemented for the `copy_dist` task and will be implemented for `copy_build`, too.
Also our `requirejs` configuration can be improved to support usage of third party libraries.
We provide some of our basic libraries through global objects, like `underscore` and `Backbone`.
However, configuring `require` to serve those objects as shims might be a good idea for the future.
Despite those short comings, extending the shared configuration is possible and quite easy.
Since we are aware of these minor problems, new features to support such use-cases will be added to the shared
config, soon.

### References

Additionally to this post, you might find some parts of the grunt user documentation helpful:

* [Grunt Getting Started](http://gruntjs.com/getting-started)
* [How files are handled in grunt](http://gruntjs.com/configuring-tasks#files)
