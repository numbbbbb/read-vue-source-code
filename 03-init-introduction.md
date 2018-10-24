# Init - Introduction

This article belongs to the series [Read Vue Source Code](https://github.com/numbbbbb/read-vue-source-code).

In this article, we will learn:

- What those mixins do
- Understand the init process

## What those mixins do

We are inside `src/core/instance/index.js` now.

```javascript
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue
```

First, we will walk through those five mixins, find out what they do. Then we go into the `_init` function and see what happens when you execute `var app = new Vue({...})`.

### initMixin

Open `./init.js`, scroll to the bottom and read definitions.

This file defines:

- function `initMixin()`, it defines `Vue.prototype._init`, we will come back at next section
- function `initInternalComponent()`, its comments implies that this function can speed up internal component instantiation because dynamic options merging is pretty slow
- function `resolveConstructorOptions()`, it collects options
- function `resolveModifiedOptions()`, this is related to [a bug](https://github.com/vuejs/vue/issues/4976). In short, it can let you modify or attach options during hot-reload
- function `dedupe()`, used by `resolveModifiedOptions` to ensure lifecycle hooks won't be duplicated

### stateMixin

Open `./state,js`, this is a long file, search `statemixin`.

```javascript
export function stateMixin (Vue: Class<Component>) {
  // flow somehow has problems with directly declared definition object
  // when using Object.defineProperty, so we have to procedurally build up
  // the object here.
  const dataDef = {}
  dataDef.get = function () { return this._data }
  const propsDef = {}
  propsDef.get = function () { return this._props }
  if (process.env.NODE_ENV !== 'production') {
    dataDef.set = function (newData: Object) {
      warn(
        'Avoid replacing instance root $data. ' +
        'Use nested data properties instead.',
        this
      )
    }
    propsDef.set = function () {
      warn(`$props is readonly.`, this)
    }
  }
  Object.defineProperty(Vue.prototype, '$data', dataDef)
  Object.defineProperty(Vue.prototype, '$props', propsDef)

  Vue.prototype.$set = set
  Vue.prototype.$delete = del

  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: Function,
    options?: Object
  ): Function {
    const vm: Component = this
    options = options || {}
    options.user = true
    const watcher = new Watcher(vm, expOrFn, cb, options)
    if (options.immediate) {
      cb.call(vm, watcher.value)
    }
    return function unwatchFn () {
      watcher.teardown()
    }
  }
}
```

This function defines:

- `dataDef` and it's getter
- `propsDef` and it's getter
- setters for `dataDef` and `propsDef` if not built for production, which just log two warnings
- `dataDef` to `Vue.prototype` as `$data`
- `propsDef` to `Vue.prototype` as `$props`
- `Vue.prototype.$set` and `Vue.prototype.$delete`
- `Vue.prototype.$watch`

Sounds familiar? Yes, here we get `$data`, `$props`, `$set`, `$delete` and `$watch`. Read this file carefully, you can learn some coding skills and use them in your own projects.

Have you noticed the `Watcher`? Seems like an important class! You are right, in later articles we will explain `Observer`, `Dep` and `Watcher`. They cooperate to implement data and view synchronization.

### eventsMixin

Open `./events.js` and search `eventsMixin`, it's too long to put the screenshot here, so read it by yourself.

This function defines:

- `Vue.prototype.$on`
- `Vue.prototype.$once`
- `Vue.prototype.$off`
- `Vue.prototype.$emit`

You must have used them for many times, just read the code and learn how to implement event manipulation elegantly.

### lifecycleMixin

Open `lifecycle.js`, scroll down to find `lifecycleMixin`.

This function dedines:

- `Vue.prototype._update()`, DOM updating happens here! Will be covered in later articles
- `Vue.prototype.$forceUpdate()`
- `Vue.prototype.$destroy()`

Hey, what's that below `$destroy`? `mountComponent`! We have seen it before, it's the core of `$mount` and is wrapped twice.

Keep going, we get several functions about the component. They are used in DOM updating, just ignore them now.

### renderMixin

Open `./render.js`, it defines `Vue.prototype._render()` and some helpers. They will also appear in later articles, just keep in mind that we meet `_render` here.

---

Okay, so now we understand what those mixins do, they just set some functions to `Vue.prototype`.

![](http://i.imgur.com/MhqgVXP.jpg)


The important thing here is how to divide and organize a bunch of functions. How many parts would you make if you are the author? Which part should one function go? Think from the point of author's view, it's very interesting and helpful.

## Understand the init process

After looking the static parts, now we go back to the core.

```
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}
```

Here we have a small demo from Vue's official document:

```
var app = new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue!'
  }
})
```

Now load them to your mind and let's begin the execution.

First, we call `new Vue({...})`, which means:

```
options = {
  el: '#app',
  data: {
    message: 'Hello Vue!'
  }
}
```

Then `this._init(options)`. Remember where `_init()` is defined? Yes, in `./init.js`, open it and read the function codes.

`_init()` do:

- set `_uid`
- mark performance start tag 
- set `_isVue`
- set options
- set `_renderProxy`. Proxy is used during development to show you more render information
- set `_self`
- call a bunch of init functions
- mark performance end tag
- call `$mount()` to update DOM

Now we focus on those init functions.

```javascript
initLifecycle(vm)
initEvents(vm)
initRender(vm)
callHook(vm, 'beforeCreate')
initInjections(vm) // resolve injections before data/props
initState(vm)
initProvide(vm) // resolve provide after data/props
callHook(vm, 'created')
```

`callHook()` is easy to understand, it just calls your hook functions. Next, we will explain the other six init functions in detail.

### initLifecycle

It's located in `./lifecycle`. 

This function connects this component with its parent, initializes some variables used in lifecycle methods.

### initEvents

It's located in `./events.js`.

This function initializes variables and updates them with its parent's listeners.

### initRender

It's located in `./render.js`.

This function initializes `_vnode`, `_staticTrees` and some other variables and methods.

Here we meet VNode for the first time. 

What is VNode? It's used to build the VDom. VNode and VDom correspond to real Node and DOM. The reason Vue chose these two representations is performance. 

When your data change, Vue needs to update the webpage. The simplest way is refreshing the whole page. But it costs a lot of browser resources and much of them are just wasted. Normally you just update a few properties, why not just update the parts that change? Thus Vue adds a layer of VNode and VDom between data and view, implements an algorithm to calculate the best DOM manipulation strategy and apply that to the DOM.

We will talk about render and update later.

### initInjections

It's located in `./inject.js`.

This function is short and simple, it just resolves the injections in options and set them to your component.

But wait, what's that, is it `defineProperty`? No, it's `defineReactive`. The word `reactive` must remind you of something, Vue can update view automatically while data change, maybe we can find something related to that inside this function, let's go.

Open `../observer/index.js` and search `defineReactive`.

This function first defines `const dep = new Dep()`, does some validation, then extracts the getter and setter.

```javascript
let childOb = observe(val) // <-- IMPORTANT
Object.defineProperty(obj, key, {
  enumerable: true,
  configurable: true,
  get: function reactiveGetter () {
    const value = getter ? getter.call(obj) : val
    if (Dep.target) {
      dep.depend() // <-- IMPORTANT
      if (childOb) {
        childOb.dep.depend() // <-- IMPORTANT
      }
      if (Array.isArray(value)) {
        dependArray(value)
      }
    }
    return value
  },
  set: function reactiveSetter (newVal) {
    const value = getter ? getter.call(obj) : val
    /* eslint-disable no-self-compare */
    if (newVal === value || (newVal !== newVal && value !== value)) {
      return
    }
    /* eslint-enable no-self-compare */
    if (process.env.NODE_ENV !== 'production' && customSetter) {
      customSetter()
    }
    if (setter) {
      setter.call(obj, newVal)
    } else {
      val = newVal
    }
    childOb = observe(newVal) // <-- IMPORTANT
    dep.notify() // <-- IMPORTANT
  }
})
```

Next it defines `childOb = observe(val)`, set a new property to our component.

I have marked the key parts in comment. Even if you haven't read the related code, you can tell how Vue do view updating while data changes. It just wraps value with getter and setter inside which it constructs dependency and sends notify.

This `defineReactive` function is used in many places, not only in initInjections, we will talk about `Observer`, `Dep` and `Watcher` in later articles, just go back to our initialization and go on.

### initState

It's located in `./state.js`.

```javascript
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch) initWatch(vm, opts.watch)
}
```

Old friends again. Here we get our props, methods, data, computed properties and watch functions. Let's check them one by one.

#### initProps

Do some validations and use `defineReactive` to wrap props and set them to the component.

#### initMethods

Just set methods to the component.

#### initData

More validations, but here it uses `proxy` to set data, search `proxy` and read it, you know it will proxy operations like `this.name` to `this._data['name']`. 

At the end, this function calls `observe(data, true) /* asRootData */`. We will talk about Observer in later articles, here I just want to explain that `true`. Each observed object has a property called `vmCount` which means how many components use this object as root data. If you call `observe` with `true`, it will call `vmCount++`. The default value of `vmCount` is `0`.

Maybe this property will be used in other process, but after a global search, I just find Vue uses it as a sign for root data. If `ob && ob.vmCount`, this object must be a root data.

#### initComputed

This function first extracts the function you put in as a getter, then creates a Watcher with it and store in `watchers` array. Finally, it calls `defineComputed` to set this computed property to the component.

You can guess what the name Watcher means, but we will leave it to later articles.

#### initWatch

This function creates watcher for each input uses `createWatcher()`. `createWatcher()` calls `vm.$watch()` which is defined in `stateMixin`. Scroll to the end to see it. `vm.$watch()` will create Watcher and...What?! It may call `createWatcher()` again! What the hell is it doing?

Read it carefully, if `cb` is plain object, `$watch()` will call `createWatcher()`. And inside `createWatcher()`, it will extract options and handler if the handler(in `$watch()` it's named `cb`) is plain object. Alright, `$watch()` just through it back because he doesn't want to do extra jobs.

### initProvider

It's located also in `./inject.js`.

This function extracts providers in options and calls them on the component.

Have you noticed the comments after `initInjections` and `initProvider`? It says:

```
initInjections(vm) // resolve injections before data/props
initState(vm)
initProvide(vm) // resolve provide after data/props
```

Why do they have this order? I'll let you answer this.

---

That's all. Hard to remember all inits? I made a picture for you:

![](http://i.imgur.com/DImNrXn.jpg)

This article is a little long and contains many details. Init process is the basement of later articles, so make sure you have understood all contents.

I'm not intend to tell you everything, so I suggest you read the whole init process again and go into the implementation of unfamiliar functions to see how they work.

## Next Step

This article shows the whole initialization process. After initialization, data can be modified, views will also sync with data. How does Vue implement the data updating process? Next article will reveal it.

Read next chapter: [Dynamic Data - Observer, Dep and Watcher](https://github.com/numbbbbb/read-vue-source-code/blob/master/04-dynamic-data-observer-dep-and-watcher.md).

## Practice

```
initInjections(vm) // resolve injections before data/props
initState(vm)
initProvide(vm) // resolve provide after data/props
```

Why do they have this order? I'll let you answer this.

Hint: think from another side, what will happen if you change their order?

