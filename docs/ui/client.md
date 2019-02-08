# client 端

client 端可以看作是一个 vue 项目，它是通过 vue 组件构成，在 `ui` 方面使用了自家的 [@vue/ui](https://www.npmjs.com/package/@vue/ui)，由于 server 端采用了 graphql，因此 client 端使用了
[vue-apollo](https://github.com/Akryum/vue-apollo)，这样可以利用其提供的 API 和组件高效方便地使用 graphql 进行查询，变更以及订阅。

client 端的内容非常多，从项目创建到项目配置、运行以及打包发布，涉及到的代码很多，但大部分的流程基本上一直，这就不会一一做分析了，会选择导入项目这部分来分析，因为 ui 命令也是基于插件机制的，而导入项目的时候会涉及到插件加载，利用 PluginAPI 增强项目的配置和任务，以及分享数据和在进程间进行通信，下面就开始分析项目导入的整个过程。

::: tip 源码目录
`vue ui` 运行的客户端是要打包压缩过的代码，目录为 `@vue/cli-ui/dist`，通过以下代码设置了静态资源（文件）目录，访问 `localhost:8000` 则指向
`@vue/cli-ui/dist/index.html`,从而启动了 client 端，对应的源码目录为 `@vue/cli-ui/src`。
``` js
app.use(express.static(distPath, { maxAge: 0 }))
```
:::

## importProject

导入项目的组件为 `@vue/cli-ui/src/components/project-manager/ProjectSelect.vue`，部分代码如下

``` html
<div class="actions-bar center">
  <VueButton
    icon-left="unarchive"
    :label="$route.query.action || $t('org.vue.views.project-select.buttons.import')"
    class="big primary import-project"
    :disabled="folderCurrent && !folderCurrent.isPackage"
    :loading="busy"
    @click="importProject()"
  />
</div>
```
这是不是看着就熟悉了，接着看 `importProject` 方法：

``` js
async importProject (force = false) {
  this.showNoModulesModal = false
  this.busy = true
  await this.$nextTick()
  try {
    await this.$apollo.mutate({
      mutation: PROJECT_IMPORT,
      variables: {
        input: {
          path: this.folderCurrent.path,
          force
        }
      }
    })

    this.$router.push({ name: 'project-home' })
  } catch (e) {
    if (e.graphQLErrors && e.graphQLErrors.some(e => e.message === 'NO_MODULES')) {
      this.showNoModulesModal = true
    }
    this.busy = false
  }
}
```
代码写的比较明了，当执行 `importProject` 时候会利用 vue-apollo 提供的 `this.$apollo.mutate()` 来发送一个 GraphQL 变更，从而改变服务端的数据，接下来就看服务端的 Mutation： `projectImport`。

## Mutation

处理 `projectImport` 变更的代码目录在 `@vue/cli-ui/apollo-server/schema/project.js` 中，代码如下：

``` js
exports.resolvers = {
  Project: { // ... 
  },
  Query: { // ... 
  },
  Mutation: {
    // ...
    projectImport: (root, { input }, context) => projects.import(input, context),
    // ...
  }
}
```
接着看 `projects.import`,代码目录在 `@vue/cli-ui/apollo-server/connectors/projects.js` 中，代码如下：

``` js
async function importProject (input, context) { // 导入项目，执行 projectImport mutate
  if (!input.force && !fs.existsSync(path.join(input.path, 'node_modules'))) { // 强制导入没有 node_modules 的情形
    throw new Error('NO_MODULES')
  }

  const project = {
    id: shortId.generate(), // shortId
    path: input.path, // 导入项目的路径
    favorite: 0,
    type: folders.isVueProject(input.path) ? 'vue' : 'unknown' // 是否为 vue 项目
  }
  const packageData = folders.readPackage(project.path, context)
  project.name = packageData.name
  context.db.get('projects').push(project).write() // 将 project 信息存在本地的 db 中 （ lowdb 实现 ）
  return open(project.id, context)
}
```
`importProject` 方法的作用就是获取导入项目的信息，并利用 [lowdb](https://github.com/typicode/lowdb) 将数据存储在本地（~/.vue-cli-ui/db.json），接着执行 open 方法加载插件。

## open

``` js
async function open (id, context) {
  const project = findOne(id, context)
  // ...
  cwd.set(project.path, context) // process.env.VUE_CLI_CONTEXT
  // Reset locales
  locales.reset(context)
  // Load plugins
  // 加载插件
  await plugins.list(project.path, context)
  // ...
  return project
}
```
以上为 open 方法的部分代码，比较核心的就是  `await plugins.list(project.path, context)`,接着看

``` js
async function list (file, context, { resetApi = true, lightApi = false, autoLoadApi = true } = {}) {
  let pkg = folders.readPackage(file, context)
  let pkgContext = cwd.get()
  // Custom package.json location
  if (pkg.vuePlugins && pkg.vuePlugins.resolveFrom) { // 加载其他文件夹里的 package.json
    pkgContext = path.resolve(cwd.get(), pkg.vuePlugins.resolveFrom)
    pkg = folders.readPackage(pkgContext, context)
  }
  pkgStore.set(file, { pkgContext, pkg })

  let plugins = []
  // package.json 中 devDependencies，dependencies 插件
  plugins = plugins.concat(findPlugins(pkg.devDependencies || {}, file))
  plugins = plugins.concat(findPlugins(pkg.dependencies || {}, file))

  // Put cli service at the top
  const index = plugins.findIndex(p => p.id === CLI_SERVICE)
  if (index !== -1) {
    const service = plugins[index]
    plugins.splice(index, 1)
    plugins.unshift(service)
  }

  pluginsStore.set(file, plugins)

  log('Plugins found:', plugins.length, chalk.grey(file))

  if (resetApi || (autoLoadApi && !pluginApiInstances.has(file))) {
    await resetPluginApi({ file, lightApi }, context)
  }
  return plugins
}

```
`list` 方法首先会获取 package.json 里 `devDependencies` 和 `dependencies` 字段中的 UI 插件，接着执行 `resetPluginApi` 函数调用 UI 插件的 API，`resetPluginApi` 方法部分代码如下：

``` js
function resetPluginApi ({ file, lightApi }, context) {
  return new Promise((resolve, reject) => {
    // ...
    // Cyclic dependency with projects connector
    setTimeout(async () => {
      // ...
      pluginApi = new PluginApi({
        plugins,
        file,
        project,
        lightMode: lightApi
      }, context)
      pluginApiInstances.set(file, pluginApi)

      // Run Plugin API
      // 默认的插件 suggest,task,config,widgets
      runPluginApi(path.resolve(__dirname, '../../'), pluginApi, context, 'ui-defaults')

      // devDependencies dependencies 插件
      plugins.forEach(plugin => runPluginApi(plugin.id, pluginApi, context))
      // Local plugins
      // package.json 中 vuePlugins.ui 插件
      const { pkg, pkgContext } = pkgStore.get(file)
      if (pkg.vuePlugins && pkg.vuePlugins.ui) {
        const files = pkg.vuePlugins.ui
        if (Array.isArray(files)) {
          for (const file of files) {
            runPluginApi(pkgContext, pluginApi, context, file)
          }
        }
      }
      // Add client addons
      pluginApi.clientAddons.forEach(options => {
        clientAddons.add(options, context)
      })
      // Add views
      for (const view of pluginApi.views) {
        await views.add({ view, project }, context)
      }
      // Register widgets
      for (const definition of pluginApi.widgetDefs) {
        await widgets.registerDefinition({ definition, project }, context)
      }
      // callHook ...
      // Load widgets for current project
      widgets.load(context)
      resolve(true)
    })
  })
}
```

`resetPluginApi` 方法主要利用函数 `runPluginApi` 执行所有 UI 插件的 PluginAPI，这里的 UI 插件来源主要有三部分组成：

* **内置 UI 插件**：包括了配置插件，建议插件（vue-router，vuex），任务插件以及看板部分的 widget 插件
* **package.json UI 插件**：项目中依赖的 UI 插件，可以通过 vuePlugins.resolveFrom 指定 package.json 位置
* **vuePlugins.ui**： package.json 中 vuePlugins 字段中的 UI 插件，这样可以直接访问插件 API