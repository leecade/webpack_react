# Building Kanban

Now that we have a nice Kanban application up and running, we can worry about showing it to the public. The goal of this chapter is to set up a nice production grade build. There are various techniques we can apply to bring the bundle size down. We can also leverage browser caching.

## Setting Up a Build Target

In our current setup, we always serve the application through `webpack-dev-server`. To create a build, we'll need to extend the `scripts` section in *package.json*.

**package.json**

```json
{
  ...
  "scripts": {
    "build": "webpack",
    ...
  },
  ...
}
```

We'll also need some build specific configuration to make Webpack pick up our JSX. We can set up sourcemaps while we're at it. I'll be using the `source-map` option here as that's a good choice for production.

**webpack.config.js**

```javascript
...

if(TARGET === 'build') {
  module.exports = merge(common, {
    devtool: 'source-map'
  });
}
```

After these changes, `npm run build` should yield something like the following:

```bash
> webpack

Hash: bd3b549c6c712233167f
Version: webpack 1.10.1
Time: 4903ms
        Asset       Size  Chunks             Chunk Names
    bundle.js    1.09 MB       0  [emitted]  main
bundle.js.map    1.28 MB       0  [emitted]  main
   index.html  184 bytes          [emitted]
    + 345 hidden modules
```

1.09 MB is quite a lot. We should do something about that.

## Optimizing Build Size

There are a couple of basic things we can do to slim down our build. We can apply some minification to it. We can also tell React to optimize itself. Doing both will result in significant size savings. Provided we apply gzip compression on the content when serving it, further gains may be made.

### Minification

Minification will convert our code into a smaller format without losing any meaning. Usually this means some amount of rewriting code through predefined transformations. Sometimes this can break code as it can rewrite pieces of code you inadvertently depend upon. This is the reason why we gave explicit ids to our stores for instance.

The easiest way to enable minification is to call `webpack -p` (`-p` as in `production`). As Uglify will output a lot of warnings and they don't provide value in this case, we'll be disabling them. Add the following section to your Webpack configuration:

**webpack.config.js**

```javascript
if(TARGET === 'build') {
  module.exports = merge(common, {
    devtool: 'source-map',
    plugins: [
      new webpack.optimize.UglifyJsPlugin({
        compress: {
          warnings: false
        }
      })
    ]
  });
}
```

T> Uglify warnings can help you to understand how it processes the code. Therefore it may be beneficial to have a peek at the output every once in a while.

If you hit `npm run build` now, you should see better results:

```bash
> webpack

Hash: d3508663532b5b3565cc
Version: webpack 1.10.1
Time: 12221ms
        Asset       Size  Chunks             Chunk Names
    bundle.js     324 kB       0  [emitted]  main
bundle.js.map    2.66 MB       0  [emitted]  main
   index.html  184 bytes          [emitted]
    + 345 hidden modules
```

Given it needs to do more work, it took longer. But on the plus side the build is much smaller now.

T> It is possible to push minification further by enabling variable name mangling. It comes with some extra complexity to worry about, but it may be worth it when you are pushing for minimal size. See [the official documentation](https://webpack.github.io/docs/list-of-plugins.html#uglifyjsplugin) for details.

### `process.env.NODE_ENV`

We can perform one more step to decrease build size further. React relies on `process.env.NODE_ENV` based optimizations. If we force it to `production`, React will get built in an optimized manner. This will disable some checks (e.g., property type checks). Most importantly it will give you a smaller build and improved performance.

In Webpack terms, you can add the following snippet to the `plugins` section of your configuration:

**webpack.config.js**

```javascript
if(TARGET === 'build') {
  module.exports = merge(common, {
    devtool: 'source-map',
    plugins: [
      new webpack.DefinePlugin({
        'process.env': {
          // This affects react lib size
          'NODE_ENV': JSON.stringify('production')
        }
      }),
      ...
    ]
  });
}
```

This is a useful technique for your own code. If you have a section of code that evaluates as `false` after this process, the minifier will remove it from build completely. You can attach debugging specific utilities and such to your code easily this way. For instance, you could build a powerful logging system just for development. Here's a small example of what that could look like:

```javascript
if(process.env.NODE_ENV !== 'production') {
  console.log('developing like an ace');
}
```

T> That `JSON.stringify` is needed, as Webpack will perform string replace "as is". In this case, we'll want to end up with strings, as that's what various comparisons expect, not just `production`. The latter would just cause an error. An alternative would be to use a string such as `'"production"'`. Note the double quotation marks (").

Hit `npm run build` again, and you should see improved results:

```bash
> TARGET=build webpack

Hash: 37ebe639517bfeb72ff6
Version: webpack 1.10.1
Time: 10930ms
        Asset       Size  Chunks             Chunk Names
    bundle.js     264 kB       0  [emitted]  main
bundle.js.map    2.53 MB       0  [emitted]  main
   index.html  184 bytes          [emitted]
    + 339 hidden modules
```

So we went from 1.09 MB to 324 kB, and finally to 264 kB. The final build is a little faster than the previous one. As that 264k can be served gzipped, it is quite reasonable. gzipping will drop around another 40%. It is well supported by browsers.

We can do a little better, though. We can split `app` and `vendor` bundles and add hashes to their filenames.

## Splitting `app` and `vendor` Bundles

The main advantage of splitting the application into two separate bundles is that it allows us to benefit from client caching. We might, for instance, make most of our changes to the small `app` bundle. In this case, the client would have to fetch only the `app` bundle, assuming the `vendor` bundle has already been loaded.

This scheme won't load as fast as a single bundle initially due to the extra request. Thanks to client-side caching, we might not need to reload all the data for every request. This is particularly true if a bundle remains unchanged. For instance if only `app` updates, only that may need to be downloaded.

### Defining a `vendor` Entry Point

To get started, we need to define a `vendor` entry point:

**webpack.config.js**

```javascript
...

var pkg = require('./package.json');

var TARGET = process.env.npm_lifecycle_event;
var ROOT_PATH = path.resolve(__dirname);
var APP_PATH = path.resolve(ROOT_PATH, 'app');
var BUILD_PATH = path.resolve(ROOT_PATH, 'build');

...

if(TARGET === 'build') {
  module.exports = merge(common, {
    entry: {
      app: APP_PATH,
      vendor: Object.keys(pkg.dependencies)
    },
    output: {
      path: BUILD_PATH,
      filename: 'bundle.js'
    },
    devtool: 'source-map',
    ...
  });
}
```

This tells Webpack that we want a separate *entry chunk* for our project `vendor` level dependencies. Webpack provides multiple ways to define chunks. Each chunk will contain a part of your application code. For example it's possible to set up chunks that are loaded dynamically.

### Adding Hashes to Filenames

To make sure client-side caching works, we'll need to attach hashes to filenames. Webpack provides placeholders that are useful for this. The most useful ones are:

* `[name]` - Returns entry name.
* `[hash]` - Returns build hash.
* `[chunkhash]` - Returns a chunk specific hash.

Using these placeholders you could end up with filenames such as:

```bash
app.d587bbd6e38337f5accd.js
vendor.dc746a5db4ed650296e1.js
```

If the file contents are different, the hash will change as well, thus invalidating the cache, or more accurately the browser will send a new request for the new file.

An alternative way to achieve the same would be to generate static filenames and invalidate the cache through a querystring (i.e., `app.js?d587bbd6e38337f5accd`). The part behind the question mark will invalidate the cache. This method is not recommended. According to [Steve Souders](http://www.stevesouders.com/blog/2008/08/23/revving-filenames-dont-use-querystring/), attaching the hash to the filename is a more performant way to go.

It's just as easy to include the hash in the filename itself, and I find this convention to be cleaner.

We can use the placeholder idea within our configuration like this:

**webpack.config.js**

```javascript
if(TARGET === 'build') {
  module.exports = merge(common, {
    entry: {
      app: APP_PATH,
      vendor: Object.keys(pkg.dependencies)
    },
    /* important! */
    output: {
      path: BUILD_PATH,
      filename: '[name].[chunkhash].js?'
    },
    ...
  });
}
```

If you hit `npm run build` now, you should see output like this.

```bash
> webpack

Hash: 93b7068ed91340e3f5ac
Version: webpack 1.12.0
Time: 16398ms
                             Asset       Size  Chunks             Chunk Names
       app.d9e6f800dcb46e3638b9.js     263 kB       0  [emitted]  app
    vendor.f6e6ad0dd8123b64e914.js     208 kB       1  [emitted]  vendor
   app.d9e6f800dcb46e3638b9.js.map    2.51 MB       0  [emitted]  app
vendor.f6e6ad0dd8123b64e914.js.map    2.13 MB       1  [emitted]  vendor
                        index.html  266 bytes          [emitted]
   [0] multi vendor 76 bytes {1} [built]
    + 316 hidden modules
```

This isn't quite what we expected. Both our `app` and `vendor` bundles are big. The problem is that Webpack will include all the required files in the `app` bundle. This includes the `vendor` modules.

### Setting Up `CommonsChunkPlugin`

To fix our problem, we can use `CommonsChunkPlugin`. It can extract the code we need. We can tell it what *entry chunk* to extract and how to output it. Consider the configuration:

**webpack.config.js**

```javascript
if(TARGET === 'build') {
  module.exports = merge(common, {
    ...
    module: {
      ...
    },
    plugins: [
      /* important! */
      new webpack.optimize.CommonsChunkPlugin(
        'vendor',
        '[name].[chunkhash].js'
      ),
      ...
    ]
  });
}
```

If you run `npm run build` now, you should see output as below:

```bash
> webpack

Hash: 5a0cf7711f1d3fc5c930
Version: webpack 1.12.0
Time: 10565ms
                             Asset       Size  Chunks             Chunk Names
       app.bf777bbb05bec4070276.js      55 kB       0  [emitted]  app
    vendor.f33176572a7ad31551ee.js     209 kB       1  [emitted]  vendor
   app.bf777bbb05bec4070276.js.map     389 kB       0  [emitted]  app
vendor.f33176572a7ad31551ee.js.map    2.13 MB       1  [emitted]  vendor
                        index.html  266 bytes          [emitted]
   [0] multi vendor 76 bytes {1} [built]
    + 316 hidden modules
```

Note how small `app` bundle is in comparison. If we update the application now and deploy it, the users that have used it before will have to reload only 55 kB.

One more way to push the build further would be to load popular dependencies, such as React, through a CDN. That would decrease the size of the vendor bundle even further while adding an external dependency on the project. The idea is that if the user has hit the CDN earlier, caching can kick in just like here.

## Cleaning the Build

Our current setup doesn't clean the `build` directory between builds. As this can get annoying if we change our setup, we can use a plugin to clean the directory for us. Execute

```bash
npm i clean-webpack-plugin --save-dev
```

to install the plugin. Change the build configuration as follows to integrate it:

**webpack.config.js**

```javascript
...
var Clean = require('clean-webpack-plugin');

...

if(TARGET === 'build') {
  module.exports = merge(common, {
    ...
    plugins: [
      new Clean(['build']),
      ...
    ]
  });
}
```

After this change our `build` directory should remain nice and tidy when building. See [clean-webpack-plugin](https://www.npmjs.com/package/clean-webpack-plugin) for further options.

T> An alternative would be to use your terminal (`rm -rf ./build/`) and set that up in the `scripts` section of *package.json*.

## Separating CSS

Even though we have a nice build set up now, where did all the CSS go? As per our configuration it has been inlined to JavaScript! Even though this can be convenient during development, it doesn't sound ideal. The current solution doesn't allow us to cache CSS. In some cases we might suffer from a flash of unstyled content (FOUC).

It just so happens that Webpack provides a means to generate a separate CSS bundle. We can achieve this using the `ExtractTextPlugin`. It comes with overhead during the compilation phase, and it won't work with Hot Module Replacement (HMR) by design. Given we are using it only for production, that won't be a problem.

It will take some configuration to make it work. Hit

```bash
npm i extract-text-webpack-plugin --save-dev
```

to get started. Next we need to get rid of our current CSS related declaration at `common` configuration. After that, we need to split it up between `build` and `dev` configuration sections as follows:

**webpack.config.js**

```javascript
...
var ExtractTextPlugin = require('extract-text-webpack-plugin');

var pkg = require('./package.json');

var TARGET = process.env.npm_lifecycle_event;
var ROOT_PATH = path.resolve(__dirname);
var APP_PATH = path.resolve(ROOT_PATH, 'app');
var BUILD_PATH = path.resolve(ROOT_PATH, 'build');

var common = {
  entry: APP_PATH,
  resolve: {
    extensions: ['', '.js', '.jsx']
  },
  module: {
    loaders: [
      {
        test: /\.jsx?$/,
        loaders: ['babel'],
        include: APP_PATH
      }
    ]
  },
  plugins: [
    new HtmlwebpackPlugin({
      title: 'Kanban app'
    })
  ]
};

if(TARGET === 'start' || !TARGET) {
  module.exports = merge(common, {
    ...
    module: {
      loaders: [
        {
          test: /\.css$/,
          loaders: ['style', 'css'],
          include: APP_PATH
        }
      ]
    },
    ...
  });
}

if(TARGET === 'build') {
  module.exports = merge(common, {
    ...
    devtool: 'source-map',
    module: {
      loaders: [
        {
          test: /\.css$/,
          loader: ExtractTextPlugin.extract('style', 'css'),
          include: APP_PATH
        }
      ]
    },
    plugins: [
      new Clean(['build']),
      new ExtractTextPlugin('styles.[chunkhash].css'),
      ...
    ]
  });
}
```

Using this setup we can still benefit from the HMR during development. For a production build, we generate a separate CSS. `html-webpack-plugin` will pick it up automatically and inject it into our `index.html`.

W> Definitions such as `loaders: [ExtractTextPlugin.extract('style', 'css')]` won't work and will cause the build to error instead! So when using `ExtractTextPlugin`, use the `loader` form instead.

W> If you want to pass more loaders to the `ExtractTextPlugin`, you should use `!` syntax. Example: `ExtractTextPlugin.extract('style', 'css!autoprefixer-loader')`.

After running `npm run build` you should see output similar to the following:

```bash
> webpack

Hash: 27584124a5659a941eea
Version: webpack 1.12.0
Time: 10589ms
                              Asset       Size  Chunks             Chunk Names
        app.4a3890cdb2f12f6bd4d5.js    54.3 kB       0  [emitted]  app
     vendor.876083b45225c03d8a74.js     208 kB       1  [emitted]  vendor
    styles.bf777bbb05bec4070276.css  557 bytes       0  [emitted]  app
    app.4a3890cdb2f12f6bd4d5.js.map     389 kB       0  [emitted]  app
styles.bf777bbb05bec4070276.css.map   87 bytes       0  [emitted]  app
 vendor.876083b45225c03d8a74.js.map    2.12 MB       1  [emitted]  vendor
                         index.html  317 bytes          [emitted]
   [0] multi vendor 64 bytes {1} [built]
    + 316 hidden modules
Child extract-text-webpack-plugin:
        + 2 hidden modules
```

This means we have separate app and vendor bundles. In addition, styles have been pushed to a separate file. And on top of this, we have sourcemaps and an automatically generated *index.html*.

T> If you have a complex project with a lot of dependencies, it is likely a good idea to use the `DedupePlugin`. It will find possible duplicate files and deduplicate them. Use `new webpack.optimize.DedupePlugin()` in your plugins definition to enable it.

W> Note that there's [a bug](https://github.com/webpack/webpack/issues/1315) in Webpack preventing this feature from working correctly at the moment! That is, if you change your application code, the hash for `vendor` will change!

## Analyzing Build Statistics

Analyzing build statistics is a good step towards understanding Webpack better. We can get statistics from it easily and we can visualize them using a tool. This shows us the composition of our bundles.

In order to get suitable output we'll need to do a couple of tweaks to our configuration:

**package.js**

```json
{
  ...
  "scripts": {
    "stats": "webpack --profile --json > stats.json",
    ...
  },
  ...
}
```

**webpack.config.js**

```javascript
...

//if(TARGET === 'build') {
if(TARGET === 'build' || TARGET === 'stats') {
  ...
}

...
```

If you hit `npm run stats` now, you should find *stats.json* at your project root after it has finished processing. We can take this file and pass it to [the online tool](http://webpack.github.io/analyse/). Note that the tool works only over HTTP! If your data is sensitive, consider using [the standalone version](https://github.com/webpack/analyse) instead.

Besides helping you to understand your bundle composition, the tool can help you to optimize your output further.

## Deployment

There's no one right way to deploy our application. `npm run build` provides us something static to host. If you drop that on a suitable server, it will just work. One neat way to deal with it for small demos is to piggyback on GitHub Pages.

### Hosting on GitHub Pages

A package known as [gh-pages](https://www.npmjs.com/package/gh-pages) allows us to achieve this easily. You point it to your build directory first. It will then pick up the contents and push them to the `gh-pages` branch. To get started, hit

```bash
npm i gh-pages --save-dev
```

We are also going to need an entry point at *package.json*:

**package.json**

```json
{
  ...
  "scripts": {
    "deploy": "node ./lib/deploy.js",
    ...
  },
  ...
}
```

In order to get access to our build path, we need to expose something useful for `deploy` case. We can match for the case like this so we get useful configuration:

**webpack.config.js**

```javascript
...

//if(TARGET === 'build' || TARGET === 'stats') {
if(TARGET === 'build' || TARGET === 'stats' || TARGET === 'deploy') {

...
```

To glue it all together, we need a deployment script like this:

**lib/deploy.js**

```javascript
var ghpages = require('gh-pages');
var config = require('../webpack.config');

main();

function main() {
  ghpages.publish(config.output.path, console.error.bind(console));
}
```

If you hit `npm run deploy` now and everything goes fine, you should have your application hosted through GitHub Pages. You should find it at `https://<name>.github.io/<project>` assuming it worked.

## Conclusion

Beyond the features discussed Webpack allows you to [lazy load](https://webpack.github.io/docs/code-splitting.html) content through `require.ensure`. This is handy if you happen to have a specific dependency on some view and want to load it when you need it.

Our Kanban application is now ready to be served. We went from a chunky build to a slim one. Even better the production version can benefit from caching and it is able to invalidate it. You can also understand how isomorphic rendering works on a basic level.
