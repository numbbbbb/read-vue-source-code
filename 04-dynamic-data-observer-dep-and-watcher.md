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

In the previous article, we have seen `defineReactive` which is used to make a value `reactive`. Let's see its usage in `defineReactive()`.

![](http://i.imgur.com/1qHoCtG.jpg)

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

![](http://i.imgur.com/p1TKC2S.jpg)

`observe()` will extract the exist observer or create a new one with `new Observer(value)`. Notice that observe only works for an object, primitive value won't be observed.

If this value is used as root data, it will increments `ob.vmCount++`, we have talked about that in init process.

Okay, now we have got or created the watcher. Next, `dependArray()`.

![](http://i.imgur.com/85sa8Gz.jpg)

It just iterates the array recursively and calls `e.__ob__.dep.depend()` which leads us to `depend()` again.

So now we have found the usage of `Dep()`, `Observer()`, `Watcher()`. And `dep.depend()`, `dep.notify()`.

If you use `defineReactive()` to convert a value, that reactive value has one `dep` and one `childOb` set by `observe(val)` if the value is object.

Let's read `Observer()` now.

![](http://i.imgur.com/YHSDSec.jpg)

It first defines a `__ob__` property to the value you pass in. 

Then if the value is an array, it will intercept array methods(like `push`, `pop`) to make sure Vue can detect array manipulation. After that, it calls `observeArray()` which will iterates items and call `observe()`.

If the value is not an array, this function just walks through all keys and use `defineReactive()` to convert all values into a reactive property.

As you can see, `defineReactive()` calls `new Observer()`, `Observer()` may also call `defineReactive()`. Thus, when you want to convert a value with `defineReactive()`, it will recursively converts all sub values into reactive property.

In short, `Observer` will add a `dep` to the value and all it's sub values while wrapping their getter and setter to trigger dep.

Our next target is `Dep()`.

## Dep

Open `./dep.js`, you can see this class has only four methods.

![](http://i.imgur.com/hEoe7In.jpg)

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

![](http://i.imgur.com/8bgITCW.jpg)

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

We know that the value of `data` will be converted to reactive property, and if you get data use `this.name` it will be proxied to `this._data['name']`.

Now let's try to build a watcher step-by-step:

- assign our input function to getter
- call `this.get()`
- call `pushTarget(this)` which changes `Dep.target` to this watcher
- call `this.getter.call(vm, vm)`
- run `return this.name + 'new!'`
- because `this.name` is proxied to `this._data[name]`, the reactive value `_data`'s getter is triggered
- inside the getter, it calls `dep.depend()`(no `childOd` because 'name' is a primitive value)
- inside `depend()`, it calls `Dep.target.addDep(this)`, here `this` is `_data`'s dep, it's stored at `vm._data.__ob__.dep`
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

## Practice

Read the `cleanupDeps` method in `./watcher.js` and tell how this method updates the dependency during `get()` process.

Hint: the key is those two arrays: `this.newDepIds` and `this.depIds`. You may want to read `addDep()` first.


