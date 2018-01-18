---
layout: post
title: Grunt Task Race Conditions
post-img: "images/snails.jpg"
excerpt_separator: <!--more-->
---

!['Race Condition']({{ site.baseurl }}/images/snails.jpg){:class="text-wrap-left"} Normally, task control flow in Grunt is a simple process of defining the order of task processes in one, or more, default tasks.  This is especially true when your tasks require static data in their configurations, e.g. a static list of paths used for the `src` attribute of an uglify task.  <!--more-->However, we can introduce problems into task flow control when we begin adding dynamic task configuration property data. Grunt will run all methods and populate task properties early, during the load task stage, so if you write a method that is to be called at a specific time, you really don't have control over the timing of that method call.  With static data this is not a problem because the data exists during the task load. Conversely, with dynamic data, this becomes a problem due to data either being stale or non-existent during the load phase.

#### Custom Tasks With Asynchronous Configuration
In order to address the issue of race conditions encountered with dynamic task properties, we need to introduce custom tasks that implement asynchronous task configuration.  By doing so we can gain control over configuration timing an ensure our task properties have the expected data at the time it is needed.  This is best illustrated by example:

##### The Race Condition

```javascript
(function () {

    'use strict';
    module.exports = function(grunt) {
    
        // load all grunt tasks
        require('load-grunt-tasks')(grunt);
        var scriptPathPattern = 'src/main/webapp/project/components/**/*.js';

        // Get the dynamically generated file paths and return an array of paths
        function createFilepathArray() {
            var buffer = grunt.file.read('src/main/webapp/project/imports.txt');
            var pathString = buffer.toString();
            var filePathArray = pathString.split('\n');

            return filePathArray;
        }

        // Build configuration
        grunt.initConfig({
            uglify: {
                options: {
                    banner: '/*! <% pkg.name %> <%= grunt.template.today("yyyy-mm-dd") %> */\n',
                    mangle: false
                },
                my_target: {
                    files: {
                        'dist/project.min.js': createFilepathArray()
                    }
                }
            },
            angularFileLoader: {
                options: {
                    scripts: scriptPathPattern,
                    relative: false
                },
                my_target: {
                    src: ['src/main/webapp/project/imports.txt'];
                }
            }
        });

        grunt.loadNpmTasks('grunt-contrib-uglify');
        grunt.loadNpmTasks('grunt-angular-file-sort');

        grunt.registerTask('default', ['aungularFileLoader', 'uglify']);
    };
})();
```
The problem with the code above is that `createFilepathArray()` will be called prior to `'src/main/webapp/project/imports.txt'` being updated by the `angularFileLoader` task.  This is due to the method being called at load time and then not re-run during the task. The filePathArray has already returned and set the value on `uglify.my_target.files`.  It does not matter if you move the `createFilepathArray()` method outside of the self-invoking function or outside to an external file, it will be called at load time.

##### Go Async Yourself
In order to gain control over this race condition, we can implement asynchronous code into a custom task.  We will do all of our configuration of uglify, along with the logic contained in `createFilepathArray()` within this custom task.  Here is the basic structure, from the Grunt documentation, that we will implement:

```javascript
// Tell Grunt this task is asynchronous.
var done = this.async();
// Your async code.
setTimeout(function() {
  // Let's simulate an error, sometimes.
  var success = Math.random() > 0.5;
  // All done!
  done(success);
}, 1000);
```
Reference: <a href="https://gruntjs.com/api/inside-tasks#inside-all-tasks">GruntJS Inside All Tasks</a>

The above `uglify` task will now be implemented via a custom task (`minifyJS`) with its own configuration:

```javascript
(function () {

    'use strict';
    module.exports = function(grunt) {
    
        // load all grunt tasks
        require('load-grunt-tasks')(grunt);
        var scriptPathPattern = 'src/main/webapp/project/components/**/*.js';

        // Build configuration
        grunt.initConfig({
            uglify: {
                // intentionally left empty - see minifyJS task
            },
            angularFileLoader: {
                options: {
                    scripts: scriptPathPattern,
                    relative: false
                },
                my_target: {
                    src: ['src/main/webapp/project/imports.txt'];
                }
            }
        });

        grunt.loadNpmTasks('grunt-contrib-uglify');
        grunt.loadNpmTasks('grunt-angular-file-sort');
        
        // custom async task to manage timing of filepath array and configuration of uglify
        grunt.task.registerTask('minifyJS', function() {
            // async process
            var done = this.async();
            
            var buffer = grunt.file.read('src/main/webapp/project/imports.txt');
            var pathString = buffer.toString();
            var filePathArray = pathString.split('\n');
            
            setTimeout(function() {
                var config = {};
                config.options = {
                    banner: '/*! <% pkg.name %> <%= grunt.template.today("yyyy-mm-dd") %> */\n',
                    mangle: false
                };
                config.my_target = {};
                config.my_target.files = {
                    'dist/project.min.js': filePathArray
                };
                
                grunt.config('uglify', config);
                grunt.task.run(['uglify']);
              
                done();
            }, 1000);
        })

        grunt.registerTask('default', ['aungularFileLoader', 'minifyJS']);
    };
})();
```
In this version, we've removed the `createFilepathArray()` method altogether.  The uglify task configuration was gutted and its config moved into a new custom task named `minifyJS`.  Within `minifyJS` we constructed an async block that will run and create the updated filePathArray which will be used in assigning the source array to the destination that uglify will write to.  When the block of code outside the `setTimout()` method has completed and the source file data has been generated, the code within `setTimout()` will then run, in this case performing the `uglify` config setup and running the `uglify` task.  It then calls the `done()` function to allow the flow to continue on as usual.  Also note, we've replaced `'uglify'` with `'minifyJS'` in the `default` task.

With this implementation we can successfully introduce dynamic data into grunt task configuration in a controlled task process flow and defeat race conditions that otherwise break our builds.  This technique can be used for any task requiring dynamic build-time data which is setting task configuration properties.