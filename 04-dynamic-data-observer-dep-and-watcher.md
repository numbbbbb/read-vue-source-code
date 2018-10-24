# Dynamic Data - Observer, Dep and Watcher



This article belongs to the series [Read Vue Source Code](https://github.com/numbbbbb/read-vue-source-code).

In this article, we will learn:

- Observer
- Dep
- Watcher
- How they cooperate

In [previous article](https://github.com/numbbbbb/read-vue-source-code/blob/master/03-init-introduction.md), we have learned how Vue does the initialization. After init, many interesting things happen.

For example, if you change one of your properties `name`, then your webpage is automatically updated with the new value.

How to implement that? You will see in this article.

I won't give you the entire structure now, cause I want to show you how I build that through reading source code.

## Observer

In the previous article, we have seen `defineReactive` which is used to make a property `reactive`. Let's see its usage in `defineReactive()`.

```javascript
/**
 * Define a reactive property on an Object.
 */
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: Function
) {
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set

  let childOb = observe(val)  // <-- IMPORTANT
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
          dependArray(value) // <-- IMPORTANT
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
}
```

Here are the key points:

```
const dep = new Dep()
let childOb = observe(val)

...
  dep.depend()
  childOb.dep.depend()
  dependArray(value)
  
...
  childOb = observe(newVal)
  dep.notify()
```

Here we meet `Dep`, `observe()`, `dependArray()`, `depend()` and `notify()`.

It's clear that `observe()` and `dependArray()` are helpers, let's read them first.

```javascript
/**
 * Attempt to create an observer instance for a value,
 * returns the new observer if successfully observed,
 * or the existing observer if the value already has one.
 */
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value)) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    observerState.shouldConvert &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

`observe()` will extract the exist observer or create a new one with `new Observer(value)`. Notice that observe only works for an object, primitive value won't be observed.

If this value is used as root data, it will increments `ob.vmCount++`, we have talked about that in init process.

Okay, now we have got or created the watcher. Next, `dependArray()`.

```javscript
/**
 * Collect dependencies on array elements when the array is touched, since
 * we cannot intercept array element access like property getters.
 */
function dependArray (value: Array<any>) {
  for (let e, i = 0, l = value.length; i < l; i++) {
    e = value[i]
    e && e.__ob__ && e.__ob__.dep.depend()
    if (Array.isArray(e)) {
      dependArray(e)
    }
  }
}
```

It just iterates the array recursively and calls `e.__ob__.dep.depend()` which leads us to `depend()` again.

So now we have found the usage of `Dep()`, `Observer()`, `Watcher()`. And `dep.depend()`, `dep.notify()`.

If you use `defineReactive()` to convert a property, that reactive property has one `dep` and one `childOb` set by `observe(val)` if the value is object.

Let's read `Observer()` now.

```javscript
/**
 * Observer class that are attached to each observed
 * object. Once attached, the observer converts target
 * object's property keys into getter/setters that
 * collect dependencies and dispatches updates.
 */
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that has this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      const augment = hasProto
        ? protoAugment
        : copyAugment
      augment(value, arrayMethods, arrayKeys)
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }

  /**
   * Walk through each property and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i], obj[keys[i]])
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

It first defines a `__ob__` property to the value you pass in. 

Then if the value is an array, it will intercept array methods(like `push`, `pop`) to make sure Vue can detect array manipulation. After that, it calls `observeArray()` which will iterates items and call `observe()`.

If the value is not an array, this function just walks through all keys and use `defineReactive()` to convert all values into a reactive property.

As you can see, `defineReactive()` calls `new Observer()`, `Observer()` may also call `defineReactive()`. Thus, when you want to convert a property with `defineReactive()`, it will recursively converts all sub properties into reactive property.

**To be clear, we use `defineReactive()` to create reactive PROPERTY, and we use `observe()` to create Observer for the VALUE of that PROPERTY(if the value is object).**

The reason is simple. If the value is an object, change the property of that object won't trigger the setter of property. Property only save the reference to that object in memory, change the content of that object won't affect it's memory address, thus won't really change the property's value.

If we have a `data` like this:

```
data: {
  name: 'foo',
  parents: {
    mom: 'foomom',
    dad: 'foodad'
  }
}
```

After calling `defineReactive(vm._data)`, we got this:

![](https://i.imgur.com/TM5j5GL.jpg)

Give yourself some time to fully understand it.

Our next target is `Dep()`.

## Dep

Open `./dep.js`, you can see this class has only four methods.

```javascript
addSub (sub: Watcher) {
  this.subs.push(sub)
}

removeSub (sub: Watcher) {
  remove(this.subs, sub)
}

depend () {
  if (Dep.target) {
    Dep.target.addDep(this)
  }
}

notify () {
  // stabilize the subscriber list first
  const subs = this.subs.slice()
  for (let i = 0, l = subs.length; i < l; i++) {
    subs[i].update()
  }
}
```

`addSub()`, `removeSub()` and `notify()` deal with watchers. Each `Dep` instance has an array to store its watchers and tell them to `update()` during `notify()`. We have seen that `notify()` will be called in a setter, so if you change a reactive property, it will trigger watchers' updating.

`depend()` is strange, it first checks `Dep.target`, if it exists, call `Dep.target.addDep(this)`. What is `Dep.target`?

In the comments below this class, we can learn that `Dep.target` is globally unique. It's the watcher being evaluated now.

Next to it are two functions for stack operations. It's easy to understand if one watcher wants to get another watcher's value during evaluation, we need to store current target, switch to the new target and come back after it finishes.

`Dep.target` must be a watcher, so `Dep.target.addDep(this)` inside `depend()` tells us watcher has a method named `addDep()`. Its name implies that each watcher also has a list of `Dep`s it's watching.

Let's turn to watcher now.

## Watcher

Open `./watcher.js`, it's a little long but...hey, we are right, Watcher has the list to store it's `Dep`s.

The `constructor` simply initials some variables, set your computed function or watch expression to `this.getter` and try to get the value if it's not lazy.

Let's go on with `get()`, this is the only thing we get from `constructor()`.

```javascript
/**
 * Evaluate the getter, and re-collect dependencies.
 */
get () {
  pushTarget(this)
  let value
  const vm = this.vm
  if (this.user) {
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      handleError(e, vm, `getter for watcher "${this.expression}"`)
    }
  } else {
    value = this.getter.call(vm, vm)
  }
  // "touch" every property so they are all tracked as
  // dependencies for deep watching
  if (this.deep) {
    traverse(value)
  }
  popTarget()
  this.cleanupDeps()
  return value
}
```

Remember `Dep.target`? Here it calls `pushTarget()` and `popTarget()`, and do the evaluation between them!

Imagine we have a component like this:

```
{
  data: {
    name: 'foo'
  },
  computed: {
    newName () {
      return this.name + 'new!'
    }
  }
}
```

We know that `data` will be converted to reactive property, it's value, the object will be observed. If you get data use `this.foo` it will be proxied to `this._data['foo']`.

Now let's try to build a watcher step-by-step:

- assign our input function to getter
- call `this.get()`
- call `pushTarget(this)` which changes `Dep.target` to this watcher
- call `this.getter.call(vm, vm)`
- run `return this.foo + 'new!'`
- because `this.foo` is proxied to `this._data[foo]`, the reactive property `_data`'s getter is triggered
- inside the getter, it calls `dep.depend()`
- inside `depend()`, it calls `Dep.target.addDep(this)`, here `this` refers to the const `dep`, it's `_data`'s dep
- then it calls `childOb.dep.depend()` which add the dep of `childOb` to our target. Notice this time the `this` of `Dep.target.addDep(this)` refers to `childOb.__ob__.dep`
- inside `addDep()`, the watcher add this dep to it's `this.newDepIds` and `this.newDeps`
- because the default value of `this.depIds` is `[]`, the watcher calls `dep.addSub(this)`
- inside `addSub`, the dep add the watcher to it's `this.subs`
- now the watcher has gotten the value, it will `traverse()` the value to collect dependencies, calls `popTarget()` and `this.cleanupDeps()`

After this complex process, the watcher knows its dependencies, the dep knows its subscribers, the dynamic data net is built. With this net, Dep can `notify()` its subscribers when the reactive property gets the new value, which may trigger the `get()` again and refresh the value and relations.

And what `cleanupDeps` does? After reading the code, you can tell how it works to refresh the dependence relations.

---

![](http://i.imgur.com/5BRYgfi.jpg)

Above is the initialization of dynamic data net, this can help you understand the process.

If reactive property changes, it just triggers this process again to refresh computed property value and rebuild the dynamic data net.

## Next Step

Now we know how to build the dynamic data net. Next article will focus on three watcher updating ways and discuss how Vue keeps the correct updating order.

Read next chapter: [Dynamic Data - Lazy, Sync and Queue](https://github.com/numbbbbb/read-vue-source-code/blob/master/05-dynamic-data-lazy-sync-and-queue.md).

## Practice

Read the `cleanupDeps` method in `./watcher.js` and tell how this method updates the dependency during `get()` process.

Hint: the key is those two arrays: `this.newDepIds` and `this.depIds`. You may want to read `addDep()` first.


