# Dynamic Data - Lazy, Sync and Queue

This article belongs to the series [Read Vue Source Code](https://github.com/numbbbbb/read-vue-source-code).

In this article, we will see:

- Update Watcher in three ways
- How to trigger view updating
- How to keep the updating order

## Update Watcher in Three Ways

In [previous article](https://github.com/numbbbbb/read-vue-source-code/blob/master/04-dynamic-data-observer-dep-and-watcher.md) we have learned how Vue builds dynamic data net with Observer, Dep and Watcher. But that's just a glance, there are several important things we need to talk.

Go back to `./watcher.js` again.

In [previous article](https://github.com/numbbbbb/read-vue-source-code/blob/master/04-dynamic-data-observer-dep-and-watcher.md) we just simulate the init process, now let's talk about updating.

Remember that when you update a reactive property, the setter will be called and it will call `dep.notify()` which calls `update()` of its subscribers(they are Watchers).

So we can go directly to `update()`.

![](http://i.imgur.com/4U2J8Ue.jpg)

This `if-else` statement has three clauses, let's look through them one by one.

### Lazy

If this watcher is lazy according to the options you pass in during initialization, it just marks itself dirty.

Let's find out where `dirty` is used.

Search `dirty`, you got:

![](http://i.imgur.com/WY4aCZF.jpg)

When `evaluate()` is called, it calls `this.get()` to get the real value and set `dirty` to `false`. Where is `evaluate()` called? Let's do a global search.

If you use Sublime Text, right click on `src` directory and choose `Find in Folder...`.

![](http://i.imgur.com/sO4k7GQ.jpg)

Then enter `evaluate` and click `Find`.

![](http://i.imgur.com/0qtiIU8.jpg)

The first result calls `watcher.evaluate()`, double click that line to jump to that file.

![](http://i.imgur.com/qcP4R3w.jpg)

Got it. When computed property's getter is called, if the watcher is dirty, it will do the evaluation. Use lazy mode can put off the evaluation until you really need the value.

### Sync

Go back to our second `if-else` clause.

![](http://i.imgur.com/4U2J8Ue.jpg)

If this watcher's `sync` is `true`, it will call `this.run()`. Search `run`.

![](http://i.imgur.com/BEHuDQM.jpg)

This function calls `this.get()`. If the value changes or it's an object or this watcher is deep, the old value will be replaced and the callback function will be called.

In [previous article](https://github.com/numbbbbb/read-vue-source-code/blob/master/04-dynamic-data-observer-dep-and-watcher.md) we have learned how `this.get()` works, you can read it again if you forget.

Sync mode is easy to understand, but unfortunately, it's `false` by default. The most frequently used mode is Async.

### Queue

![](http://i.imgur.com/4U2J8Ue.jpg)

If your watcher is neither lazy nor sync, the execution will flow to `queueWatcher(this)`.

![](http://i.imgur.com/ANzbFqj.jpg)

If the queue isn't flushing now, it simply pushes the watcher into the queue.

If the queue is flushing, it will find the right position of this watcher based on its id. 

Finally, if we are not waiting, calls `flushSchedulerQueue()` at nextTick.

Here we meet two flags: `flushing` and `waiting`. Seems they are very similar, why should we use two flags?

We can do a reverse think. What if we only have `flushing`? 

`flushing` will be set to `true` when `flushSchedulerQueue()` is executed. Oh, notice that `flushSchedulerQueue()` is called with `nextTick()`, thus it won't be executed now. If we call `queueWatcher()` multiple times, there will be duplicated `flushSchedulerQueue()` at nextTick!

That's it. `flushing` marks whether the tasks in the queue is executing, `waiting` marks whether the flush operation is placed at nextTick.

## How to Trigger View Updating

Now we know how watchers update their value, but hey, watchers are used for computed properties and watch callbacks, how do our views update when reactive properties change?

There are no reasons for Vue to implement another dynamic data process, it should reuse Watcher for view updating. But we haven't seen any watchers created for view updating.

Let's use global searching again. What's the keyword? Remember we have met `_update` and `_render` in init process, let's try `_update` first.

![](http://i.imgur.com/SCB7qkC.jpg)

Seems the `updateComponent` in the first result is what we need, double click it.

![](http://i.imgur.com/2V83kfm.jpg)

Here it is! We are right, Vue creates a Watcher for `updateComponent`. These lines are inside `mountComponent`, and `mountComponent` is the core of `$mount`. So after initializing the component, Vue will call `$mount` and inside it, the Watcher is created.

When a new Watcher is created, it's `lazy` is `false` by default, so at the end of the constructor, it will call `get()` and build the whole dynamic data net.

Notice that `updateComponent` is the second parameter, and it will become the `getter` of the Watcher. So when Vue tries to get this Watcher's value, it will update the view. If it's hard to understand, you can treat this getter as a wrapper of the real getter, it updates the view after calling the real getter(the `vm._render()`).

A little tricky, but it works well.

## How to Keep The Updating Order

After learning initialization and data updating process, we can try to solve a complicated problem.

How to make sure all data and views update in the correct order?

Let's see a small example:

```
<div id="app">
  {{ newName }}
</div>

var app = new Vue({
  el: '#app',
  data: {
    name: 'foo'
  },
  computed: {
    newName () {
      return this.name + 'new!'
    }
  }
})
```

In this example, we have one property, one computed property. And we display the computed property in view.

After initialization, we have one reactive property and two watchers subscribe to it. Notice the view doesn't subscribe to the computed property because the computed property is a watcher, not a reactive property.

Now we change the value of `name`, we know that the computed property and view will both update. But would them update in the correct order? If view updates first, it will show the old computed property value.

How Vue solves this problem?

Let's simulate the updating process.

When `name` changes, it will call `dep.notify()`. `notify()` will iterate it's subscriber array and call their `update()`. By default, the watcher's `lazy` and `sync` are both `false`, so two tasks will be pushed to queue.

Okay, the key is the task order.

Read `flushSchedulerQueue` again, we can find there are a sort call and some comments. Before running all tasks, the queue will sort them based on their `id`. Recall that `$mount` is the last operation during initialization, we know that the computed watcher is created before rendering watcher, so it's `id` is smaller, thus it's executed early.

> Queue introduces a new problem: if you use the queue and read computed property right after changing the data it depends, you will get old value. However, after global searching, I found that `sync` is always `true`, so seems the queue is never used.

You see, the updating order is set based on initialization order, now you know why we have to learn init process first.

You may ask, what would happen if the computed watcher is `lazy`? I will leave this to you.

## Next Step

We have learned three updating ways and how to keep the correct updating order. But these all happen "inside", how does Vue apply the updating to DOM? How to convert your `.vue` files into browser executable code? Next several articles will talk about the entire render process.

## Practice

Try to tell how Vue keeps the correct updating order if the computed watcher is `lazy`.

Hint: simulate the updating process by yourself.


