---
title: TypeScript
sidebar_order: 3
---

Please read [Source Maps docs]({%- link _documentation/platforms/node/sourcemaps.md -%}) first to learn how to configure Sentry SDK, upload artifacts to our servers or use Webpack (if you’re willing to use _ts-loader_ for your TypeScript compilation).

Using Sentry SDK and Source Maps with TypeScript unfortunatelly requires slightly more configuration.

There are two main reasons for this:

1.  TypeScript compiles and outputs all files separately
2.  SourceRoot is by default set to, well, source directory, which would require uploading artifacts from 2 separate directories and modification of source maps themselves

We can still make it work with two additional steps, so let’s do this.

The first one is configuring TypeScript compiler in a way, in which we’ll override _sourceRoot_ and merge original sources with corresponding maps. The former is not required, but it’ll help Sentry display correct filepaths, eg. _/lib/utils/helper.ts_ instead of a full one like _/Users/Sentry/Projects/TSExample/lib/utils/helper.ts_. You can skip this option if you’re fine with such a long names.

Assuming you already have a _tsconfig.json_ file similar to this:

```json
{
  "compilerOptions": {
    "target": "es6",
    "module": "commonjs",
    "allowJs": true,
    "moduleResolution": "node",
    "outDir": "dist"
  },
  "include": [
    "./src/**/*"
  ]
}
```

create a new one called _tsconfig.production.json_ and paste the snippet below:

```json
{
  "extends": "./tsconfig",
  "compilerOptions": {
    "sourceMap": true,
    "inlineSources": true,
    "sourceRoot": "/"
  }
}
```

From now on, when you want to run the production build, that’ll be uploaded you specify this very config, eg. _tsc -p tsconfig.production.json_. This will create necessary source maps and attach original sources to them instead of making us to upload them and modify source paths in our maps by hand.

The second step is changing events frames, so that Sentry can link stacktraces with correct source files.

This can be done using _dataCallback_, in a very similar manner as we do with a single entrypoint described in Source Maps docs, with one, very important difference. Instead of using _basename_, we have to somehow detect and pass the root directory of our project.

Unfortunately, Node is very fragile in that manner and doesn’t have a very reliable way to do this. The easiest and the most reliable way we found, is to store the ___dirname_ or _process.cwd()_ in the global variable and using it in other places of your app. This _has to be done_ as the first thing in your code and from the entrypoint, otherwise the path will be incorrect.

If you want, you can set this value by hand to something like _/var/www/html/some-app_ if you can get this from some external source or you know it won’t ever change.

This can also be achieved by creating a separate file called _root.js_ or similar that’ll be placed in the same place as your entrypoint and exporting obtained value instead of exporting it globally.

```javascript
// index.js

// This allows TypeScript to detect our global value
declare global {
  namespace NodeJS {
    interface Global {
      __rootdir__: string;
    }
  }
}

global.__rootdir__ = __dirname || process.cwd();
```

```javascript
Sentry.init({
  dsn: '___PUBLIC_DSN___',
  integrations: [new Sentry.Integrations.RewriteFrames({
    root: global.__rootdir__
  })]
});
```

This config should be enough to make everything work and use TypeScript with Node and still being able to digest all original sources by Sentry.
