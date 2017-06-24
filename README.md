# hmr-redux-mapper
[![NPM version][npm-image]][npm-url]
[![Build/Test status][travis-image]][travis-url]

<div align="center">
  <a href="https://github.com/paulbrom/hmr-redux-mapper">
    <img src="https://raw.githubusercontent.com/paulbrom/hmr-redux-mapper/master/assets/redux-mapper-logo.png">
  </a>
  <h3><i>The usage mapping command line tool for <a href="http://redux.js.org/">Redux</a></i></h3>
  <h4>
    <p>&nbsp;</p>
  </h4>
</div>

<h2>Why do we need a 'Redux mapper'?</h2>

If you have written [Redux](http://redux.js.org/) applications in a hot-module-reloading environment, you have
probably written a significant amount of boilerplate code in your routing handler to hot-swap all the Redux reducer and
saga modules needed to render each route.  Adding new Redux stores to existing shared components requires each route
using those shared components to swap in those new stores, but it can be tedious to manually determine exactly which
routes end up using these shared components so that this requirement can be satisfied.

Enter `hmr-redux-mapper`, which is a build tool whose purpose is to eliminate the need to manually specify the Redux
components which are needed for hot swapping upon route change.  The tool is designed to recursively walk each component
in your project, starting at each route container, looking for all the Redux modules that are imported either directly
by the route container or by any of its child components.  That information is written in a mapping file
(`reducerMap.js`).

You can then use the `reduxMapperLoader` integration module to automatically hot-swap all the Redux modules needed for
a new route without the need to write any boilerplate code in your routing handlers to manually load the necessary
modules.

<h2>Installation</h2>

```
npm install hmr-redux-mapper --save-dev
```

<h2>Usage</h2>

The following usage instructions are specific to [React](https://facebook.github.io/react/), and specifically to
[react-router](https://github.com/ReactTraining/react-router), but in theory similar principles should apply to other
Redux HMR environments where modules need to be swapped on route change.

First, ensure the Redux mapper tool is run early in the build process (before [webpack](https://webpack.js.org/) or
[browserify](http://browserify.org/)).  If you use npm to perform your build, you can add lines similar to these to the
"scripts" section of `package.json:`

``` javascript
"scripts": {
   ...
   "redux-mapper": "node ./node_modules/hmr-redux-mapper/bin/hmr-redux-mapper",
   "prebuild": "npm run build:clean && npm run redux-mapper",
   ...
},
```

If you use [gulp](http://gulpjs.com/), [grunt](https://gruntjs.com/), or another build tool, you simply need to
ensure that you use the proper methodology for those tools to run `hmr-redux-mapper` as a node script early in the build
process (before webpack or browserify).

The build tool will look for a `redux-mapper.json` file in the root of your project that specifies some details about
the way your project folders are laid out, such as what subfolder paths in your project can potentially contain
route-handling components or Redux modules, what kind of filenames your Redux stores are implemented using (e.g.,
`reducer.js`, `saga.js`, etc), and where to output the resulting mapper files.  More details on the available build tool
configuration options and how to specify them to meet the needs of your project can be found in the API documentation.

Here is an example of the version of the file that is needed to integrate with
[react-boilerplate](https://github.com/react-boilerplate/react-boilerplate):

``` javascript
{
  "config": {
    "basePath": "app",
    "mainAppPath": "utils/request.js",
    "containerPaths": "containers,components",
    "actionFilenames": "actions.js,selectors.js",
    "globalReducersOutputPath": "globalReducerImports.js",
    "ignorePaths": "-spec",
    "reducerMapOutputPath": "reducerMap.js",
    "reduxPaths": "containers",
    "sagaFilename": "sagas.js"
  }
}
```

Once you have configured the build tool itself, you need to decorate each Redux reducer file with an constant that
provides a name for the reducer for mapping purposes:

``` javascript
// this is needed for hmr-redux-mapper to identify the reducer name for this reducer
export const PRM_REDUCER_NAME = 'home';
```

Finally, remove all the boilerplate code for loading Redux and component modules on route change and replace that
code with a handler from `reduxMapperLoader` that can load the necessary modules by referring to the `reducerMap.js`
generated by the build tool.

Right now, `reduxMapperLoader` only directly supports react-router.  To integrate with react-router, replace any
`getComponent` callbacks inside of your implementation of `createRoutes` calls to `reduxMapperLoader`'s
`getComponentFromReduxMapper` function.

Consider this example diff showing how to replace boilerplate component loading code with the
`reduxMapperLoader` (this example diff is based on app/routes.js from
[react-boilerplate](https://github.com/react-boilerplate/react-boilerplate)):

``` javascript
+import reduxMapperLoader from 'hmr-redux-mapper';
+import reducerMap from './reducerMap';

export default function createRoutes(store) {
   // create reusable async injectors using getAsyncInjectors factory
   const { injectReducer, injectSagas } = getAsyncInjectors(store);

+  const getComponentFromReduxMapper = reduxMapperLoader({
+    store,
+    createReducer,
+    reducerMap,
+    injectReducer,
+    injectSagas,
+    loadModule,
+    errorLoading,
+  });
+
   return [
     {
       path: '/',
       name: 'home',
-      getComponent(nextState, cb) {
-        const importModules = Promise.all([
-          import('containers/HomePage/reducer'),
-          import('containers/HomePage/sagas'),
-          import('containers/HomePage'),
-        ]);
-
-        const renderRoute = loadModule(cb);
-
-        importModules.then(([reducer, sagas, component]) => {
-          injectReducer('home', reducer.default);
-          injectSagas(sagas.default);
-
-          renderRoute(component);
-        });
-
-        importModules.catch(errorLoading);
-      },
+      getComponent: getComponentFromReduxMapper('./containers/HomePage/index.js'),
     }, {
       path: '/features',
       name: 'features',
-      getComponent(nextState, cb) {
-        import('containers/FeaturePage')
-          .then(loadModule(cb))
-          .catch(errorLoading);
-      },
+      getComponent: getComponentFromReduxMapper('./containers/FeaturePage'),
     }, {
       path: '*',
       name: 'notfound',
-      getComponent(nextState, cb) {
-        import('containers/NotFoundPage')
-          .then(loadModule(cb))
-          .catch(errorLoading);
-      },
+      getComponent: getComponentFromReduxMapper('./containers/NotFoundPage'),
     },
   ];
 }
```

That's right - all you need to do to implement react-router's getComponent call is specify the path to the component!
All of the necessary Redux modules and the React component itself will be injected automatically on route change by
`getCompoenntFromReduxMapper`, using the data collected by the build tool.

<h3>For more information, see the full, more detailed integration guide
<a href="https://github.com/paulbrom/hmr-redux-mapper/blob/master/docs/Integration_Guide.md">here</a></h3>

<h2>Contributing</h2>

Don't hesitate to create a pull request.  Right now we only offer direct support for npm-built React projects using
react-router, and we hope that the community can help integrate this tool with other Redux HMR environments.  Any
contribution will be greatly appreciated!

<h3>For more information on how to build, run, and create tests, see the guide
<a href="https://github.com/paulbrom/hmr-redux-mapper/blob/master/docs/Build_and_Contribute.md">here</a></h3>

<h2>Maintainers</h2>

<div>
  <img width="100" height="100"
    src="https://avatars.githubusercontent.com/paulbrom">
  <div>
    <a href="https://github.com/paulbrom">Paul Broman</a>
    <div><sub>Senior Front-End Engineer</sub></div>
    <div><sup>PureCars/Raycom Media</sup></div>
  </div>
</div>

<h2>LICENSE</h2>

(c)2017 Raycom Media, Inc.
Released under the MIT license.

#### [MIT](./LICENSE)

[travis-url]: http://travis-ci.org/paulbrom/hmr-redux-mapper
[travis-image]: https://secure.travis-ci.org/paulbrom/hmr-redux-mapper.png?branch=master
[npm-url]: https://npmjs.org/package/hmr-redux-mapper
[npm-image]: https://badge.fury.io/js/hmr-redux-mapper.svg
