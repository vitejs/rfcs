- Start Date: 2022-07-30
- Reference Issues: https://github.com/vitejs/vite/issues/5936
- Implementation PR: https://github.com/vitejs/vite/pull/9634

# Summary

Introduce capability for plugins to run [`writeBundle`](https://rollupjs.org/guide/en/#writebundle) and [`closeBundle`](https://rollupjs.org/guide/en/#closebundle) hooks sequentially. RFC targeting to Vite plugins, with the hope to be backported by Rollup.

# Basic example

`writeBundle` and `closeBundle` can be provided as functions, resulting in parallel execution to keep backward compatibility. With this RFC, it can optionally be an object to provide additional metadata, in this case, allowing `execution` to control the behavior of the hooks.

```ts
export default function myPlugin() {
  return {
    name: 'my-plugin',
    writeBundle: {
      sequential: true,
      async handler(options, bundle) {
        // ...
      }
    },
    closeBundle: {
      sequential: true,
      async handler() {
        // ...
      }
    }
  }
}
```

# Motivation

With the growth of Vite and its ecosystem, we have developed many more plugin usages which might go beyond Rollup/Vite's initial design scope (targeting Web, SSR, SSG, etc.). Inherit from Rollup, both [`writeBundle`](https://rollupjs.org/guide/en/#writebundle)
and [`closeBundle`](https://rollupjs.org/guide/en/#closebundle) are executed in parallel. This creates a racing condition where some plugins are not able to read the bundle processed by other plugins.

Related Issues:
- https://github.com/rollup/rollup/issues/2826 (Rollup: Add sequential before build hook)
- https://github.com/vitejs/vite/issues/5936
- https://github.com/vitejs/vite/pull/9326
- *please suggest more*

# Detailed design

While this pattern may be able to develop into a common convention applying to other hooks in the future. This RFC will focus on supporting `writeBundle` and `closeBundle` hooks.

## TypeScript Definition

`writeBundle` as an example:

```ts
type ExecutionMode = 'sequential' | 'parallel'
type WriteBundleHookFn = (options: Options, bundle: Bundle) => void | Promise<void>

interface WriteBundleHookOptions {
  execution?: ExecutionMode
  handle: WriteBundleHookFn
}

interface Plugin {
  // ...
  writeBundle?: WriteBundleHookOptions | WriteBundleHookFn
}
```

## Ordering

For simplicity, we might use the plugin level `enforce` option to control the order of execution. The question here is if Rollup team is willing to backport the `enforce` concept to Rollup.

## Execution

Leave the order to be defined normally by plugin position in the array, and each time a sequential execution hook is hit, then it is awaited. So they act as limits on what gets done in parallel

```
p1 p2 p3 sA p4 p5 sB
```

Will do `p1 p2 p3` in parallel, await that, then do `sA`, then `p4 p5` in parallel, then `sB`

Ordering should be respected as the sync part of each hook could expect that.

```js
let parallelHooks = []
for (const hook of hooks) {
  const h = normalizeHook(hook)
  if (h.execution === 'sequential')) {
    await Promise.all(parallelHooks)
    parallelHooks = []
    await h.handle(...)
  }
  else {
    parallelHooks.push(h.handle(...))
  }
}
await Promise.all(parallelHooks)
```

> Credit @patak-dev

# Drawbacks

Until Rollup backport it, this RFC will make:
- The hooks to be Vite specific
- Needs to do some hack outside Rollup to make it work

# Additional Context

In Vite, we do support a similar convention in [`transformIndexHtml`](https://vitejs.dev/guide/api-plugin.html#transformindexhtml) which accepts an object as the hook to specify the `enforce`. If this RFC gets accepted, we might take the chance to unify the convention for `transformIndexHtml` in another RFC.

# Alternatives

Introduce new separate hooks like `writeBundleSequential` and `closeBundleSequential`.

# Adoption strategy

This RFC should be additive and does not require migrations from plugin authors.
