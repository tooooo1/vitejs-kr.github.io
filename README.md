# vite

> No-bundle Dev Server for Vue 3 Single-File Components

**⚠️ Warning: Experimental ⚠️**

Create the following files:

**index.html**

```html
<div id="app"></div>
<script type="module">
  import { createApp } from 'vue'
  import Comp from './Comp.vue'

  createApp(Comp).mount('#app')
</script>
```

**Comp.vue**

```vue
<template>
  <button @click="count++">{{ count }}</button>
</template>

<script>
export default {
  data: () => ({ count: 0 })
}
</script>

<style scoped>
button {
  color: red;
}
</style>
```

Then run:

```bash
npx vite
```

`npx` will automatically install `vite` to `npm`'s global cache before running it. Go to `http://localhost:3000`, edit the `.vue` file to see changes hot-updated instantly.

## Local Installation

Alternatively, you can install `vite` locally as a dev dependency and run it via npm scripts:

```bash
npm install -D vite
# OR
yarn add -D vite
```

Add scripts to `package.json` (here showing that serving the files in `src` instead of project root):

```json
{
  "scripts": {
    "dev": "vite"
  }
}
```

```bash
npm run dev
# OR
yarn dev
```

If you are placing your files in a sub-directory, you can also ask `vite` to serve a different directory with `vite --root some-dir`.

## How It Works

Imports are requested by the browser as native ES module imports - there's no bundling. The server intercepts requests to `*.vue` files, compiles them on the fly, and sends them back as JavaScript.

## Bare Module Resolving

Native ES imports doesn't support bare module imports like

```js
import { createApp } from 'vue'
```

The above will throw an error by default. `vite` detects such bare module imports in all served `.js` files and rewrite them with special paths like `/@modules/vue`. Under these special paths, `vite` performs module resolution to locate the correct files on disk:

- `vue` has special handling: you don't need to install it since `vite` will serve it by default. But if you want to use a specific version of `vue` (only supports Vue 3.x), you can install `vue` locally into `node_modules` and it will be preferred (`@vue/compiler-sfc` of the same version will also need to be installed).

- If a `web_modules` directory (generated by [Snowpack](https://www.snowpack.dev/))is present, we will try to locate the module it.

- Finally we will try resolving the module from `node_modules`, using the package's `module` entry if available.

## Hot Module Replacement

- `*.vue` files come with HMR out of the box.

- For `*.js` files, a simple HMR API is provided:

  ```js
  import { foo } from './foo.js'
  import { hot } from '/@hmr'

  foo()

  hot.accept('./foo.js', ({ foo }) => {
    // the callback recevies the updated './foo.js' module
    foo()
  })
  ```

  Note it's simplified and not fully compatible with webpack's HMR API, for example there is no self-accepting modules, and if you re-export `foo` from this file, it won't reflect changes in modules that import this file.

## CSS Pre-Processors

Install the corresponding pre-processor and just use it! (**Currently requires local installation of `vite` for correct resolution**).

``` bash
yarn add -D sass
```
``` vue
<style lang="scss">
/* use scss */
</style>
```

## API

You can customize the server using the API. The server can accept plugins which have access to the internal Koa app instance. You can then add custom Koa middlewares to add pre-processor support:

``` js
const { createServer } = require('vite')

const myPlugin = ({
  root, // project root directory, absolute path
  app, // Koa app instance
  server, // raw http server instance
  watcher // chokidar file watcher instance
}) => {
  app.use(async (ctx, next) => {
    // You can do pre-processing here - this will be the raw incoming requests
    // before vite touches it.
    if (ctx.path.endsWith('.scss')) {
      // Note vue <style lang="xxx"> are supported by
      // default as long as the corresponding pre-processor is installed, so this
      // only applies to <link ref="stylesheet" href="*.scss"> or js imports like
      // `import '*.scss'`.
      console.log('pre processing: ', ctx.url)
      ctx.type = 'css'
      ctx.body = 'body { border: 1px solid red }'
    }

    // ...wait for vite to do built-in transforms
    await next()

    // Post processing before the content is served. Note this includes parts
    // compiled from `*.vue` files, where <template> and <script> are served as
    // `application/javascript` and <style> are served as `text/css`.
    if (ctx.response.is('js')) {
      console.log('post processing: ', ctx.url)
      console.log(ctx.body) // can be string or Readable stream
    }
  })
}

createServer({
  plugins: [
    myPlugin
  ]
}).listen(3000)
```

## Building for Production

Starting with version `^0.5.0`, you can run `vite build` to bundle the app and deploy it for production.

- `vite build --root dir`: build files in the target directory instead of current working directory.

- `vite build --cdn`: import `vue` from a CDN link in the built js. This will make the build faster, but overall the page payload will be larger because therer will be no tree-shaking for Vue APIs.

Internally, we use a highly opinionated Rollup config to generate the build. There is currently intentionally no exposed way to configure the build -- we will likely tackle that at a later stage.

## TODOs

- Vue SFC Source Map support
- Custom imports map (alias) support
- Auto loading postcss config
