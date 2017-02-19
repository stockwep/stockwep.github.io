---
layout: post
title:  "Handling SPA configuration"
date:   2017-02-18 00:00:00 +0000
categories: ALM
tags: 
    - single page application
    - continuous integration
    - vsts
    - webpack 
---

I do a lot of work with customers building single page applications with
Angular, React etc. and a topic which has kept coming up over the last couple of
weeks is how to handle configuration.

Now when I say configuration I'm specifically talking about configuration
required by the application running in the browser, things like where the API
endpoint is. The kind of things that will vary from one environment to the next.
We could simply rerun webpack - you are using a bundler right? Have a build
process? Continuous integration? - maybe we'll revisit those another time. That
solution is however not ideal. If we were to rebuild the application at each
stage of the release pipeline then we would no longer be flowing the same artifacts
through the whole release pipeline, therefore impacting traceability.

So assuming we aren't going to rebuild our application for each environment 
where should we put our configuration?

There are several options, the three I'm going focus on are:

- Script module
- JSON
- Custom data attributes (data-*)

## Script Module

Modern web applications are structured as small modules, it's therefore
reasonable to define configuration as a module which can then be imported.

One small issue - we've already bundled the application and don't want to
recompile. We could lazy load the module using [`require.ensure`
(webpack)](https://webpack.js.org/guides/code-splitting-require/):

```js
require.ensure(['./config'], function(require) {
    var config = require('./config');
})
```

Or if you're using [webpack
2](https://webpack.js.org/guides/code-splitting-import/) you can use the
ECMAScript proposed `import()` instead of the webpack specific `require.ensure`:

```js
// ECMAScript Proposal, also using new async/await
const config = await import('./config');
```

This approach provides a lot of flexibility but does introduce an additional
request, which depending on the application might delay use by the user - do you
need that configuration in order to initialize the application?

## JSON

If you think configuration, you might well think of JSON - well in the web
context anyway. This seems a logical choice at first.

A typical pattern used for handling configuration is to include a base or
default configuration in the deployment package, and then apply environment specific
overrides at deployment time.

It would be easy to maintain a JSON override file for each environment and merge
as part of the deployment, there are however some disadvantages with this
approach - namely how we deliver the configuration to the browser. In order to
load the config we have to make a request using the trusty `XMLHttpRequest`
(or if we're targeting evergreen browsers or polyfilling we can use
the newer fetch API). That request will be made from JavaScript so we will have to
either load an external script or inline it into the HTML page.

So overall JSON is a good way to handle the actual configuration and overrides
between environments but the act of loading it suffers the same downsides as the
script module approach.

## Custom Data Attributes

If the configuration is limited to only a few simple values then using the HTML5
custom data attributes (more commonly known as data-* attributes) is a good
option. Typically a DOM node is used as a mount point for the application, this
provides the ideal place to stash the configuration.

```html
<div id="root" data-api-root="https://api.example.com"></div>
```

## Application Bootstrapper

Before we get to the final thought I think it's worth thinking about the parts
that make up a single page application. They're composed of one or more
JavaScript bundles, a collection of static assets (images, CSS etc.) and a
single HTML page. 

Our humble `index.html` is not a web page but a bootstrapper. It's purpose to
download and start the application. Obviously this is not true if we are
embedding the application into existing pages - there are different
considerations in that scenario.

Thinking about the web page in this way leads us to see it as a good place to put
the configuration.

## Final Thought

For me, I want to enable fast application load and therefore avoid any
unnecessary HTTP requests so the custom data attributes would be my preferred
route. This approach falls down when we have more than a few config
values or complex types but for most simple applications how much config is
truly needed to put the application into a working state? I would rather see the
critical API endpoints specified and those used to load any additional config.

For those cases where more complex configuration really is needed I would use
JSON inlined into `index.html` (bootstrapper). This is the pattern taken for
setting the initial state in Redux when [server
rendering](http://redux.js.org/docs/recipes/ServerRendering.html). This approach
removes any additional HTTP requests but allows a more suitable format to be
used.

If your application is using Redux and server rendering then the configuration
should be part of your application state.

```html
<!doctype html>
<html>
    <head>
        <title>SPA Config Example</title>
    </head>
    <body>
        <div id="root"></div>
        <script>
            window.__CONFIG__ = {
                "apiRoot": "https://api.example.com"
            };
        </script>
        <script src="/bundle.js"></script>
    </body>
</html>
```

While not explicitly highlighted we've looked at this from a perspective of
static pages, however the approaches discussed apply equally to dynamically
generated pages. The real difference is in those cases where the configuration
is inlined. For static pages the transformation would be applied as part of the
deployment. In the case of dynamic pages the server application would handle
writing the configuration to the page.