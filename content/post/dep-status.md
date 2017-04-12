+++
date = "2017-04-11T20:22:29-04:00"
draft = true
title = "dep status, April 10"

+++

Hi! I've decided to try to do weekly updates of what's going on with the ongoing development of `dep`, the prototype-to-what'll-become-official dependency management tool for Go. Every week, I'll summarize the changes we've gotten in, where we currently need the most focus, and anything else that's noteworthy.

My hope is that folks will find it easier to keep up and/or jump in on `dep` development by following these posts.

### What's changed

* We started splitting gps up into [multiple packages](https://github.com/sdboyer/gps/pull/189).
* I finally finished [a large refactor of gps](https://github.com/sdboyer/gps/pull/196), eliminating some race conditions that were responsible for a number of bugs in dep.


### What's coming

* **SUPER IMPORTANT** - we're about to change the file format we use for the manifest and lock from JSON to TOML - [issue](https://github.com/golang/dep/issues/119), [PR](https://github.com/golang/dep/pull/342). You'll need to start from scratch. I hope everyone was listening when we said, "this is experimental!"
* We're nearing the finish on support for [symlinking project roots](https://github.com/golang/dep/pull/247). This is *very* basic, and only for the project root, but it should help with some common ways people set up their environments.
* We're working on moving gps directly into dep. This is basically just [pending relicensing](https://github.com/golang/dep/issues/300).
