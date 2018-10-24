# Find the Entry

This article belongs to the series [Read Vue Source Code](https://github.com/numbbbbb/read-vue-source-code).

In this article, we will:

- Get the code
- Open your editor
- Find the entry

## Get the Code

You can read code using GitHub, but it's slow and hard to see the directory structure. So let's download the code first.

Open [Vue's GitHub page](https://github.com/vuejs/vue), click "Clone or download" button and then click "Download ZIP".

![](http://i.imgur.com/Fshkk3Z.jpg)

You can also use `git clone git@github.com:vuejs/vue.git` if you like using console.

> I use the latest dev code I can get, so it may have some differences with the code you download.
> 
> Don't worry, this series is aimed at telling you **how** to understand the source code. You can use the same methods even if the code is different.
> 
> You can also download [the version I use](https://github.com/numbbbbb/read-vue-source-code/blob/master/vue-2.3.4.zip) if you like.

## Open Your Editor

I prefer to use Sublime Text, you can use your favorite editor.

Open your editor, double click the zip file you downloaded and drag the folder to your editor.

![](http://i.imgur.com/WgediMc.jpg)

## Find the Entry

Now we meet our first question: where should we start?

It's a common question for big open source projects. Vue is a npm package, so we can open `package.json` first.

```json
{
  "name": "vue",
  "version": "2.3.3",
  "description": "Reactive, component-oriented view layer for modern web interfaces.",
  "main": "dist/vue.runtime.common.js",
  "module": "dist/vue.runtime.esm.js",
  "unpkg": "dist/vue.js",
  "typings": "types/index.d.ts",
  "files": [
    "src",
    "dist/*.js",
    "types/*.d.ts"
  ],
  "scripts": {
    "dev": "rollup -w -c build/config.js --environment TARGET:web-full-dev",
    "dev:cjs": "rollup -w -c build/config.js --environment TARGET:web-runtime-cjs",
    "dev:esm": "rollup -w -c build/config.js --environment TARGET:web-runtime-esm",
    "dev:test": "karma start build/karma.dev.config.js",
    "dev:ssr": "rollup -w -c build/config.js --environment TARGET:web-server-renderer",
    ...
```

First three keys are `name`, `version` and `description`, no need to explain.

There is a `dist/` in `main`, `module` and `unpkg`, that implies they are related to generated files. Since we are looking for the entry, just ignore them.

Next is `typings`. After Googling, we know it's a TypeScript definition. We can see many type definitions in `types/index.d.ts`.

Go on.

`files` contains three paths. First one is `src`, it's the source code directory. This narrows the range, but we still don't know which file to start with.

Next key is `scripts`. Oh, the `dev` script! This command must know where to start.

```
rollup -w -c build/config.js --environment TARGET:web-full-dev
```

Now we have `build/config.js` and `TARGET:web-full-dev`. Open `build/config.js` and search `web-full-dev`:

```javascript
  // Runtime+compiler development build (Browser)
  'web-full-dev': {
    entry: resolve('web/runtime-with-compiler.js'),
    dest: resolve('dist/vue.js'),
    format: 'umd',
    env: 'development',
    alias: { he: './entity-decoder' },
    banner
  },
```

Great! 

This is the config of dev building. It's entry is `web/runtime-with-compiler.js`. But wait, where is the `web/` directory?

Check the entry value again, there is a `resolve`. Search `resolve`:

```javascript
const aliases = require('./alias')
const resolve = p => {
  const base = p.split('/')[0]
  if (aliases[base]) {
    return path.resolve(aliases[base], p.slice(base.length + 1))
  } else {
    return path.resolve(__dirname, '../', p)
  }
}
```


Go through `resolve('web/runtime-with-compiler.js')` with the parameters we have:

- p is `web/runtime-with-compiler.js`
- base is `web`
- convert the directory to `alias['web']`

Let's jump to `alias` to find `alias['web']`.

```javascript
module.exports = {
  vue: path.resolve(__dirname, '../src/platforms/web/runtime-with-compiler'),
  compiler: path.resolve(__dirname, '../src/compiler'),
  core: path.resolve(__dirname, '../src/core'),
  shared: path.resolve(__dirname, '../src/shared'),
  web: path.resolve(__dirname, '../src/platforms/web'),
  weex: path.resolve(__dirname, '../src/platforms/weex'),
  server: path.resolve(__dirname, '../src/server'),
  entries: path.resolve(__dirname, '../src/entries'),
  sfc: path.resolve(__dirname, '../src/sfc')
}
```

Okay, it's `src/platforms/web`. Concat it with the input file name, we get `src/platforms/web/runtime-with-compiler.js`.

```javascript
/* @flow */

import config from 'core/config'
import { warn, cached } from 'core/util/index'
import { mark, measure } from 'core/util/perf'

import Vue from './runtime/index'
import { query } from './util/index'
import { shouldDecodeNewlines } from './util/compat'
import { compileToFunctions } from './compiler/index'

const idToTemplate = cached(id => {
  const el = query(id)
  return el && el.innerHTML
})

const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)

  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
      `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
    )
    return this
  }
  ...
```

Cool! You have found the entry!

> Have you noticed the `/* flow */` comment at the top? After Google, we know it's a type checker. It reminds me of `typings`, I search again and find out that `flow` can use `typings` definitions to validate your code.

> Maybe next time I can use `flow` and `typings` in my own project. You see, even if we haven't really started reading the code, we have learned some useful and practical things.

Let's go through this file step-by-step:

- import config
- import some util functions
- import Vue(What? Another Vue?)
- define `idToTemplate`
- define `getOuterHTML`
- define `Vue.prototype.$mount` which use `idTotemplate` and `getOuterHTML`
- define `Vue.compile`

The two important points are:

1. This is **NOT** the real Vue code, we should know that from the filename, it's just an entry
2. This file extracts the `$mount` function and defines a new `$mount`. After reading the new definition, we know it just add some validations before calling the real mount

## Next Step

Now you know how to find the entry of a brand new project. In next article, we will go on to find the core code of Vue.

Read next chapter: [Dig into the Core](https://github.com/numbbbbb/read-vue-source-code/blob/master/02-dig-into-the-core.md).

## Practice

Remember those "util functions"? Read their codes and tell what they do. It's not difficult, but you need to be patient. 

Don't miss `shouldDecodeNewlines`, you can see how they fight with IE.


