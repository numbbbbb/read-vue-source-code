# Dig into the Core

This article belongs to the series [Read Vue Source Code](https://github.com/numbbbbb/read-vue-source-code).

In this article, we will:

- Find the real Vue implementation


## Find the Real Vue Code

We have found the entry at [previous article](https://github.com/numbbbbb/read-vue-source-code/blob/master/01-find-the-entry.md) with the clue `import Vue from './runtime/index'`, let's go along that.

Open `./runtime/index`.

![](http://i.imgur.com/SNWZ60U.jpg)

`import Vue` again! Let's look through this file step-by-step:

- import config
- import util funcions
- import patch, mountComponent
- import directives and components
- install platform specific utils
- install platform runtime directives & components
- install platform patch function
- define public mount method
- console log the message of Vue Devtools and development mode warning(now you know the origin of your console logs)

As you can see, this file just adds some platform specific things to Vue.

Two important points here:

1. `Vue.prototype.__patch__ = inBrowser ? patch : noop`, we will talk about patch in later articles, it's responsibility is updating the webpage, which means it will manipulate the DOM
2. `mount` again, so the core implementation of `mountComponent` is encapsulated twice

After checking the Vue directory, we can find there is another platform `weex`. It's a framework similar to ReactNative and it's maintained by Alibaba.

Go on with our new clue `import Vue from 'core/index'`.

![](http://i.imgur.com/qeY5n53.jpg)

`function Vue (options) {`! We made it! This is the core Vue implementation.

Below are five mixins, their names imply what they are.

But why the core Vue function is such short? Only a `this._init`?

Init process will be introduced in next article, now let's talk about how Vue is designed.

Vue is a big open source project, which means it must be divided into many layers and parts. Let's start from the core and reverse the path to see how Vue is organized:

- Core: the origin Vue function, calls `this._init()`
- Mixins: five mixins, add init, state, events, lifecycle and render functions to core
- Platform: add platform specific things to core, add patch and public mount functions
- Entry: add configs and outmost $mount function to core

The entire Vue function is composited after four steps.

![](http://i.imgur.com/cpz3Izw.jpg)

This multiple layers pattern has many advantages:

1. Decouple. Different layers focus on **different** things
2. Encapsulate. Each layer **only** focus on itself
2. Reuse. Closer to core, more **generic**. This makes Vue easy to compatible with different platforms and easy to build for different environment

This layer pattern reminds me of OSI layer model, maybe Vue's author is inspired by that?

## Next Step

Vue's core function just calls `this._init()`. What does it do? We will reveal it in next article.

## Practice

Find a friend and try to explain Vue's layer pattern to him/her.

Nothing is perfect. Can you tell some disadvantages of this pattern?


