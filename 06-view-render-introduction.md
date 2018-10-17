# View Render - Introduction

This article belongs to the series [Read Vue Source Code](https://github.com/numbbbbb/read-vue-source-code).

In this article, we will learn:

- How to find render functions
- The structure of view render

## Find Render Functions

After understanding the initialization and data updating, now we turn to the view render part to see how Vue converts our data to the real DOM nodes.

> Notice: the word "render" refers to the whole view rendering process, while the formatted `render()` or `_render()` refers to the real function.

First of all, we need to find functions related to render process.

In [previous article](https://github.com/numbbbbb/read-vue-source-code/blob/master/05-dynamic-data-lazy-sync-and-queue.md), we have learned that `mountComponent()` will use `_update()` and `_render()` to update views. Let's do a global search to find `_update`.

![](http://i.imgur.com/0YMdDxG.jpg)

`_update()` uses `__patch__()` to calculate which part needs updating and manipulate corresponding DOMs. We will talk about `__patch__()` later.

Go back to `mountComponent()`, our next target is `_render()`.

We have seen `_render()` is defined in `core/instance/render.js`, let's look at it again.

![](http://i.imgur.com/FPn36ty.jpg)

Inside `_render()`, it calls `render()` which is extracted from `vm.$options`.

After global searching `$option`, we find:

```
vm.$options = mergeOptions(
  resolveConstructorOptions(vm.constructor),
  options || {},
  vm
)
```

`options` is the parameter passed in when you call `new Vue({...})`, so `render()` must comes from `vm.constructor`. `vm` is the instance of Vue, so we try to global search `options.render`, maybe we can find where it's set.

Look through the search results, this one set a value.

![](http://i.imgur.com/RvT8WgO.jpg)

Double click it.

![](http://i.imgur.com/wVMfCcr.jpg)

Cool! This is the outmost wrapper of `$mount()`, it calls `compileToFunctions()` with your `template`, gets `render` and `staticRenderFns` as return value and set it to `options.render`.

Go on, `compileToFunctions()` comes from `./compiler/index.js`:

```
const { compile, compileToFunctions } = createCompiler(baseOptions)

export { compile, compileToFunctions }
```

`createCompiler()` comes from `compiler/index.js`:

![](http://i.imgur.com/K8QIUHf.jpg)

`createCompilerCreator()`, good name! It's a high order function which creates the `createCompiler()` function which is used to create the compiler. Later our template will be compiled using this compiler.

After reading the code of `createCompilerCreator()`, we learn that it's just a wrapper. The core function is `baseCompile()` which uses `parse()`, `optimize()` and `generate()` to do the real jobs.

Let's make it clear:

- first, you choose your own parser, optimizer and codegen, use them to build the core compile function(or use the default one)
- then, pass your compile function to `createCompilerCreator()`, it will return a function which can be used to create the compiler, thus its name is `createCompiler()`
- next, call `createCompiler()` with options, it will return the real compiler
- finally, use that compiler to compile the input template

The purpose of this complicated process is extracting the core compiler and options. If you treat **core compiler function** and **options** as two parameters for `createCompiler()`, you can see it's similar with currying. Two parameters come at different time, so in `createCompilerCreator()` it has to create a new function to store **core compiler function** and return that function(this function is `createCompiler()`). When **options** is passed in, `createCompiler()` combines them and create the final compiler.

Now we have found all functions related to render process: `_update()`, `__patch__()`, `_render()`, `createCompiler()`, `parse()`, `optimize()`, `generate()`. Let's organize them.

## The Structure of Render

Render process starts when `mountComponent()` is called. It will call `_render()` to compile template into `render` and `staticRenderFns`, then pass them to `_update()` to calculate the operations and apply them to DOMs.

![](http://i.imgur.com/NM77eiy.jpg)

## Next Step

Next two articles will talk about **compiler** and **patch**, you will see how VDom is designed and how to do the diff calculation fastly.

Read next chapter: [View Rendering - Compiler](https://github.com/numbbbbb/read-vue-source-code/blob/master/07-view-render-compiler.md).

## Practice

```
vnode = render.call(vm._renderProxy, vm.$createElement)
```

This line is located in `_render()`, try to figure out what is `vm._renderProxy` and it's role.




