---
title: PreFetch Feature
---
The PreFetch is a feature **available when using Quasar CLI** that allows a way for the components picked up by Vue Router (defined in `/src/router/routes.js`) to:
* pre-fetch data
* validate the route
* redirect to another route should some conditions are not met (like user should be logged in)
* can help in initializing the Store state

All the above will run before the actual route component gets to be rendered.

**It is designed to work with all Quasar modes** (SPA, PWA, SSR, Cordova, Electron), but it is especially useful for SSR builds.

## Installation
```js
// quasar.conf.js
return {
  preFetch: true
}
```
::: warning
  When you use it to pre-fetch data, you are required to use a Vuex Store, so make sure that your project folder has the `/src/store` folder when you create your project, otherwise generate a new project and copy the store folder contents to your current project.
:::

## How It Helps SSR Mode
This feature is especially useful for the SSR mode (but not limited to it only). During SSR, we are essentially rendering a "snapshot" of our app, so if the app relies on some asynchronous data, **these data need to be pre-fetched and resolved before we start the rendering process**.

Another concern is that on the client, the same data needs to be available before we mount the client side app - otherwise the client app would render using different state and the hydration would fail.

To address this, the fetched data needs to live outside the view components, in a dedicated data store, or a "state container". On the server, we can pre-fetch and fill data into the store before rendering. The client-side store will directly pick up the server state before we mount the app.

## When It Gets Activated
The `preFetch` hook (described in next sections) is determined by the route visited - which also determines what components are rendered. In fact, the data needed for a given route is also the data needed by the components rendered at that route. **So it is natural (and also required) to place the hook logic ONLY inside route components.** This includes `/src/App.vue`, which in this case will run only once at the app bootup.

Let's take an example in order to understand when the hook is being called. Let's say we have these routes and we've written `preFetch` hooks for all these components:

```js
// routes
[
  {
    path: '/',
    component: LandingPage
  },
  {
    path: '/shop',
    component: ShopLayout,
    children: [
      {
        path: 'all',
        component: ShopAll
      },
      {
        path: 'new',
        component: ShopNew
      },
      {
        path: 'product/:name',
        component: ShopProduct,
        children: [{
          path: 'overview',
          component: ShopProductOverview
        }]
      }
    ]
  }
]
```

Now, let's see how the hooks are called when user visits these routes in the order specified below, one after another.

| Route being visited | Hooks called from | Observations |
| --- | --- | --- |
| `/` | App.vue then LandingPage | App.vue hook is called since our app boots up. |
| `/shop/all` | ShopLayout then ShopAll | - |
| `/shop/new` | ShopNew | ShopNew is a child of ShopLayout, and ShopLayout is already rendered, so ShopLayout isn't called again. |
| `/shop/product/pijamas` | ShopProduct | - |
| `/shop/product/shoes` | ShopProduct | Quasar notices the same component is already rendered, but the route has been updated and it has route params, so it calls the hook again. |
| `/shop/product/shoes/overview` | ShopProduct then ShopProductOverview | ShopProduct has route params so it is called even though it's already rendered. |
| `/` | LandingPage | - |

## Usage
The hook is defined as a custom static function called `preFetch` on our route components. Note that because this function will be called before the components are instantiated, it doesn't have access to `this`.

``` html
<!-- some .vue component used as route -->
<template>
  <div>{{ item.title }}</div>
</template>

<script>
export default {
  // our hook here
  preFetch ({ store, currentRoute, previousRoute, redirect, ssrContext }) {
    // fetch data, validate route and optionally redirect to some other route...

    // ssrContext is available only server-side in SSR mode

    // No access to "this" here as preFetch() is called before
    // the component gets instantiated.

    // Return a Promise if you are running an async job
    // Example:
    return store.dispatch('fetchItem', currentRoute.params.id)
  },

  computed: {
    // display the item from store state.
    item () {
      return this.$store.state.items[this.$route.params.id]
    }
  }
}
</script>
```

### Redirecting Example
Example of redirecting the user under some circumstances, like when they try to access a page that only an authenticated user should see.

```js
// We assume here we already wrote the authentication logic
// in the Vuex Store, so take as a high-level example only.
preFetch ({ store, redirect }) {
  if (!store.state.authenticated) {
    // IMPORTANT! Always use the String form of a
    // route if also building for SSR. The Object form
    // won't work on SSR builds.
    redirect('/login')
  }
}
```

If `redirect(false)` is called (supported only on client-side!), it aborts the current route navigation. Note that if you use it like this in `src/App.vue` it will halt the app bootup, which is undesirable.

### Using preFetch to Initialize the Store
The `preFetch` hook runs only once, when the app boots up, so you can use this opportunity to initialize the Vuex Store here.

```js
// App.vue
export default {
  // ...
  preFetch ({ store }) {
    // initialize something in store here
  }
}
```

### Store Code Splitting
In a large application, your Vuex store will likely be split into multiple modules. Of course, it is also possible to code-split these modules into corresponding route component chunks. Suppose we have the following store module:

```js
// src/store/foo.js
// we've merged everything into one file here;
// an initialized Quasar project splits every component of a Vuex module
// into separate files, but for the sake of the example
// here in the docs, we show this module as a single file

export default {
  namespaced: true,
  // IMPORTANT: state must be a function so the module can be
  // instantiated multiple times
  state: () => ({
    count: 0
  }),
  actions: {
    inc: ({ commit }) => commit('inc')
  },
  mutations: {
    inc: state => state.count++
  }
}
```

Now, we can use `store.registerModule()` to lazy-register this module in a route component's `preFetch()` hook:

```js
// inside a route component
<template>
  <div>{{ fooCount }}</div>
</template>

<script>
// import the module here instead of in `src/store/index.js`
import fooStoreModule from 'store/foo'

export default {
  preFetch ({ store }) {
    store.registerModule('foo', fooStoreModule)
    return store.dispatch('foo/inc')
  },

  // IMPORTANT: avoid duplicate module registration on the client
  // when the route is visited multiple times.
  destroyed () {
    this.$store.unregisterModule('foo')
  },

  computed: {
    fooCount () {
      return this.$store.state.foo.count
    }
  }
}
</script>
```

Also note that because the module is now a dependency of the route component, it will be moved into the route component's async chunk by Webpack.

## Loading State
A good UX includes notifying the user that something is being worked on in the background while he/she waits for the page to be ready. Quasar CLI offers two options for this out of the box.

### LoadingBar
When you add Quasar [LoadingBar](/quasar-plugins/loading-bar) plugin to your app, Quasar CLI will use it while it runs the preFetch hooks by default.

### Loading
There's also the possibility to use Quasar [Loading](/quasar-plugins/loading) plugin. Here's an example:

```js
// a route .vue component
import { Loading } from 'quasar'

export default {
  // ...
  preFetch ({ /* ... */ }) {
    Loading.show()

    return new Promise(resolve => {
      // do something async here
      // then call "resolve()"
    }).then(() => {
      Loading.hide()
    })
  }
}
```
