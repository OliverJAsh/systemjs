SystemJS
========

Spec-compliant universal module loader - loads ES6 modules, AMD, CommonJS and global scripts.

Designed as a collection of small extensions to the ES6 specification System loader, which can also be applied individually.

* Loads any module format, by detecting the format automatically. Modules can also [specify their format with meta syntax](#meta-configuration).
* Provides comprehensive and exact replications of AMD, CommonJS and ES6 circular reference handling.
* Loads [ES6 modules compiled into the `System.register` form for production](#es6-systemregister-compilation), maintaining full circular references support.
* Supports RequireJS-style [map](#map-configuration), [paths](https://github.com/ModuleLoader/es6-module-loader#paths-implementation), [bundles](#bundles), [shim](#global-module-format-support) and [plugins](#plugins).
* Tracks package versions, and resolves semver-compatibile requests through [package version syntax](#versions) - `package@x.y.z`, `package^@x.y.z`.
* [Loader plugins](#plugins) allow loading assets through the module naming system such as CSS, JSON or images.

Designed to work with the [ES6 Module Loader polyfill](https://github.com/ModuleLoader/es6-module-loader) (7KB) for a combined total footprint of 13.3KB minified and gzipped. In future, with native implementations, the ES6 Module Loader polyfill should no longer be necessary. As jQuery provides for the DOM, this library can smooth over inconsistiencies and missing practical functionality provided by the native System loader.

Runs in IE8+ and NodeJS. ES6 modules are only supported in IE9+.

For discussion, [see the Google Group](https://groups.google.com/group/systemjs).

Basic Configuration
---

### Setup

Download [`es6-module-loader.js`](https://github.com/ModuleLoader/es6-module-loader/blob/v0.6.0/dist/es6-module-loader.js) and [`traceur.js`](https://github.com/google/traceur-compiler/blob/0.0.41/bin/traceur.js) and locate them in the same folder as `system.js` from this repo.

We then include `dist/system.js` with a script tag in the page.

`es6-module-loader.js` will then be included from the same folder automatically and [Traceur](https://github.com/google/traceur-compiler) is dynamically included from `traceur.js` when loading an ES6 module only.

### Simple Application Structure

The standard application structure would be something like the following:

index.html:
```html
<script src="system.js"></script>
<script>
  // Identical to writing System.baseURL = ...
  System.config({

    // set all requires to "lib" for library code
    baseURL: '/lib',
    
    // set "app" as an exception for our application code
    paths: {
      'app/*': '/app/*.js'
    }
  });

  System.import('app/app')
  .catch(function() {
    // we add this since promises suppress all errors
    setTimeout(function() {
      throw e;
    });
  });
</script>
```

app/app.js:
```javascript
  // relative require for within the package
  require('./local-dep');    // -> /app/local-dep.js

  // library resource
  var $ = require('jquery'); // -> /lib/jquery.js

  // format detected automatically
  console.log('loaded CommonJS');
```

Module format detection happens in the order System.register, ES6, AMD, then CommonJS and falls back to global modules.

Named defines are also supported, with the return value for a module containing named defines being its last named define.

> _Note that when running locally, ensure you are running from a local server or a browser with local XHR requests enabled. If not you will get an error message._

> _For Chrome on Mac, you can run it with: `/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --allow-file-access-from-files &> /dev/null &`_

> _In Firefox this requires navigating to `about:config`, entering `security.fileuri.strict_origin_policy` in the filter box and toggling the option to false._

### Loading ES6

app/es6-file.js:
```javascript
  export class q {
    constructor() {
      this.es6 = 'yay';
    }
  }
```

```html
  <script>
    System.import('app/es6-file').then(function(m) {
      console.log(new m.q().es6); // yay
    });
  </script>
```

ES6 modules define named exports, provided as getters on a special immutable `Module` object.

To build for production, see the [System.register build workflow](#es6-systemregister-compilation).

For further infomation on ES6 module loading, see the [ES6 Module Loader polyfill documentation](https://github.com/ModuleLoader/es6-module-loader).

Features
---

### Convenience Form

Typically we would have configuration shared between pages, so we can create a shared `config.json` file:

config.json:
```js
{
  "paths": {
    "app/*": "/app/*.js"
  }
}
```

Then with script attributes, the above example can be rewritten as:

```html
  <script src="system.js" baseurl="/lib" config="/" main="app/"></script>
```

This will load `/config.json` before loading the main from `app/app` (`app/` being a shorthand for this). Any loading errors will be caught and thrown.

> Data Attributes like `data-baseurl` can be used instead of the above for valid XHTML.

### Map Configuration

Map configuration alters the module name at the normalization stage. It is useful for creating aliases:

```javascript
  System.map['jquery'] = 'location/for/jquery';

  System.import('jquery')           // -> 'location/for/jquery'
  System.import('jquery/submodule') // -> `location/for/jquery/submodule'
```

Contexual map configuration can also be used to provide maps only for certain modules, which is useful for version mappings:

```javascript  
  System.map['some-module'] = {
    jquery: 'jquery@2.0.3'
  };

  // some-module.js now gets 'jquery@2.0.3'
  // everything else still gets 'location/for/jquery'
```

Contextual maps apply from the most specific module name match only.

### Meta Configuration

The ES6 module loader uses a special `metadata` object that is passed between hooks.

An example of meta config is the module format of a module, which is stored at `metadata.format`.

The meta extension opens up this object for setting defaults through `System.meta` as well as inline module syntax.

In this way, we can specify the module format of a module through config:

```javascript
  System.meta['some/module'] = {
    format: 'amd'
  };

  System.import('some/module') // always loaded as AMD even if it is a UMD module
```

Or the module can even specify its own meta:

some/module.js
```javascript
  "format amd";

  if (typeof module != 'undefined' && module.exports)
    module.exports = 'cjs';
  else
    define(function() { return 'amd' });
```

Since it is impossible to write 100% accurate module detection, this inline `format` hint provides a useful way of informing the module format of a module.

The format options are - `register`, `es6`, `amd`, `cjs`, `global`.

### Global Module Format Support

When a module is loaded as a global, the global object is detected automatically from the change in the `window` properties:

app/sample-global.js
```javascript
  hello = 'world';
```

```javascript
  System.import('app/sample-global').then(function(m) {
    m == 'world';
  });
```

Global dependencies can be specified with the `deps` [meta config](#meta-configuration):

app/another-global.js
```javascript
  $(document).fn();
  this.is = 'a global dependent on jQuery';
```

```javascript
  System.meta['app/another-global'] = { deps: ['jquery'] };
```

The `exports` meta config can also be set (using inline meta as an example):

app/more-global.js
```javascript
  "format global";
  "deps jquery";
  "exports my.export";

  (function(global) {
    global.my = {
      export: 'value'
    };
    $(document).fn();
  })(typeof window != 'undefined' ? window : global);
```

There is also supports for the `init` function meta config just like RequireJS as well.

**IE8 Support**

In IE8, automatic global detection does not work for globals declared as variables or implicitly:

```javascript
  var someGlobal = 'IE8 wont detect this';
  anotherGlobal = 'unless using an explicit shim';
```

### Versions

An optional syntax for version support can be used: `moduleName@version`.

For example, consider an app which wants to specify the jQuery version through config:

```javascript
  System.versions['jquery'] = '2.0.3';
```

Now an import of the form:

```javascript
  System.import('jquery');
```

will load a load will be made to the file `/lib/jquery@2.0.3.js`.

This centralises the version management to the configuration file, which is key to handling versions with correct caching.

For multi-version support, provide an array of versions:

```javascript
  System.versions['jquery'] = ['2.0.3', '1.8.3'];
```

These correspond to `/lib/jquery@2.0.3.js` and `/lib/jquery@1.8.3.js`.

Semver-compatible requires of any of the following forms can be used:

```javascript
  System.import('jquery')        // -> /lib/jquery@2.0.3.js
  System.import('jquery@2')      // -> /lib/jquery@2.0.3.js
  System.import('jquery@2.0')    // -> /lib/jquery@2.0.3.js
  
  System.import('jquery@1')      // -> /lib/jquery@1.8.3.js
  System.import('jquery@1.8')    // -> /lib/jquery@1.8.3.js
  System.import('jquery@1.8.2')  // -> /lib/jquery@1.8.2.js
  
  // semver compatible form (caret operator ^)
  System.import('jquery@^2')     // -> /lib/jquery@2.0.3.js
  System.import('jquery@^1.8.2') // -> /lib/jquery@1.8.3.js
  System.import('jquery@^1.8')   // -> /lib/jquery@1.8.3.js
```

### Relative Dynamic Loading

Modules can check their own name from the global variable `__moduleName`.

This allows easy relative dynamic loading, allowing modules to load additional functionality after the initial load:

```javascript
export function moreFunctionality() {
  return System.import('./extrafunctionality', { name: __moduleName });
}
```

This can be useful for modules that may only know during runtime which functionality they need to load.

### Plugins

Plugins handle alternative loading scenarios, including loading assets such as CSS or images, and providing custom transpilation scenarios.

Plugins are indicated by `!` syntax, which unlike RequireJS is appended at the end of the module name, not the beginning.

The plugin name is just a module name itself, and if not specified, is assumed to be the extension name of the module.

Supported Plugins:

* [CSS](https://github.com/systemjs/plugin-css) `System.import('my/file.css!')`
* [Image](https://github.com/systemjs/plugin-image) `System.import('some/image.png!image')`
* [JSON](https://github.com/systemjs/plugin-json) `System.import('some/data.json!').then(function(json){})`
* [Text](https://github.com/systemjs/plugin-text) `System.import('some/text.txt!text').then(function(text) {})`

Additional Plugins:

* [Markdown](https://github.com/guybedford/plugin-md) `System.import('app/some/project/README.md!').then(function(html) {})`
* [WebFont](https://github.com/guybedford/plugin-font) `System.import('google Port Lligat Slab, Droid Sans !font')`

Creating custom plugins can be quite simple. See the plugins above, and [read the guide here](https://github.com/systemjs/systemjs/wiki/Creating-a-Plugin).


### ES6 System.register Compilation

If writing an application in ES6, we can compile into ES5 with Traceur:

```
  npm install traceur -g
```

```
  traceur --dir app app-built --modules=instantiate
```

This will compile all ES6 files in the directory `app` into corresponding AMD files in `app-built`.

The `instantiate` modules option writes the modules out using a `System.register` call, which is supported by SystemJS.

Then include `traceur-runtime.js` (this is found inside traceur's `bin` folder when installed via npm) before es6-module-loader.js:

```html
  <script src="traceur-runtime.js"></script>
  <script src="system.js"></script>
```

Now the application will continue to behave identically without needing to compile ES6 in the browser.

### Compiling ES6 to ES5 and AMD

The same method above can also be used to compile ES6 into AMD with `--modules=amd`.

We can then use the r.js optimizer to create a bundle with named defines, which are supported by SystemJS.

Note that the ES6 live bindings and circular references don't work in AMD, although circular references still work in many cases.

### Bundles

We can create a custom bundle with Traceur by comining together a module with all its dependencies into a single file:

```
  traceur --out build.js app/main.js app/core.js app/another.js
```

Each file will be traced and all its dependencies included in the final build file.

We can then include this bundle with a `<script>` tag in the page, or we can use the `bundles` config to dynamically load it when a module
is dynamically requested from that bundle.

### CSP-Compatible Production

SystemJS comes with a separate build for production only. This is fully CSP-compatible using script tag injection to load scripts, while still remaining an
extension of the ES6 Module Loader.

Replace the `system.js` file with `dist/system-production-csp.js`.

If we have compiled all our modules into a `System.register` bundle, we can do:

```html
  <script src="system-production-csp.js"></script>
  <script>
    System.paths['app-built'] = '/app-built.js';
    System.bundles['app-built'] = ['app/main'];
    System.import('app/main').then(function(m) { 
      // loads app/main from the app-built bundle
    });
  </script>
```

_Note that this CSP-compatibility mode doesn't fully support plugins, CommonJS or global script loading._

### NodeJS Usage

To load modules in NodeJS, install SystemJS with:

```
  npm install systemjs
```

We can then load modules equivalently to in the browser:

```javascript
var System = require('systemjs');

// loads './app.js' from the current directory
System.import('./app').then(function(m) {
  console.log(m);
}, function(e) {
  console.log(e);
});
```

## Contributing

Contributions are welcome. The goal of SystemJS is to encourage loaders made out of small self-contained features.

Since different builds can be created for different use cases, new builds or new features are welcome to be submitted for
consideration with pull requests.

#### Running the tests

To install the dependencies correctly, run `bower install` from the root of the repo, then open `test/test.html` in a browser with a local server
or file access flags enabled.


License
---

MIT

