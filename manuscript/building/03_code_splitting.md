# Code Splitting

Web applications have the tendency to grow big as features are developed. The longer it takes for your application to load, the more frustrating it's to the user. This problem is amplified in a mobile environment where the connections can be slow.

Even though splitting bundles can help a notch, they are not the only solution, and you can still end up having to download a lot of data. Fortunately, it's possible to do better thanks to **code splitting**. It allows to load code lazily as you need it.

You can load more code as the user enters a new view of the application. You can also tie loading to a specific action like scrolling or clicking a button. You could also try to predict what the user is trying to do next and load code based on your guess. This way the functionality would be already there as the user tries to access it.

T> Incidentally, it's possible to implement Google's [PRPL pattern](https://developers.google.com/web/fundamentals/performance/prpl-pattern/) using webpack's lazy loading. PRPL (Push, Render, Pre-cache, Lazy-load) has been designed with mobile web in mind.

## Code Splitting Formats

Code splitting can be done in two primary ways in webpack: through a dynamic `import` or `require.ensure` syntax. The former is used in this project.

The goal is to end up with a split point that gets loaded on demand. There can be splits inside splits, and you can structure an entire application based on splits. The advantage of doing this is that then the initial payload of your application can be smaller than it would be otherwise.

![Code splitting](images/dynamic.png)

### Dynamic `import`

The [dynamic `import` syntax](https://github.com/tc39/proposal-dynamic-import) isn't in the official language specification yet. To use it, minor tweaks are needed especially at the Babel setup.

Dynamic imports are defined as `Promise`s:

```javascript
import(/* webpackChunkName: "optional-name" */ "./module").then(
  module => {...}
).catch(
  error => {...}
);
```

{pagebreak}

The optional name allows you to pull multiple split points into a single bundle. As long as they have the same name, they will be grouped together. Each split point generates a separate bundle by default.

The interface allows composition, and you could load multiple resources in parallel:

```javascript
Promise.all([
  import("lunr"),
  import("../search_index.json"),
]).then(([lunr, search]) => {
  return {
    index: lunr.Index.load(search.index),
    lines: search.lines,
  };
});
```

This creates separate bundles to request. If you wanted only one, you would have to use naming or define an intermediate module to `import`.

W> The syntax works only with JavaScript after configured the right way. If you use another environment, you may have to use alternatives covered in the following sections.

{pagebreak}

### `require.ensure`

[require.ensure](https://webpack.js.org/api/module-methods/#require-ensure) provides an alternate way:

```javascript
require.ensure(
  // Modules to load, but not execute yet
  ["./load-earlier"],
  () => {
    const loadEarlier = require("./load-earlier");

    // Load later on demand and include to the same chunk
    const module1 = require("./module1");
    const module2 = require("./module2");

    ...
  },
  err => console.error(err),
  "optional-name"
);
```

Often you can achieve what you want through a dynamic `import`, but it's good to know this alternate form exists as well. `require.ensure` supports naming as well and [the official example](https://github.com/webpack/webpack/tree/master/examples/named-chunks) shows the output in detail.

W> `require.ensure` relies on `Promise`s internally. If you use `require.ensure` with older browsers, remember to shim `Promise` using a polyfill such as [es6-promise](https://www.npmjs.com/package/es6-promise).

{pagebreak}

### `require.include`

The example above could be rewritten using webpack particular `require.include`:

```javascript
require.ensure(
  [],
  () => {
    require.include("./load-earlier");

    const loadEarlier = require("./load-earlier");

    // Load later on demand and include to the same chunk
    const module1 = require("./module1");
    const module2 = require("./module2");

    ...
  }
);
```

If you had nested `require.ensure` definitions, you could pull a module to the parent chunk using either syntax. It's a similar idea as you saw in the *Bundle Splitting* chapter.

T> The formats respect `output.publicPath` option. You can also use `output.chunkFilename` to shape where they output. Example: `chunkFilename: "[name].js"`.

{pagebreak}

## Setting Up Code Splitting

To demonstrate the idea of code splitting, you can use dynamic `import`. The Babel setup of the project needs additions to make the syntax work.

### Configuring Babel

Given Babel doesn't support the dynamic `import` syntax out of the box, it needs [babel-plugin-syntax-dynamic-import](https://www.npmjs.com/package/babel-plugin-syntax-dynamic-import) to work.

Install it first:

```bash
npm install babel-plugin-syntax-dynamic-import --save-dev
```

To connect it with the project, adjust the configuration as follows:

**.babelrc**

```json
{
leanpub-start-insert
  "plugins": ["syntax-dynamic-import"],
leanpub-end-insert
  ...
}
```

W> If you are using ESLint, you should install `babel-eslint` and set `parser: "babel-eslint"` in addition to `parserOptions.allowImportExportEverywhere: true` at ESLint configuration.

{pagebreak}

### Defining a Split Point Using a Dynamic `import`

The idea can be demonstrated by setting up a module that contains a string that replaces the text of the demo button:

**app/lazy.js**

```javascript
export default "Hello from lazy";
```

You also need to point the application to this file, so the application knows to load it. This can be done by binding the loading process to click. Whenever the user happens to click the button, you trigger the loading process and replace the content:

**app/component.js**

```javascript
export default (text = "Hello world") => {
  const element = document.createElement("div");

  element.className = "fa fa-hand-spock-o fa-1g";
  element.innerHTML = text;
leanpub-start-insert
  element.onclick = () => {
    import("./lazy")
      .then(lazy => {
        element.textContent = lazy.default;
      })
      .catch(err => {
        console.error(err);
      });
  };
leanpub-end-insert

  return element;
};
```

If you open up the application (`npm start`) and click the button, you should see the new text in the button.

![Lazy loaded content](images/lazy.png)

If you run `npm run build`, you should see something:

```bash
Hash: bd020a9fce9965e910a3
Version: webpack 3.8.1
Time: 2184ms
        Asset       Size  Chunks                    Chunk Names
    vendor.js    83.7 kB       2  [emitted]         vendor
  ...font.eot     166 kB          [emitted]
...font.woff2    77.2 kB          [emitted]
 ...font.woff      98 kB          [emitted]
  ...font.svg     444 kB          [emitted]  [big]
leanpub-start-insert
         0.js  315 bytes       0  [emitted]
leanpub-end-insert
  ...font.ttf     166 kB          [emitted]
       app.js    2.48 kB       1  [emitted]         app
      app.css    3.23 kB       1  [emitted]         app
leanpub-start-insert
     0.js.map  233 bytes       0  [emitted]
leanpub-end-insert
   app.js.map    2.13 kB       1  [emitted]         app
  app.css.map   84 bytes       1  [emitted]         app
vendor.js.map     178 kB       2  [emitted]         vendor
   index.html  274 bytes          [emitted]
    [6] ./app/index.js 176 bytes {1} [built]
   [14] ./app/main.css 41 bytes {1} [built]
   [15] ./app/component.js 464 bytes {1} [built]
...
```

That *0.js* is your split point. Examining the file reveals that webpack has wrapped the code in a `webpackJsonp` block and processed the code bit.

### Lazy Loading Styles

Lazy loading can be applied to styling as well. Expand the definition:

**app/lazy.js**

```javascript
leanpub-start-insert
import "./lazy.css";
leanpub-end-insert

export default "Hello from lazy";
```

And to have a style definition to load, set up a rule:

**app/lazy.css**

```css
body {
  color: blue;
}
```

The idea is that after *lazy.js* gets loaded, *lazy.css* is applied as well. You can confirm this by running the application (`npm start`). The same behavior is visible if you build the application (`npm run build`) and examine the output (`0.js`). This is due to the `ExtractTextPlugin` definition.

![Lazy styled content](images/lazy-styled.png)

{pagebreak}

### Defining a Split Point Using `require.ensure`

It's possible to achieve the same with `require.ensure`. Consider the full example below:

```javascript
export default (text = "Hello world") => {
  const element = document.createElement("div");

  element.className = "fa fa-hand-spock-o fa-1g";
  element.innerHTML = text;
  element.onclick = () => {
    require.ensure([], (require) => {
      element.textContent = require("./lazy").default;
    });
  };

  return element;
};
```

You could name the split point as outlined above. If you add another split point and give it the same name, the splits should end up in the same bundle.

T> [bundle-loader](https://www.npmjs.com/package/bundle-loader) gives similar results, but through a loader interface. It supports bundle naming through its `name` option.

T> The *Dynamic Loading* chapter covers other techniques that come in handy when you have to deal with more complicated splits.

{pagebreak}

## Code Splitting in React

The splitting pattern can be wrapped into a React component. Airbnb uses the following solution [as described by Joe Lencioni](https://gist.github.com/lencioni/643a78712337d255f5c031bfc81ca4cf):

```javascript
import React from "react";

// Somewhere in code
<AsyncComponent loader={() => import("./SomeComponent")} />

class AsyncComponent extends React.Component {
  constructor(props) {
    super(props);

    this.state = { Component: null };
  }
  componentDidMount() {
    this.props.loader().then(
      Component => this.setState({ Component })
    );
  }
  render() {
    const { Component } = this.state;
    const { Placeholder, ...props } = this.props;

    return Component ? <Component {...props} /> : <Placeholder>;
  }
}
AsyncComponent.propTypes = {
  loader: PropTypes.func.isRequired,
  Placeholder: PropTypes.node.isRequired,
};
```

T> [react-async-component](https://www.npmjs.com/package/react-async-component) wraps the pattern in a `createAsyncComponent` call and provides server side rendering specific functionality. [react-loadable](https://www.npmjs.com/package/react-loadable) is another option.

## Conclusion

Code splitting is a feature that allows you to push your application a notch further. You can load code when you need it to gain faster initial load times and improved user experience especially in a mobile context where bandwidth is limited.

To recap:

* **Code splitting** comes with extra effort as you have to decide what to split and where. Often, you find good split points within a router. Or you notice that specific functionality is required only when a particular feature is used. Charting is a good example of this.
* To use dynamic `import` syntax, both Babel and ESLint require careful tweaks. Webpack supports the syntax ouf of the box.
* Use naming to pull separate split points into the same bundles.
* The techniques can be used within modern frameworks and libraries like React. You can wrap related logic to a specific component that handles the loading process in a user-friendly manner.

You'll learn to tidy up the build in the next chapter.

T> The *Searching with React* appendix contains a complete example of code splitting. It shows how to set up a static site index that's loaded when the user searches information.

T> [webpack-pwa](https://github.com/webpack/webpack-pwa) illustrates the idea on a larger scale and discusses different shell based approaches. You get back to this topic in the *Multiple Pages* chapter.
