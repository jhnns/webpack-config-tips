# webpack-config-tips

**No so secret tips and examples on how to write webpack configs.**

## Config features

### [You don't need a webpack config](examples/no-config)

When no config is provided webpack assumes the following default config:

```js
const {resolve} = require("path");

module.exports = {
    entry: resolve(process.cwd(), "src", "index.js"),
    output: {
        path: resolve(process.cwd(), "dist"),
        filename: "main.js",
    }
};
```

You will still see an error like this in your console:

![](./assets/stdout-error-mode.jpg)

This means that webpack used production defaults because you haven't specified a <code>mode</code>.

The <code>mode</code> has been introduced with webpack 4 and can be set to <code>development</code>, <code>production</code> and <code>none</code>. It allows webpack to choose the best default configuration for the given environment.

**It's strongly recommended to set the <code>mode</code> and let webpack do the rest. 😎**

You can set the `mode` via the command line...

```sh
webpack --mode development
```

...or inside your webpack config:

```js
module.exports = {
    mode: "development",
};
```

### [You can use TypeScript in your webpack config](examples/typescript)

- Step 1: Rename your `webpack.config.js` to `webpack.config.ts`.
- Step 2: Install `@types/webpack` and `ts-node`.
- Step 3: Create a variable using the `webpack.Configuration` type:

```ts
import webpack from "webpack";

const config: webpack.Configuration = {
    // ...
};

export default config;
```

You also need to set the `esModuleInterop` flag in your `tsconfig.json`:

```json
{
    "compilerOptions": {
        "esModuleInterop": true
    }
}
```

This gives you nice intellisense in your config:

![](./assets/ts-config-1.jpg)
![](./assets/ts-config-2.jpg)

**Note:**
- This is a [webpack-cli](https://webpack.js.org/api/cli/) feature. It won't work if you're using webpack's node API.
- [`ts-node`](https://github.com/TypeStrong/ts-node) needs to be available in your `node_modules`
- webpack-cli uses [interpret](https://github.com/gulpjs/interpret) which maintains a dictionary of file extensions and associated module loaders.
- This means that you can also use [Babel](https://babeljs.io/) to precompile your `webpack.config.js`. Rename it to `webpack.config.babel.js` and make sure that there is a Babel config.
- Devon Zuegel has written a [comprehensive guide on using TypeScript in your webpack config](https://medium.com/webpack/unambiguous-webpack-config-with-typescript-8519def2cac7).

### [Did you know that your webpack config can also export a function?](examples/function)

The function should return the webpack config:

```js
module.exports = () => {
    return {
        mode: "development",
    };
};
```

Where is that useful?

Now you can pass arguments to your config via webpack's CLI:

```sh
webpack --env.mode=production --env.debug
```

```js
module.exports = (env = {}) => {
    const {
        mode = "development",
        debug,
    } = env;

    return {
        mode,
        output: {
            pathinfo: debug === true,
        },
    };
};
```

This feature is called [Environment Options](https://webpack.js.org/api/cli/#environment-options).

### [Multi-compiler mode](examples/multi)

You can also return an array of configurations. This switches webpack into **multi-compiler mode**:

```js
module.exports = [
    {
        name: "web",
        target: "web",
        output: {
            path: resolve(__dirname, "dist", "web")
        },
    },
    {
        name: "node",
        target: "node",
        output: {
            path: resolve(__dirname, "dist", "node")
        },
    },
];
```

![](./assets/stdout-multi-compiler.jpg)

This is useful for isomorphic apps that need to create two bundles, one for the browser and one for Node.

It's also useful for creating localized bundles:

```js
const I18nPlugin = require("i18n-webpack-plugin");

module.exports = Object
    .entries({
        en: require("./en.json"),
        de: require("./de.json"),
    })
    .map(([language, translations]) => ({
        name: language,
        output: {
            path: resolve(__dirname, "dist", language)
        },
        plugins: [new I18nPlugin(translations)]
    }));
```

In multi-compiler mode webpack builds each config concurrently in the same process while re-using the file system cache.

**Note:**

**Multi-compiler mode does not execute each build in parallel**. Often you will see significant longer build times.

Use [parallel-webpack](https://www.npmjs.com/package/parallel-webpack) from [Trivago](https://tech.trivago.com/), one of our Platinum sponsors, if you want to execute each config in parallel.

### [Async webpack configs](examples/async)

Just put an `async` before your config function:

```js
module.exports = async () => {
    // load some config values from database
    return {
        // ...
    };
};
```

With the language example from the previous tip:

```js
module.exports = async () => {
    const languages = await loadLanguagesFromProvider();

    return Object
        .entries(languages)
        .map(([language, translations]) => ({
            /* ...*/
        }));
};
```

Also the `entry` option can be an async function:

```js
module.exports = {
    entry: async () => loadEntriesFromDb(),
};
```

This can be useful if you're building a static site generator that pulls all entries from a database.

It's not the most important feature but still good to know 😉.

### Loader configuration tips

1.  Prefer `module.rules[].oneOf`
    -   Explain rules
    -   Order of rules is important
1.  Prefer `module.rules[].include` over `exclude`
1.  `module.rules[].resourceQuery` to match on the querystring in a module request like `import text from "./file?raw"` `{ resourceQuery: "?raw", use: "raw-loader" }`
1.  `module.rules[].issuer` to match on the importing module

### Development tips

#### [Use nodemon to restart webpack everytime the config changes](examples/webpack-nodemon)

Running `webpack --watch` tells webpack to watch all source files that contribute to the bundle. But what if you made changes to the webpack config itself? webpack does not notice any changes to files that are required by webpack itself.

With [nodemon](https://github.com/remy/nodemon) you can restart webpack everytime you make changes to the webpack config:

```sh
nodemon --watch webpack.config.js --exec "webpack --watch --mode=development"
```

#### [Use `require.resolve` when referencing absolute paths](examples/webpack-nodemon)

When writing your own webpack config it can happen easily that a file path is not correct.
Although webpack will report some errors like a missing entry file, it will not complain about all errors.

For instance: when using `resolve.alias` it can happen easily that the path to be aliased is not correct anymore.
In this case it's good to use Node's `require.resolve` which will throw an error as soon as the file can not be found.

This is the easiest and safest way to replace one module with another using `resolve.alias`:

```js
module.exports = {
    resolve: {
        alias: {
            // Each require.resolve() will throw an error if the module cannot be found
            [require.resolve("./src/index.js")]: require.resolve("./src/replaced.js"),
        },
    },
};
```

Also useful if you want to execute a loader just on a certain file:

```js
module.exports = {
    module: {
        rules: [
            {
                include: [require.resolve("./src/replaced.js")],
                use: "raw-loader",
            },
        ],
    },
};
```

Now everytime you move the file but don't update the path in your webpack config, `require.resolve` will complain about it.

#### [Use `.filter(Boolean)` to remove unwanted loaders or plugins depending on the env](examples/webpack-nodemon)

Adding `.filter(Boolean)` at the end of an array removes all falsy values from an array:

```js
module.exports = {
    module: {
        rules: [
            isDev && {
                include: /\.css$/,
                loader: "style-loader",
            },
        ].filter(Boolean),
    },
    plugins: [
        analyze && new BundleAnalyzerPlugin(),
    ].filter(Boolean),
};
```


1.  Avoid config splitting

### Speed up webpack build

-   Build time performance `--progress --profile`
-   Use `include`
-   `noParse` to speed up builds
-   Remove unnecessary steps like
    -   Type check
    -   Linting
-   cache-loader
-   thread-loader

### Optimization

-   webpack-bundle-analyzer
-   `maxSize` with HTTP2
    -   Webpack's defaults are usually good enough for most people
    -   If you've already done your homework (reduce JS payload), you can optimize here
    -   The goal: create smaller chunks than can be cached individually
    -   Why? HTTP2 has no head-of-line blocking
    -   The problem: find a good algorithm that selects modules that change often together (you could group by folder structure)
    -   Also problem: GZIP works better with larger files
-   `output.jsonpScriptType: "module"`
    -   Loads async chunks as modules instead of oldschool script tags
    -   Be careful with dependencies: All code is interpreted in strict mode.

### Debugging and error reporting

-   `module.strictExportPresence`
    -   true turns missing export warnings into errors
    -   Could be true for production builds in the future
-   `output.crossOriginLoading: "anonymous"`
    -   Uses the crossorigin attribute when loading chunks
    -   Useful when the bundle is served via a CDN with a different origin
    -   With no attribute, the global error handler will only report "Script error"
    -   https://stackoverflow.com/a/7778424
-   `stats.errorDetails: true` adds more information to errors like "Module not found"
-   `stats.logging: "verbose"`
-   `stats.optimizationBailout` shows which optimization didn't work for which module
    -   Example: Concatenation
-   `output.pathinfo: true`
    -   Enabled in development by default
    -   Disabled in production
    -   Useful to debug production build
