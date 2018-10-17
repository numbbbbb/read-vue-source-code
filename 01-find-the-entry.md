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

![](http://i.imgur.com/YFW6HYF.jpg)

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

![](http://i.imgur.com/prmDkBG.jpg)

Great! 

This is the config of dev building. It's entry is `web/entry-runtime-with-compiler.js`. But wait, where is the `web/` directory?

Check the entry value again, there is a `resolve`. Search `resolve`:

![](http://i.imgur.com/8OlC2XC.jpg)


Go through `resolve('web/entry-runtime-with-compiler.js')` with the parameters we have:

- p is `web/entry-runtime-with-compiler.js`
- base is `web`
- convert the directory to `alias['web']`

Let's jump to `alias` to find `alias['web']`.

![](http://i.imgur.com/RGG2thw.jpg)

Okay, it's `src/platforms/web`. Concat it with the input file name, we get `src/platforms/web/entry-runtime-with-compiler.js`.

![](http://i.imgur.com/EgfmC7A.jpg)

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


