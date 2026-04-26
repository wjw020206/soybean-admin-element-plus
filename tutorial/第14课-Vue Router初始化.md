# 第 14 课：Vue Router 初始化

## 课程定位

前面几节课我们已经看过：

- `main.ts` 里 `setupRouter(app)` 在启动链路中的位置。
- `route store` 如何初始化常量路由和权限路由。
- 路由守卫里如何判断登录、403、404。

但还有一个更基础的问题必须先吃透：

```txt
router 实例到底是怎么创建出来的？
项目为什么一开始只注册了两条内置路由？
hash / history / memory 三种模式在这个仓库里是怎么切换的？
为什么 setupRouter 里最后还要 await router.isReady()？
```

这就是本课的主题。

这一课建议你把注意力收回到“应用刚启动时发生了什么”这条主线，不要一上来就钻进权限路由。先把路由实例如何创建、何时可用、守卫何时挂上去这些基础动作看明白，后面再看动态路由才不会乱。

## 学习目标

学完本课后，你应该能够：

- 理解 `src/router/index.ts` 的职责。
- 理解这个项目如何根据环境变量切换路由模式。
- 区分 `hash`、`history`、`memory` 三种路由 history。
- 理解为什么项目启动时只先注册内置路由。
- 理解 `ROOT_ROUTE` 和 `NOT_FOUND_ROUTE` 的作用。
- 理解 `createBuiltinVueRoutes()` 为什么要把 elegant route 转成 Vue Router route。
- 理解 `setupRouter(app)` 的执行顺序。
- 理解 `createRouterGuard(router)` 的职责和注册顺序。
- 理解 `await router.isReady()` 的含义。
- 能自己动手切换 `hash` / `history` 模式并观察 URL 和刷新行为变化。

## 建议课时

建议用 70-100 分钟完成本课：

- 15 分钟：阅读 `src/router/index.ts`。
- 15 分钟：阅读 `src/router/routes/builtin.ts`。
- 10 分钟：阅读 `src/router/guard/index.ts`。
- 10 分钟：回到 `src/main.ts` 看 `setupRouter(app)` 的调用位置。
- 10 分钟：阅读 `.env` 中的 `VITE_ROUTER_HISTORY_MODE`、`VITE_ROUTE_HOME`。
- 10-20 分钟：动手把 `history` 改成 `hash` 并观察变化。

## 一、本课重点文件

本课重点阅读：

- [src/router/index.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/router/index.ts)
- [src/router/routes/builtin.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/router/routes/builtin.ts)
- [src/router/guard/index.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/router/guard/index.ts)
- [src/main.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/main.ts)
- [.env](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/.env)

## 二、先看主线：应用启动时路由是怎么接上的

在 [src/main.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/main.ts) 中：

```ts
async function setupApp() {
  setupLoading();

  setupNProgress();

  setupIconifyOffline();

  setupDayjs();

  const app = createApp(App);

  setupUI(app);

  setupStore(app);

  await setupRouter(app);

  setupI18n(app);

  setupAppVersionNotification();

  app.mount('#app');
}
```

这里要先抓住一个事实：

```txt
router 不是在 app.mount 之后再慢慢准备，
而是在 mount 之前就先初始化并等待 ready。
```

所以路由初始化链路是：

```txt
createApp(App)
  -> setupUI(app)
  -> setupStore(app)
  -> await setupRouter(app)
  -> setupI18n(app)
  -> app.mount('#app')
```

这说明项目作者的意图很明确：

```txt
先把 router 插件装好，守卫挂好，初始导航跑完，
再让应用真正挂载到页面。
```

## 三、`src/router/index.ts` 做了什么

本课最核心的文件是 [src/router/index.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/router/index.ts)：

```ts
const { VITE_ROUTER_HISTORY_MODE = 'history', VITE_BASE_URL } = import.meta.env;

const historyCreatorMap: Record<Env.RouterHistoryMode, (base?: string) => RouterHistory> = {
  hash: createWebHashHistory,
  history: createWebHistory,
  memory: createMemoryHistory
};

export const router = createRouter({
  history: historyCreatorMap[VITE_ROUTER_HISTORY_MODE](VITE_BASE_URL),
  routes: createBuiltinVueRoutes()
});

export async function setupRouter(app: App) {
  app.use(router);
  createRouterGuard(router);
  await router.isReady();
}
```

这个文件只做三件事：

1. 根据环境变量选择路由 history 模式。
2. 创建 Vue Router 实例。
3. 把 router 装到 app 上，并注册守卫，最后等待初始导航完成。

也就是说：

```txt
src/router/index.ts 不负责定义所有业务路由，
它只负责“创建 router 实例并把它接入应用”。
```

## 四、三种 history 模式分别是什么

项目通过这个映射表切换模式：

```ts
const historyCreatorMap = {
  hash: createWebHashHistory,
  history: createWebHistory,
  memory: createMemoryHistory
};
```

再通过 `.env` 中的：

```env
VITE_ROUTER_HISTORY_MODE=history
```

决定最终使用哪一种。

### 1. `history`

对应：

```ts
createWebHistory()
```

URL 看起来像：

```txt
/home
/manage/user
```

这是现在最常见的前端路由模式，URL 更干净。

但它有一个前提：

```txt
服务端必须正确做 history fallback。
```

否则你直接刷新 `/manage/user` 时，服务器会以为你在请求一个真实文件或真实后端路由，可能返回 404。

### 2. `hash`

对应：

```ts
createWebHashHistory()
```

URL 看起来像：

```txt
/#/home
/#/manage/user
```

特点是：

```txt
# 后面的内容不会发给服务器，
所以刷新时一般不需要服务端额外配合。
```

这也是很多后台模板早期常用的模式。

### 3. `memory`

对应：

```ts
createMemoryHistory()
```

它不会把路由状态写进浏览器地址栏，路由信息只存在内存里。

这个模式通常用于：

- SSR 某些场景
- 测试环境
- 非浏览器环境

在这个后台项目的正常浏览器运行里，几乎不会把它作为默认模式。

### 4. 这三种模式怎么选

在这个仓库里，默认是：

```env
VITE_ROUTER_HISTORY_MODE=history
```

原因很直接：

```txt
后台系统正式部署时，通常可以控制服务器配置，
所以 history 模式的 URL 更自然。
```

如果你只是本地练习，或者部署环境不方便配 history fallback，`hash` 会更省事。

## 五、`VITE_BASE_URL` 是干什么的

同样在 `.env` 中：

```env
VITE_BASE_URL=/
```

它会传给：

```ts
historyCreatorMap[VITE_ROUTER_HISTORY_MODE](VITE_BASE_URL)
```

也就是作为 router history 的 base。

如果项目不是部署在域名根目录，而是放在子目录下，例如：

```txt
https://example.com/admin/
```

那么这里通常就要改成：

```env
VITE_BASE_URL=/admin/
```

否则路由拼接、刷新和资源路径都可能出现问题。

## 六、为什么一开始只注册了内置路由

创建 router 时，传入的不是完整业务路由，而是：

```ts
routes: createBuiltinVueRoutes()
```

这表示：

```txt
应用刚启动时，Vue Router 里只先放“最基础、一定存在”的那几条路由。
```

对应文件在 [src/router/routes/builtin.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/router/routes/builtin.ts)：

```ts
const builtinRoutes: CustomRoute[] = [ROOT_ROUTE, NOT_FOUND_ROUTE];
```

为什么这样设计？

因为这个项目后面还有：

- 常量路由初始化
- 权限路由初始化
- 动态添加路由

这些都不是在 `createRouter(...)` 这一刻一次性写死的。

所以作者把启动阶段一定要有的路由先放进去：

1. 根路由 `/`
2. 兜底 404 路由

这样 router 实例一创建出来，最基础的导航能力就成立了。

## 七、两个内置路由各自做什么

### 1. `ROOT_ROUTE`

代码：

```ts
export const ROOT_ROUTE: CustomRoute = {
  name: 'root',
  path: '/',
  redirect: getRoutePath(import.meta.env.VITE_ROUTE_HOME) || '/home',
  meta: {
    title: 'root',
    constant: true
  }
};
```

这里的核心是：

```ts
redirect: getRoutePath(import.meta.env.VITE_ROUTE_HOME) || '/home'
```

而 `.env` 中：

```env
VITE_ROUTE_HOME=home
```

所以它的意思是：

```txt
当用户访问根路径 / 时，不直接渲染页面，
而是跳到系统的首页路由。
```

这个首页不是写死字符串 `/home`，而是先根据 `VITE_ROUTE_HOME` 计算。

这让项目具备一个好处：

```txt
首页路由可配置。
```

### 2. `NOT_FOUND_ROUTE`

代码：

```ts
const NOT_FOUND_ROUTE: CustomRoute = {
  name: 'not-found',
  path: '/:pathMatch(.*)*',
  component: 'layout.blank$view.404',
  meta: {
    title: 'not-found',
    constant: true
  }
};
```

它的作用是：

```txt
接住所有当前没匹配上的路径。
```

这里很重要的一点是：

```txt
not-found 路由不是“系统里永远不存在这个页面”的最终结论，
它在这个项目里还有一个“初始化阶段临时兜底”的职责。
```

因为权限路由后面是动态加进去的，所以用户第一次进入某些页面时，路由还没初始化完成，可能会先落到 `not-found`，随后守卫再判断这是：

- 真 404
- 403
- 还是初始化后可访问的权限路由

这个细节你在前面问过，和本课是连起来的。

## 八、`createBuiltinVueRoutes()` 为什么不是直接返回原对象

代码：

```ts
export function createBuiltinVueRoutes() {
  return transformElegantRoutesToVueRoutes(builtinRoutes, layouts, views);
}
```

注意这里的 `builtinRoutes` 类型不是原生 `RouteRecordRaw`，而是项目自己的：

```ts
CustomRoute
```

也就是 elegant-router 体系里的路由描述对象。

所以这一步的真实含义是：

```txt
先用项目内部更高层的路由描述方式写路由，
再统一转换成 Vue Router 真正需要的 RouteRecordRaw。
```

这么做的好处是：

- 路由定义风格统一。
- 可以复用布局字符串、视图字符串这套转换规则。
- 常量路由、权限路由、异常页路由都走同一套转换流程。

也就是说，`createBuiltinVueRoutes()` 本质上是一个桥接动作：

```txt
Elegant Router 描述 -> Vue Router 真实路由对象
```

## 九、`setupRouter(app)` 的顺序为什么这样安排

`setupRouter(app)` 只有三行，但顺序很关键：

```ts
app.use(router);
createRouterGuard(router);
await router.isReady();
```

### 第一步：`app.use(router)`

先把 router 插件装进 Vue 应用。

这一步之后：

- 组件里才能使用 `useRouter()`、`useRoute()`
- `<RouterView />`、`<RouterLink />` 才能正常工作

### 第二步：`createRouterGuard(router)`

然后马上注册全局守卫。

这表示：

```txt
在初始导航真正稳定下来之前，
项目希望进度条、权限判断、标题设置这些逻辑就已经挂上去了。
```

### 第三步：`await router.isReady()`

最后等待 router 完成初始导航。

这一步非常关键，后面单独讲。

## 十、路由守卫注册了哪些能力

在 [src/router/guard/index.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/router/guard/index.ts)：

```ts
export function createRouterGuard(router: Router) {
  createProgressGuard(router);
  createRouteGuard(router);
  createDocumentTitleGuard(router);
}
```

这里注册了三类守卫能力：

### 1. `createProgressGuard`

作用：

```txt
路由切换时控制顶部 NProgress 进度条。
```

### 2. `createRouteGuard`

作用：

```txt
处理登录校验、初始化常量路由、初始化权限路由、403/404 判断、登录页跳转等。
```

这是整个路由系统里最重的一层逻辑。

### 3. `createDocumentTitleGuard`

作用：

```txt
根据当前路由 meta 更新页面标题。
```

你会发现这里的注册顺序不是乱排的：

```txt
先进度条，再核心路由守卫，最后文档标题。
```

这反映的是“导航开始 -> 做权限与跳转判断 -> 最终确认页面标题”的执行思路。

## 十一、`router.isReady()` 到底在等什么

这是很多人第一次看 Vue Router 时最容易忽略的一句：

```ts
await router.isReady();
```

它的意思不是：

```txt
等待 router 对象创建完成
```

因为 `createRouter(...)` 执行完，router 对象本身早就有了。

它真正等待的是：

```txt
等待当前应用的首次导航完成。
```

也就是：

- 初始 URL 已经匹配完成
- 相关异步导航守卫已经跑完
- 最终应该停在哪个路由已经确定

这样做的好处是：

```txt
避免应用还没完成初始路由解析就提前 mount，
从而减少初始页面闪动、标题错位、守卫未完成就渲染页面等问题。
```

你可以把它理解成：

```txt
router 先把“第一跳”处理干净，
然后应用再正式挂载。
```

## 十二、为什么这个项目要在 `mount` 前等待 `isReady`

结合 [src/main.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/main.ts)：

```ts
await setupRouter(app);
setupI18n(app);
app.mount('#app');
```

项目选择这种顺序，是因为它的路由初始化并不轻：

- 有进度条守卫
- 有登录校验
- 有常量路由初始化
- 有权限路由初始化
- 有 403 / 404 二次判断

如果不等 `router.isReady()`，应用可能会先把某个中间态页面挂出来，然后再被守卫重定向，用户看到的就是：

- 页面闪一下
- 地址栏切一下
- 标题先错后对
- 进度条状态不稳定

所以这里的 `await` 不是多余动作，而是为了让启动阶段更稳定。

## 十三、把整个初始化链路串起来

现在你可以把本课的主线整理成下面这条：

```txt
main.ts
  -> await setupRouter(app)
    -> app.use(router)
    -> createRouterGuard(router)
    -> await router.isReady()

router/index.ts
  -> 根据 VITE_ROUTER_HISTORY_MODE 选择 history 创建器
  -> 根据 VITE_BASE_URL 创建 history
  -> createRouter({ history, routes: createBuiltinVueRoutes() })

router/routes/builtin.ts
  -> 先准备 ROOT_ROUTE
  -> 再准备 NOT_FOUND_ROUTE
  -> 把 CustomRoute 转成 Vue Router routes
```

这就是“Vue Router 初始化”这一课真正要掌握的骨架。

## 十四、你现在应该重点记住什么

这一课最容易记混的点，我直接帮你压缩成四句：

### 1

```txt
src/router/index.ts 的职责不是定义所有业务路由，
而是创建 router 实例并把它接入应用。
```

### 2

```txt
应用启动时只先注册内置路由，
权限路由和常量业务路由后面再初始化。
```

### 3

```txt
hash / history / memory 的切换入口就是 VITE_ROUTER_HISTORY_MODE。
```

### 4

```txt
await router.isReady() 等的不是“router 对象存在”，
而是“首次导航流程已经完成”。
```

## 十五、本课源码导读怎么用

这节课建议你按下面顺序读源码，不要来回跳：

1. 先看 [src/main.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/main.ts)，确认 `setupRouter(app)` 的位置。
2. 再看 [src/router/index.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/router/index.ts)，理解 router 创建和 `setupRouter`。
3. 再看 [src/router/routes/builtin.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/router/routes/builtin.ts)，理解为什么启动时只有两条路由。
4. 最后看 [src/router/guard/index.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/router/guard/index.ts)，理解守卫注册入口。

你这一课不需要马上把 `createRouteGuard` 的全部细节吃完，知道它是“初始化和权限判断的总入口”就够了。下一课再往里钻更合适。

## 十六、动手练习

请你自己完成下面两个练习。

### 练习 1：切换成 `hash` 模式

把 `.env` 中的：

```env
VITE_ROUTER_HISTORY_MODE=history
```

改成：

```env
VITE_ROUTER_HISTORY_MODE=hash
```

然后重新启动项目，观察：

- URL 是否出现 `#/`
- 刷新页面后行为是否变化
- 直接访问某个子路由时和 `history` 模式有什么区别

### 练习 2：修改首页路由

把：

```env
VITE_ROUTE_HOME=home
```

改成一个你确认存在的其他路由名，然后访问 `/`，观察根路由重定向结果。

这个练习的目的，是让你真正理解：

```txt
ROOT_ROUTE 的 redirect 是由环境变量驱动的。
```

## 十七、课后思考题

请你带着下面几个问题继续看源码：

1. 为什么项目要把 `not-found` 路由在启动时就注册进去？
2. 为什么权限路由不是直接在 `createRouter` 里一次性写死？
3. `createRouteGuard` 里是如何利用 `not-found` 做初始化阶段兜底的？
4. 如果部署环境不支持 history fallback，这个项目改成 `hash` 后会少掉哪些服务端要求？

## 十八、本课小结

本课你应该已经建立起这样一个认识：

```txt
Vue Router 初始化不是“new 一个 router 就完了”，
而是：
  先根据环境变量创建 history
  -> 再用内置路由创建 router 实例
  -> 把 router 装进 app
  -> 注册全局守卫
  -> 等待首次导航完成
  -> 最后再进入正式渲染
```

这条链路一旦看清，后面的路由权限、动态路由、菜单生成都会更好理解。

下一课建议继续看：

```txt
第 15 课：路由守卫与首次进入流程
```

到那一课，我们会把 `createRouteGuard` 里的初始化分支、登录判断、403/404 分流彻底拆开。
