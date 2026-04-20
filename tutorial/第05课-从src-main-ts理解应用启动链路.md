# 第 5 课：从 `src/main.ts` 理解应用启动链路

## 课程定位

前四课主要是在建立地图：后台系统地图、环境地图、技术栈地图和源码目录地图。从第 5 课开始，正式进入源码流程。

本课的核心是 `src/main.ts`。它是 Vue 应用的启动入口，也是理解整个项目如何把插件、状态、路由、国际化和根组件组合起来的第一站。

学完本课后，你不需要理解每个插件内部的全部实现，但要能清楚说明：应用从加载入口文件到页面挂载，中间按什么顺序做了哪些事，为什么这个顺序不能随意打乱。

## 学习目标

学完本课后，你应该能够：

- 说明 `src/main.ts` 在项目中的作用。
- 解释 `createApp(App)` 做了什么。
- 说明 `setupApp` 中每个步骤的职责。
- 理解为什么部分 setup 可以在 `createApp` 前执行。
- 理解为什么 `setupStore(app)` 要早于 `setupRouter(app)`。
- 理解为什么 `setupRouter(app)` 需要 `await`。
- 理解为什么最后才执行 `app.mount('#app')`。
- 能够给 `setupApp` 每一步添加日志或断点观察启动顺序。

## 建议课时

建议用 90-120 分钟完成本课：

- 15 分钟：阅读 `src/main.ts`。
- 20 分钟：理解 `setupApp` 的执行顺序。
- 20 分钟：阅读 `src/plugins/index.ts`、`src/store/index.ts`、`src/router/index.ts`、`src/locales/index.ts`。
- 15 分钟：阅读 `src/App.vue`。
- 20 分钟：添加日志或断点观察启动顺序。
- 10 分钟：画启动流程图。

## 一、`src/main.ts` 是什么

`src/main.ts` 是前端应用的入口文件。

浏览器加载项目后，Vite 会从入口模块开始解析依赖，最终执行 `src/main.ts` 中的代码。这个文件负责把项目中分散的能力组装起来：

- 全局样式和资源。
- loading。
- 进度条。
- 图标。
- 日期库。
- Vue 应用实例。
- Element Plus。
- Pinia。
- Vue Router。
- vue-i18n。
- 版本更新检测。
- 根组件挂载。

也就是说，`src/main.ts` 不是普通业务文件，而是应用启动编排文件。

## 二、先看完整源码

当前项目的 `src/main.ts`：

```ts
import { createApp } from 'vue';
import './plugins/assets';
import {
  setupAppVersionNotification,
  setupDayjs,
  setupIconifyOffline,
  setupLoading,
  setupNProgress,
  setupUI
} from './plugins';
import { setupStore } from './store';
import { setupRouter } from './router';
import { setupI18n } from './locales';
import App from './App.vue';

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

setupApp();
```

先不要急着看每个 setup 的内部实现。先把它分成 4 个阶段。

## 三、`setupApp` 的 4 个阶段

### 阶段 1：导入全局资源

```ts
import './plugins/assets';
```

这一行没有导入变量，是副作用导入。

它的作用是执行 `src/plugins/assets.ts` 里的全局资源导入：

```ts
import 'virtual:svg-icons-register';
import 'element-plus/dist/index.css';
import 'element-plus/theme-chalk/dark/css-vars.css';
import 'uno.css';
import '../styles/css/global.css';
import 'swiper/css';
import 'swiper/css/navigation';
import 'swiper/css/pagination';
```

这里做了几件事：

- 注册本地 SVG 图标。
- 引入 Element Plus 基础样式。
- 引入 Element Plus 暗色变量。
- 引入 UnoCSS 生成的样式。
- 引入项目全局 CSS。
- 引入 Swiper 样式。

这一步发生在 `setupApp()` 执行之前，因为它是模块顶层 import。

### 阶段 2：初始化不依赖 Vue app 的全局能力

```ts
setupLoading();
setupNProgress();
setupIconifyOffline();
setupDayjs();
```

这些函数不需要 `app.use(...)`，所以可以在 `createApp(App)` 之前执行。

它们属于环境准备：

- `setupLoading()`：在 Vue 应用真正挂载前，先往 `#app` 中写入首屏 loading。
- `setupNProgress()`：配置顶部进度条，并挂到 `window.NProgress`。
- `setupIconifyOffline()`：配置 Iconify 图标资源地址。
- `setupDayjs()`：扩展 Dayjs 插件，并设置语言。

### 阶段 3：创建 Vue 应用并注册插件

```ts
const app = createApp(App);

setupUI(app);

setupStore(app);

await setupRouter(app);

setupI18n(app);
```

从 `createApp(App)` 开始，项目得到了 Vue 应用实例 `app`。

后续需要接入 Vue 应用的能力，都通过 `app` 注册：

- `setupUI(app)` 注册 Element Plus。
- `setupStore(app)` 注册 Pinia。
- `setupRouter(app)` 注册 Vue Router 和路由守卫。
- `setupI18n(app)` 注册 vue-i18n。

### 阶段 4：应用增强和挂载

```ts
setupAppVersionNotification();

app.mount('#app');
```

版本更新检测是应用级增强能力。

最后执行：

```ts
app.mount('#app');
```

把 Vue 根组件挂载到 `index.html` 中的：

```html
<div id="app"></div>
```

挂载完成后，Vue 才真正接管页面渲染。

## 四、为什么先导入 `./plugins/assets`

`import './plugins/assets'` 是全局资源导入。

它必须尽早执行，因为它包含：

- Element Plus 样式。
- UnoCSS 样式。
- 项目全局样式。
- SVG 图标注册。
- Swiper 样式。

如果不导入这些资源，可能出现：

- Element Plus 组件没有样式。
- UnoCSS 原子类不生效。
- 本地 SVG 图标无法使用。
- 全局 reset、transition、nprogress 样式缺失。
- Swiper 组件样式异常。

这类文件的特点是：它不提供业务函数，而是影响整个应用运行环境。

所以它写在入口文件顶部。

## 五、`setupLoading()` 为什么最先执行

`setupLoading()` 的作用是首屏 loading。

在 Vue 应用完成初始化之前，页面可能需要等待：

- 插件初始化。
- 路由准备。
- 路由守卫执行。
- 权限路由初始化。
- 页面组件加载。

如果此时 `#app` 是空的，用户会看到白屏。

`setupLoading()` 会先把一段 loading HTML 写入 `#app`：

```ts
const app = document.getElementById('app');

if (app) {
  app.innerHTML = loading;
}
```

等最后执行：

```ts
app.mount('#app');
```

Vue 会接管 `#app`，首屏 loading 被真实应用替换。

所以它放在最前面是合理的。

## 六、`setupNProgress()` 做什么

`setupNProgress()` 配置页面顶部进度条：

```ts
NProgress.configure({ easing: 'ease', speed: 500 });

window.NProgress = NProgress;
```

它本身只是配置 NProgress，并挂到 `window` 上。

真正开始和结束进度条，通常由路由守卫控制：

- 路由开始跳转时 `NProgress.start()`。
- 路由完成后 `NProgress.done()`。

这就是为什么 NProgress 要早于 router guard 准备好。后面路由守卫使用它时，它已经存在。

## 七、`setupIconifyOffline()` 做什么

`setupIconifyOffline()` 用于配置 Iconify 图标 API provider：

```ts
const { VITE_ICONIFY_URL } = import.meta.env;

if (VITE_ICONIFY_URL) {
  addAPIProvider('', { resources: [VITE_ICONIFY_URL] });
}
```

如果项目部署在内网，不能访问默认 Iconify 服务，就可以通过环境变量：

```txt
VITE_ICONIFY_URL
```

指定自己的图标服务地址。

它不依赖 Vue app，因此可以在 `createApp` 前执行。

## 八、`setupDayjs()` 做什么

`setupDayjs()` 初始化日期库：

```ts
extend(localeData);

setDayjsLocale();
```

它做两件事：

- 给 Dayjs 扩展 `localeData` 插件。
- 根据当前语言设置 Dayjs locale。

日期格式化和本地化在后台项目中很常见，例如：

- 创建时间。
- 更新时间。
- 日志时间。
- 订单时间。
- 图表时间。

因此日期库应在应用渲染前准备好。

## 九、`createApp(App)` 做什么

```ts
const app = createApp(App);
```

这一步创建 Vue 应用实例。

这里的 `App` 来自：

```ts
import App from './App.vue';
```

可以理解为：

- `App.vue` 是根组件。
- `createApp(App)` 创建一个以 `App.vue` 为根的 Vue 应用。
- `app` 是应用实例，可以安装插件、注册组件、配置全局错误处理，最后挂载到 DOM。

注意：此时页面还没有被 Vue 渲染。

只有执行：

```ts
app.mount('#app');
```

Vue 应用才真正挂载。

## 十、`setupUI(app)` 为什么在前面

`setupUI(app)` 注册 Element Plus：

```ts
export const setupUI = (app: App) => {
  app.use(ElementPlus);
};
```

同时项目在 `src/plugins/ui.ts` 中还修改了部分组件默认 props：

```ts
ElTable.TableColumn.props.align = {
  type: String,
  default: 'center'
};

ElCard.props.shadow = {
  type: String,
  default: 'never'
};

ElForm.props.requireAsteriskPosition = {
  type: String,
  default: 'right'
};
```

这说明项目希望所有 Element Plus 组件在页面渲染前就已经注册好，并且默认行为已经调整好。

`App.vue` 中直接使用了：

```vue
<ElConfigProvider>
<ElWatermark>
```

所以 UI 插件必须早于 `app.mount('#app')`。

## 十一、`setupStore(app)` 为什么要早于 `setupRouter(app)`

`setupStore(app)` 注册 Pinia：

```ts
export function setupStore(app: App) {
  const store = createPinia();

  store.use(resetSetupStore);

  app.use(store);
}
```

它必须早于 `setupRouter(app)`，因为路由守卫里会使用 store。

例如路由守卫需要：

- 读取 token。
- 获取用户信息。
- 初始化权限路由。
- 生成菜单。
- 重置 auth store。
- 操作 tab store。

这些都依赖 Pinia。

如果先初始化 router，再注册 store，路由守卫中调用 `useAuthStore()`、`useRouteStore()` 时就可能出问题。

所以顺序应该是：

```txt
先注册 Pinia
再注册 Router 和路由守卫
```

对应代码：

```ts
setupStore(app);

await setupRouter(app);
```

这是本课最重要的顺序之一。

## 十二、`setupRouter(app)` 为什么需要 `await`

`setupRouter(app)` 的实现：

```ts
export async function setupRouter(app: App) {
  app.use(router);
  createRouterGuard(router);
  await router.isReady();
}
```

它做三件事：

1. 注册 router。
2. 创建路由守卫。
3. 等待初始路由解析完成。

`await router.isReady()` 很关键。

因为后台系统的初始路由可能涉及：

- 判断是否登录。
- 初始化常量路由。
- 初始化权限路由。
- 获取用户信息。
- 重定向到登录页。
- 重定向到首页。
- 处理 403/404。

如果不等待 router ready 就直接 `app.mount('#app')`，页面可能先渲染一个不稳定状态，再被路由重定向替换，造成闪烁或状态混乱。

所以这里写成：

```ts
await setupRouter(app);
```

表示：等路由系统准备好，再继续后续初始化和挂载。

## 十三、`setupI18n(app)` 为什么在 router 后面

`setupI18n(app)` 注册 vue-i18n：

```ts
export function setupI18n(app: App) {
  app.use(i18n);
}
```

在很多项目里，i18n 会放在 router 前面注册，因为路由标题、菜单、通知都可能依赖语言包。

但本项目中，`$t` 是通过模块导出的：

```ts
export const $t = i18n.global.t as App.I18n.$T;
```

这意味着很多非组件代码可以直接导入 `$t` 使用，而不一定依赖 `app.use(i18n)` 之后才能调用。

所以当前顺序是：

```ts
await setupRouter(app);

setupI18n(app);
```

它在这个项目中是可行的。

但从理解上要记住：

- `setupI18n(app)` 必须早于 `app.mount('#app')`。
- 因为 `App.vue` 和页面模板中可能使用 `$t`。

## 十四、`setupAppVersionNotification()` 为什么靠后

`setupAppVersionNotification()` 用于生产环境检查应用是否有新版本。

它会根据环境变量判断：

```ts
const canAutoUpdateApp = import.meta.env.VITE_AUTOMATICALLY_DETECT_UPDATE === 'Y' && import.meta.env.PROD;
```

如果不是生产环境，直接返回。

这个能力不是页面首次渲染的必要条件，因此放在核心插件注册之后。

它依赖通知组件和国际化文案，所以放在 UI 和 i18n 相关能力之后更合理。

## 十五、为什么最后才 `app.mount('#app')`

`app.mount('#app')` 是 Vue 应用真正接管页面的时刻。

在它之前，项目完成了：

- 全局样式加载。
- 首屏 loading。
- 进度条配置。
- 图标配置。
- 日期库配置。
- Vue app 创建。
- UI 插件注册。
- Pinia 注册。
- Router 注册和初始路由 ready。
- i18n 注册。
- 版本更新检测初始化。

这样页面首次渲染时，各项基础能力都已经准备好。

如果提前 mount，可能出现：

- 组件库未注册。
- store 未注册。
- router 未 ready。
- 页面先闪一下再跳转。
- 国际化不可用。
- App.vue 中使用的 store 或 UI 组件异常。

所以 `mount` 必须放在最后。

## 十六、App.vue 为什么依赖这些插件

`App.vue` 看起来很简单，但它已经依赖多个全局能力：

```ts
const appStore = useAppStore();
const themeStore = useThemeStore();
const authStore = useAuthStore();
```

这说明它依赖 Pinia。

模板中：

```vue
<ElConfigProvider :locale="locale">
  <AppProvider>
    <ElWatermark class="h-full" v-bind="watermarkProps">
      <RouterView class="bg-layout" />
    </ElWatermark>
  </AppProvider>
</ElConfigProvider>
```

这里依赖：

- `ElConfigProvider`：Element Plus。
- `ElWatermark`：Element Plus。
- `AppProvider`：全局组件。
- `RouterView`：Vue Router。
- `locale`：国际化和 app store。
- `watermarkProps`：theme store 和 auth store。

这也解释了为什么 `main.ts` 要在 mount 前注册：

- UI。
- Store。
- Router。
- i18n。

根组件不是孤立的，它一渲染就会使用这些能力。

## 十七、完整启动流程图

可以把启动流程画成：

```txt
加载 src/main.ts
  ↓
导入 ./plugins/assets
  ↓
执行 setupApp()
  ↓
setupLoading()
  ↓
setupNProgress()
  ↓
setupIconifyOffline()
  ↓
setupDayjs()
  ↓
createApp(App)
  ↓
setupUI(app)
  ↓
setupStore(app)
  ↓
await setupRouter(app)
  ↓
setupI18n(app)
  ↓
setupAppVersionNotification()
  ↓
app.mount('#app')
  ↓
渲染 App.vue
  ↓
RouterView 渲染当前路由页面
```

如果只记一条主线，就是：

```txt
资源准备 → 全局环境准备 → 创建 Vue 应用 → 注册插件 → 等路由就绪 → 挂载应用
```

## 十八、如何加日志观察启动顺序

你可以临时把 `src/main.ts` 改成：

```ts
async function setupApp() {
  console.log('[setupApp] 1 setupLoading start');
  setupLoading();
  console.log('[setupApp] 1 setupLoading end');

  console.log('[setupApp] 2 setupNProgress start');
  setupNProgress();
  console.log('[setupApp] 2 setupNProgress end');

  console.log('[setupApp] 3 setupIconifyOffline start');
  setupIconifyOffline();
  console.log('[setupApp] 3 setupIconifyOffline end');

  console.log('[setupApp] 4 setupDayjs start');
  setupDayjs();
  console.log('[setupApp] 4 setupDayjs end');

  console.log('[setupApp] 5 createApp start');
  const app = createApp(App);
  console.log('[setupApp] 5 createApp end');

  console.log('[setupApp] 6 setupUI start');
  setupUI(app);
  console.log('[setupApp] 6 setupUI end');

  console.log('[setupApp] 7 setupStore start');
  setupStore(app);
  console.log('[setupApp] 7 setupStore end');

  console.log('[setupApp] 8 setupRouter start');
  await setupRouter(app);
  console.log('[setupApp] 8 setupRouter end');

  console.log('[setupApp] 9 setupI18n start');
  setupI18n(app);
  console.log('[setupApp] 9 setupI18n end');

  console.log('[setupApp] 10 setupAppVersionNotification start');
  setupAppVersionNotification();
  console.log('[setupApp] 10 setupAppVersionNotification end');

  console.log('[setupApp] 11 mount start');
  app.mount('#app');
  console.log('[setupApp] 11 mount end');
}
```

然后执行：

```bash
pnpm dev
```

打开浏览器控制台，观察输出顺序。

重点观察：

- `setupStore` 是否早于 `setupRouter`。
- `setupRouter start` 和 `setupRouter end` 之间是否有等待。
- `mount` 是否最后执行。

调试完成后，要删除这些日志。

## 十九、如何加断点观察启动顺序

除了日志，你也可以使用浏览器断点。

步骤：

1. 启动项目：`pnpm dev`。
2. 打开浏览器开发者工具。
3. 进入 `Sources` 面板。
4. 搜索 `main.ts`。
5. 在 `setupApp` 每个调用行左侧点击添加断点。
6. 刷新页面。
7. 点击继续执行，观察每一步执行顺序。

也可以临时加入：

```ts
debugger;
```

例如：

```ts
debugger;
setupLoading();
```

但同样要注意：调试完后删除 `debugger`。

## 二十、哪些顺序可以调整，哪些不建议调整

### 不建议调整的顺序

```ts
setupStore(app);
await setupRouter(app);
app.mount('#app');
```

原因：

- Router guard 依赖 store。
- mount 前需要 router ready。
- App.vue 渲染时依赖 store、router、UI。

### 可以讨论但不建议随意调整的顺序

```ts
setupUI(app);
setupStore(app);
await setupRouter(app);
setupI18n(app);
```

`setupI18n(app)` 在一些项目中可能会放在 router 前面。本项目当前顺序能工作，因为 `$t` 作为模块函数导出，非组件代码可以直接使用。

如果后续你自己从零搭建项目，推荐保守顺序是：

```ts
setupUI(app);
setupStore(app);
setupI18n(app);
await setupRouter(app);
app.mount('#app');
```

这样可以避免路由守卫或标题逻辑强依赖 i18n 插件时出问题。

但学习本项目时，先理解它当前的实际顺序。

### 可以放在 `createApp` 前的初始化

```ts
setupLoading();
setupNProgress();
setupIconifyOffline();
setupDayjs();
```

这些不需要 Vue app 实例。

## 二十一、常见误区

### 误区 1：以为 `createApp(App)` 就已经渲染页面

不是。

`createApp(App)` 只是创建应用实例。

真正渲染页面的是：

```ts
app.mount('#app');
```

### 误区 2：以为插件注册顺序无所谓

不是。

有些插件之间有依赖关系。

例如：

- 路由守卫依赖 store。
- App.vue 依赖 UI、store、router。
- 页面文案依赖 i18n。

### 误区 3：忽略 `await setupRouter(app)`

后台系统的路由不只是跳转，还和登录、权限、菜单、重定向有关。

等待 router ready 可以减少首屏闪烁和不稳定状态。

### 误区 4：一开始就深入每个插件内部

本课重点是启动链路，不是插件源码。

你只需要知道每个 setup 的职责。

第 7 课会更深入地看插件系统。

## 二十二、本课源码导读

### 1. `src/main.ts`

重点看：

- 导入了哪些能力。
- `setupApp` 的顺序。
- 哪些函数在 `createApp` 前执行。
- 哪些函数需要 `app`。
- 哪一步用了 `await`。
- 哪一步挂载应用。

### 2. `src/plugins/index.ts`

重点看：

```ts
export * from './loading';
export * from './nprogress';
export * from './iconify';
export * from './dayjs';
export * from './app';
export * from './ui';
```

它只是统一导出插件初始化函数，方便 `main.ts` 一次性从 `./plugins` 导入。

### 3. `src/store/index.ts`

重点看：

```ts
const store = createPinia();
store.use(resetSetupStore);
app.use(store);
```

理解 Pinia 是如何注册到 Vue app 的。

### 4. `src/router/index.ts`

重点看：

```ts
app.use(router);
createRouterGuard(router);
await router.isReady();
```

理解 router 不只是注册，还创建守卫并等待 ready。

### 5. `src/locales/index.ts`

重点看：

```ts
const i18n = createI18n(...);
app.use(i18n);
export const $t = i18n.global.t;
```

理解 i18n 的注册和 `$t` 的模块导出。

### 6. `src/App.vue`

重点看：

- 使用了哪些 store。
- 使用了哪些 Element Plus 组件。
- 使用了 `RouterView`。
- 为什么这些依赖需要在 mount 前准备好。

## 二十三、课堂演示建议

### 演示 1：给 `setupApp` 标序号

把每一步写成：

```ts
// 1. setup loading
setupLoading();
```

讲解每一步职责。

### 演示 2：添加日志观察顺序

使用 `console.log` 标记开始和结束。

重点观察 `await setupRouter(app)`。

### 演示 3：删除 `await` 做对比

仅作为课堂实验，不要提交代码。

把：

```ts
await setupRouter(app);
```

临时改成：

```ts
setupRouter(app);
```

观察是否有类型提示或运行时行为变化。

讲解重点：

- 异步初始化不要随便忽略。
- 后台项目初始路由可能涉及权限和重定向。

### 演示 4：查看 `App.vue` 依赖

打开 `App.vue`，指出：

- store。
- Element Plus。
- RouterView。
- i18n locale。

讲解为什么 `main.ts` 的注册顺序会影响根组件。

## 二十四、本课作业

### 作业 1：给 `setupApp` 每一步写注释

在笔记中整理：

```md
# setupApp 步骤说明

1. setupLoading：
2. setupNProgress：
3. setupIconifyOffline：
4. setupDayjs：
5. createApp：
6. setupUI：
7. setupStore：
8. setupRouter：
9. setupI18n：
10. setupAppVersionNotification：
11. app.mount：
```

每一步用自己的话说明作用。

### 作业 2：画应用启动流程图

用下面格式画：

```txt
main.ts
→ assets
→ loading
→ nprogress
→ iconify
→ dayjs
→ createApp
→ UI
→ store
→ router ready
→ i18n
→ version notification
→ mount
→ App.vue
→ RouterView
```

### 作业 3：添加日志观察启动顺序

临时添加 `console.log`，记录浏览器控制台输出。

要求：

- 记录完整顺序。
- 重点记录 `setupRouter start` 和 `setupRouter end`。
- 调试完成后删除日志。

### 作业 4：回答顺序问题

回答：

```md
- 为什么 setupLoading 在 createApp 前？
- 为什么 setupStore 要早于 setupRouter？
- 为什么 setupRouter 需要 await？
- 为什么 app.mount 必须放在最后？
- App.vue 依赖哪些全局能力？
```

## 二十五、自检问题

完成本课后，尝试回答：

- `src/main.ts` 在项目中负责什么？
- `import './plugins/assets'` 为什么没有接收变量？
- `setupLoading` 为什么能在 `createApp` 前执行？
- `createApp(App)` 和 `app.mount('#app')` 的区别是什么？
- `setupUI(app)` 注册了什么？
- `setupStore(app)` 内部做了什么？
- 为什么路由守卫可能依赖 Pinia？
- `setupRouter(app)` 内部做了哪三件事？
- `router.isReady()` 的意义是什么？
- `setupI18n(app)` 为什么必须在 mount 前？
- `App.vue` 为什么能直接使用 `ElConfigProvider`、`ElWatermark`、`RouterView`？
- 调试启动顺序时应该观察哪些日志？

如果这些问题能回答出来，就可以进入第 6 课。

## 二十六、下一课预告

第 6 课会深入 Vite 配置与环境变量，重点学习：

- `defineConfig` 的作用。
- `loadEnv` 如何读取 `.env` 文件。
- `VITE_BASE_URL` 如何影响部署路径。
- `resolve.alias` 如何配置 `@` 和 `~`。
- `server.port`、`server.open`、`server.proxy` 的作用。
- `build.sourcemap` 如何受环境变量控制。

第 5 课理解的是应用如何启动，第 6 课会理解启动背后的 Vite 配置如何生效。
