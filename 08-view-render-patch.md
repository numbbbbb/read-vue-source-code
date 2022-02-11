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

```javascript
updateComponent = () => {
  vm._update(vm._render(), hydrating)
}
```

After executing `vm._render()`, we can go into `vm._update()`.

```javascript
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  if (vm._isMounted) {
    callHook(vm, 'beforeUpdate')
  }
  const prevEl = vm.$el
  const prevVnode = vm._vnode
  const prevActiveInstance = activeInstance
  activeInstance = vm
  vm._vnode = vnode
  // Vue.prototype.__patch__ is injected in entry points
  // based on the rendering backend used.
  if (!prevVnode) {
    // initial render
    vm.$el = vm.__patch__(
      vm.$el, vnode, hydrating, false /* removeOnly */,
      vm.$options._parentElm,
      vm.$options._refElm
    )
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  activeInstance = prevActiveInstance
  // update __vue__ reference
  if (prevEl) {
    prevEl.__vue__ = null
  }
  if (vm.$el) {
    vm.$el.__vue__ = vm
  }
  // if parent is an HOC, update its $el as well
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el
  }
  // updated hook is called by the scheduler to ensure that children are
  // updated in a parent's updated hook.
}
```

`vm.__patch__()` is the key part. If this is a new node, it will initialize the DOM, otherwise, it will update the DOM. Both are implemented inside `vm.__patch__()` with the VNodes we got from `render()`.w43e

Now use the skills you learn from previous articles to find the definition of `__patch__()`. It's located in `platforms/web/runtime/patch.js`, created by `createPatchFunction({ nodeOps, modules })`.

Trace `nodeOps` and `modules`, you can find `nodeOps` are real DOM operations, like these:

```javascript
export function createElementNS (namespace: string, tagName: string): Element {
  return document.createElementNS(namespaceMap[namespace], tagName)
}

export function createTextNode (text: string): Text {
  return document.createTextNode(text)
}

export function createComment (text: string): Comment {
  return document.createComment(text)
}

export function insertBefore (parentNode: Node, newNode: Node, referenceNode: Node) {
  parentNode.insertBefore(newNode, referenceNode)
}

export function removeChild (node: Node, child: Node) {
  node.removeChild(child)
}

export function appendChild (node: Node, child: Node) {
  node.appendChild(child)
}
```

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

Implement of `patchVnode()`:

```javascript
function patchVnode (oldVnode, vnode, insertedVnodeQueue, removeOnly) {
  if (oldVnode === vnode) {
    return
  }
  // reuse element for static trees.
  // note we only do this if the vnode is cloned -
  // if the new node is not cloned it means the render functions have been
  // reset by the hot-reload-api and we need to do a proper re-render.
  if (isTrue(vnode.isStatic) &&
    isTrue(oldVnode.isStatic) &&
    vnode.key === oldVnode.key &&
    (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
  ) {
    vnode.elm = oldVnode.elm
    vnode.componentInstance = oldVnode.componentInstance
    return
  }
  let i
  const data = vnode.data
  if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
    i(oldVnode, vnode)
  }
  const elm = vnode.elm = oldVnode.elm
  const oldCh = oldVnode.children
  const ch = vnode.children
  if (isDef(data) && isPatchable(vnode)) {
    for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
    if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
  }
  if (isUndef(vnode.text)) {
    if (isDef(oldCh) && isDef(ch)) {
      if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
    } else if (isDef(ch)) {
      if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
      addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
    } else if (isDef(oldCh)) {
      removeVnodes(elm, oldCh, 0, oldCh.length - 1)
    } else if (isDef(oldVnode.text)) {
      nodeOps.setTextContent(elm, '')
    }
  } else if (oldVnode.text !== vnode.text) {
    nodeOps.setTextContent(elm, vnode.text)
  }
  if (isDef(data)) {
    if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
  }
}
```

Here is the real patch part.

The outmost `if` clause checks if vnode has text. If it has, then it must be a leaf node, and if the text changes, just call `nodeOps.setTextContent()`.

If vnode doesn't have text, it means we have to deal with its children, go into the outmost `if` clause.

Here we meet four `if-else` clauses:

- if the old node and new node both have children and they are not equal: call `updateChildren()`
- if only the new node has children: if old node has text, remove the text; call `addVnodes()` to add new node's children
- if only old node has children: it means new node is empty, just call `removeVnodes()` to remove the old node
- if old node and new node both dont't have children AND old node has text: if you go into this clause, it means new node doesn't has text(otherwise the outside if will fall into the `else` clause), so just call `setTextContent()` to remove the text

Feel free to pause and think for a while before going on.

Next go to `updateChildren()`. It seems very intimidating, but don't worry, it's not such difficult to understand.

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


