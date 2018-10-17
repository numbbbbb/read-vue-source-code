# View Render - Patch

This article belongs to the series [Read Vue Source Code](https://github.com/numbbbbb/read-vue-source-code).

In this article, we will learn:

- What `render()` returns
- How `__patch__` updates your webpage

![](http://i.imgur.com/9M2VX5F.jpg)

This article will focus on `_update()` part.

## What `render()` Returns

In [previous article](https://github.com/numbbbbb/read-vue-source-code/blob/master/07-view-render-compiler.md) we have generated the final `render` function, now let's run it and see what is returned:

![](http://i.imgur.com/F91DLpO.jpg)

Modify `core/instance/render.js`, add `console.log` and run `npm run build` to generate the whole Vue bundle.

After building, copy all JS files from `dist/` to your project's `node_modules/vue/dist/` and build your project.

We use the demo project of the last article again, open it in the browser, open console, you can see.

![](http://i.imgur.com/KOk5LgP.jpg)

There are two VNodes. 

First is the root VNode, it's `child` property refers to the component(click to expand it, you'll see many familiar properties, like `_data`, `_watchers`, `_events` etc).

Second is the `div` VNode, its `parent` property refers to the first VNode. The `children` array contains our text nodes and span nodes.

Notice there is a `context` property in the second VNode, it refers to the component and is used during render process to get values.

With this clear structure and data in the console, you can understand the implementation of `render()` easily. If you are not interested in details, just remember `render()` gives you the VNodes, with all data injected.

## How `__patch__()` Updates Your Webpage

Recall this function in `mountComponent`.

![](http://i.imgur.com/wfyGaHG.jpg)

After executing `vm_render()`, we can go into `vm._update()`.

![](http://i.imgur.com/NAnKb2Z.jpg)

`vm.__patch__()` is the key part. If this is a new node, it will initial the DOM, otherwise, it will update the DOM. Both are implemented inside `vm.__patch__()` with the VNodes we got from `render()`.

Now use the skills you learn from previous articles to find the definition of `__patch__()`. It's located in `platforms/web/runtime/patch.js`, created by `createPatchFunction({ nodeOps, modules })`.

Trace `nodeOps` and `modules`, you can find `nodeOps` are real DOM operations, like this:

![](http://i.imgur.com/IYWU0vC.jpg)

`modules` are operations related to DOM node, like `setAttr`, `updateClass`, `updateStyle`.

Here the important thing is that `__patch__()` is dynamic and platform specific. No need to explain, it's the same pattern we have saw in Vue's core implementation.

Up to now, we have looked through `_render()`, `parser`, `optimizer`, `generater`, `_update()`, `__patch__()`, `nodeOps` and `modules`. Is that all about render process? Absolutely not, we missed an important part.

## How to Update DOM FAST

We have old VNodes, new VNodes, and tools to modify DOM, but how to update it fast? 

Or you can ask from another side: how to make Vue faster than other frameworks? DOM operation is the most time-consuming part, so in order to beat other frameworks, Vue must have some algorithms here to speed it up.

And yes, it has.

Open `core/vdom/patch.js`, read the top comments, we learn Vue's DOM patching algorithm is implemented based on Snabbdom.

After reading `createPatchFunction()` and `patch()` inside it, we find that `patch()` can do both `mount()` and `update()`. `mount()` is easy, just generate the DOM based on VNodes, we should focus on `update()`.

The core function of `update` is `patchVnode()`.

![](http://i.imgur.com/CKxi6L6.jpg)

Here is the real patch part.

The outmost `if` clause checks if vnode has text. If it has, then it must be a leaf node, and if the text changes, just call `nodeOps.setTextContent()`.

If vnode doesn't have text, it means we have to deal with its children, go into the outmost `if` clause.

Here we meet four `if-else` clauses:

- if the old node and new node both have children and they are not equal: call `updateChildren()`
- if only the new node has children: if old node has text, remove the text; call `addVnodes()` to add new node's children
- if only old node has children: it means new node is empty, just call `removeVnodes()` to remove the old node
- if old node and new node both doesn't have children AND old node has text: if you go into this clause, it means new node doesn't has text(otherwise the outside if will fall into the `else` clause), so just call `setTextContent()` to remove the text

Feel free to pause and think for a while before going on.

Next go to `updateChildren()`. It seems scared long, but don't worry, it's not such difficult to understand.

Now pick up a pencil and a piece of paper.

First, we have two arrays, `oldCh` and `Ch`:

![](http://i.imgur.com/4yanODV.jpg)

Each blue and green block represents a VNode in the array.

Then add variables.

![](http://i.imgur.com/EfdVdaU.jpg)

Okay, now simulate the execution in your mind.

- `while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {`, read from your paper, yes it's `true`.
- `if (isUndef(oldStartVnode)) {`, here you can fill in different Vnode to see how this function works, I want my old start Vnode and old end Vnode to be both defined
- `} else if (sameVnode(oldStartVnode, newStartVnode)) {`, here check whether old start Vnode is new start Vnode. Umm, let's try `true` first. Then it calls our familiar function `patchVnode` to update the DOM and, hey it updates variables

![](http://i.imgur.com/88FigaR.jpg)

We have updated those two start Vnodes! Okay, go back to `while` and go on with your paper.

I won't list all possibilities here, you can play it as long as you like until you really understand `updateChildren`.

In my opinion, this algorithm is not complex. The core thought is "reuse". Only create new Vnodes when all other methods are failed. Updating is easier and faster than creating and inserting.

## Next Step

Congratulations! You have walked though almost all important parts of Vue. Entry, initialization process, observer, watcher, dep, parser, optimizer, generator, Vnode, patch. You know the initialization order, the way to build dynamic data net, how template is compiled to a function and how to patch the DOM efficiently.

So what's next? Check it out by yourself.

Read next chapter: [Conclusion](https://github.com/numbbbbb/read-vue-source-code/blob/master/09-conclusion.md).

## Practice

Continue simulating the execution of `updateChildren()` until you really understand it.

What would you implement the update operation? Compare it to `updateChildren()` and see why Vue is faster.


