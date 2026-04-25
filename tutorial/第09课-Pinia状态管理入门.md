# 第 9 课：Pinia 状态管理入门

## 课程定位

前一课我们已经理解了这个项目的 TypeScript 类型体系，知道了 `Api.*`、`App.*`、`Env.*` 这些类型如何约束数据。接下来要解决另一个关键问题：**这些数据在项目里放在哪里、如何在多个页面之间共享、如何在路由切换时保持一致。**

这就是 Pinia 要解决的事情。

在后台管理项目里，有很多状态不是某个页面私有的：

- 当前是否登录。
- 当前用户信息、角色、按钮权限。
- 当前可访问的路由和菜单。
- 当前打开的标签页。
- 当前主题模式、布局模式、主题色。
- 当前语言、侧边栏折叠状态、移动端布局状态。

这些状态如果都放在单个组件里，会很快失控。项目需要一个全局状态管理层，把这些跨页面共享的数据集中管理，而 Pinia 就是这层能力。

本课会结合仓库中的真实代码来学，不会只讲概念。你会看到：

- Pinia 是怎么在项目里注册的。
- 这个仓库为什么统一使用 setup store 写法。
- 为什么 setup store 默认没有 `$reset`，项目又是怎么补上的。
- `auth`、`route`、`tab`、`theme`、`app` 这五个 store 各自负责什么。
- store 和路由、请求、本地缓存是怎么联动的。

## 学习目标

学完本课后，你应该能够：

- 理解 Pinia 在本项目中的角色。
- 说明 `src/store/index.ts` 如何注册 Pinia。
- 理解 `defineStore` 的 setup store 写法。
- 理解为什么本项目大量使用 `ref`、`computed`、`watch` 来组织 store。
- 理解 `SetupStoreId` 的作用。
- 理解 setup store 为什么默认没有 `$reset`。
- 理解 `resetSetupStore` 插件如何给 setup store 补 `$reset`。
- 说明 `auth`、`route`、`tab`、`theme`、`app` 五个 store 的职责。
- 知道 store 如何和本地缓存、路由守卫、主题系统联动。
- 能够建立后续学习登录、权限、菜单、标签页时的状态管理主线。

## 建议课时

建议用 100-130 分钟完成本课：

- 10 分钟：浏览 `src/store` 目录。
- 15 分钟：理解 `setupStore(app)` 和 Pinia 插件注册。
- 20 分钟：理解 setup store 写法和 `$reset` 插件。
- 15 分钟：理解 `auth store`。
- 20 分钟：理解 `route store` 和 `tab store`。
- 15 分钟：理解 `theme store`。
- 15 分钟：理解 `app store`。
- 20 分钟：做一次 store 联动关系梳理。

## 一、先建立一个总体认识

当前 `src/store` 目录结构是：

```txt
src/store
├── index.ts
├── modules
│   ├── app
│   │   └── index.ts
│   ├── auth
│   │   ├── index.ts
│   │   └── shared.ts
│   ├── route
│   │   ├── index.ts
│   │   └── shared.ts
│   ├── tab
│   │   ├── index.ts
│   │   └── shared.ts
│   └── theme
│       ├── index.ts
│       └── shared.ts
└── plugins
    └── index.ts
```

你可以先把它理解成三层：

```txt
index.ts            Pinia 注册入口
modules/*           具体业务 store
plugins/*           对 Pinia 能力做增强
```

这说明本项目的 store 设计不是“一个大仓库里放所有状态”，而是**按业务领域拆分多个 store**。

这很适合后台管理项目，因为后台项目天然分模块：

- 登录认证一套。
- 路由权限一套。
- 标签页一套。
- 主题布局一套。
- 应用级交互状态一套。

## 二、Pinia 在这个项目中解决什么问题

在这个项目里，Pinia 主要解决的是：

### 1. 跨页面共享状态

例如用户登录后：

- 登录页提交账号密码。
- 顶部用户信息区要显示用户名称。
- 路由守卫要根据角色决定可访问页面。
- 标签页系统要记录打开的页面。

这些地方都需要访问一份统一的用户登录状态。

### 2. 管理全局 UI 状态

例如：

- 当前是否暗色模式。
- 当前布局模式。
- 侧边栏是否折叠。
- 当前语言。
- 主题抽屉是否打开。

这些都属于“应用级状态”，不适合放在某个局部组件里。

### 3. 串联路由、请求和本地缓存

后台管理系统里常见链路是：

```txt
登录成功
  -> 缓存 token
  -> 请求用户信息
  -> 初始化权限路由
  -> 初始化首页 tab
  -> 页面跳转
```

这条链路跨了多个模块。Pinia store 是承接这条链路的核心层。

## 三、`src/store/index.ts`：Pinia 的注册入口

当前内容是：

```ts
import type { App } from 'vue';
import { createPinia } from 'pinia';
import { resetSetupStore } from './plugins';

/** Setup Vue store plugin pinia */
export function setupStore(app: App) {
  const store = createPinia();

  store.use(resetSetupStore);

  app.use(store);
}
```

你可以把它拆成三步理解。

### 第一步：创建 Pinia 实例

```ts
const store = createPinia();
```

这和 Vue Router 中的 `createRouter()` 类似。意思是创建一个全局状态容器。

### 第二步：注册 Pinia 插件

```ts
store.use(resetSetupStore);
```

这是本项目的一个关键点。它不是 Pinia 必须写法，而是项目自己加的增强能力。后面会详细讲为什么要这样做。

### 第三步：挂到 Vue 应用上

```ts
app.use(store);
```

这样应用中的组件、组合函数、路由守卫里就可以使用：

```ts
useAuthStore()
useRouteStore()
useThemeStore()
```

这一点和：

```ts
app.use(router)
app.use(i18n)
```

是同类动作，都是把全局能力注册给 Vue 应用。

## 四、Pinia 是在什么时候被注册的

在应用启动链路里，`src/main.ts` 会调用 `setupStore(app)`。

所以顺序上你可以理解为：

```txt
createApp(App)
  -> setupStore(app)
  -> setupRouter(app)
  -> setupI18n(app)
  -> app.mount('#app')
```

为什么要比较早注册？

因为后面的很多逻辑都依赖 store：

- 路由守卫里要用 `authStore`、`routeStore`。
- 应用初始化时要用 `themeStore`。
- 页面组件里要用 `tabStore`、`appStore`。

如果 Pinia 没注册，后面的 `useXxxStore()` 就拿不到上下文。

## 五、Pinia 的核心概念：store 是什么

store 可以理解为：

```txt
一组“状态 + 计算属性 + 修改状态的方法 + 监听副作用”的组合
```

这个项目里大量使用的是 Pinia 的 setup store 写法。

例如 `auth store`：

```ts
export const useAuthStore = defineStore(SetupStoreId.Auth, () => {
  const token = ref(getToken());

  const userInfo: Api.Auth.UserInfo = reactive({
    userId: '',
    userName: '',
    roles: [],
    buttons: []
  });

  const isLogin = computed(() => Boolean(token.value));

  async function login(userName: string, password: string, redirect = true) {
    ...
  }

  return {
    token,
    userInfo,
    isLogin,
    login
  };
});
```

这和组件里的组合式 API 很像，因为本质上它就是“在 store 里使用 Composition API”。

## 六、为什么本项目统一使用 setup store

Pinia 有两种常见写法：

### 1. option store

```ts
defineStore('user', {
  state: () => ({ count: 0 }),
  getters: {
    double: state => state.count * 2
  },
  actions: {
    increment() {
      this.count++;
    }
  }
});
```

### 2. setup store

```ts
defineStore('user', () => {
  const count = ref(0);
  const double = computed(() => count.value * 2);

  function increment() {
    count.value++;
  }

  return { count, double, increment };
});
```

本项目选择 setup store，原因很现实：

### 原因 1：和 Vue 3 Composition API 风格统一

整个项目本身就是 Vue 3 + `<script setup>` 风格。

组件里大量使用：

```ts
ref
reactive
computed
watch
effectScope
```

store 继续用 setup 风格，整体一致，阅读成本更低。

### 原因 2：更适合复杂逻辑

例如 `theme store` 和 `app store` 里有很多：

- `watch`
- `effectScope`
- `usePreferredColorScheme`
- `useBreakpoints`
- `useEventListener`

这类逻辑在 setup store 里写起来更自然。

### 原因 3：更容易复用组合式函数

例如：

```ts
const { loading, startLoading, endLoading } = useLoading();
const { toLogin, redirectFromLogin } = useRouterPush(false);
const breakpoints = useBreakpoints(breakpointsTailwind);
```

这些都很符合 Composition API 风格。

## 七、`SetupStoreId` 是做什么的

在 [src/enum/index.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/enum/index.ts) 中定义了：

```ts
export enum SetupStoreId {
  App = 'app-store',
  Theme = 'theme-store',
  Auth = 'auth-store',
  Route = 'route-store',
  Tab = 'tab-store'
}
```

每个 store 都这样写：

```ts
defineStore(SetupStoreId.Auth, () => { ... })
```

而不是直接写：

```ts
defineStore('auth-store', () => { ... })
```

这样做的好处是：

- 所有 store id 集中管理。
- 避免手写字符串时拼错。
- Pinia 插件里可以统一识别这些 setup store。

后面你会看到 `resetSetupStore` 插件就是根据这些 id 来工作的。

## 八、setup store 默认为什么没有 `$reset`

这是 Pinia 一个很容易被忽略的点。

对于 option store，Pinia 原生提供 `$reset()`，因为它能根据 `state: () => ({ ... })` 自动知道初始状态。

但 setup store 里状态是你手动通过：

```ts
ref(...)
reactive(...)
computed(...)
```

组合出来的。Pinia 不知道你所谓的“初始状态”应该是什么。

所以 setup store 默认没有通用的 `$reset()`。

而后台管理项目里，“重置 store”又很常见。

例如：

- 退出登录时重置认证状态。
- 切换用户时清空标签页和权限路由。
- 重新初始化主题配置。

因此这个项目专门给 setup store 补了 `$reset()` 能力。

## 九、`resetSetupStore`：给 setup store 补 `$reset`

当前代码在 [src/store/plugins/index.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/store/plugins/index.ts)：

```ts
import type { PiniaPluginContext } from 'pinia';
import { jsonClone } from '@sa/utils';
import { SetupStoreId } from '@/enum';

export function resetSetupStore(context: PiniaPluginContext) {
  const setupSyntaxIds = Object.values(SetupStoreId) as string[];

  if (setupSyntaxIds.includes(context.store.$id)) {
    const { $state } = context.store;

    const defaultStore = jsonClone($state);

    context.store.$reset = () => {
      context.store.$patch(defaultStore);
    };
  }
}
```

这段逻辑的核心思想是：

```txt
store 第一次创建时，先把当前状态拷贝一份当作初始快照
以后调用 $reset() 时，再把这份初始快照 patch 回去
```

拆开理解：

### 第一步：识别哪些 store 需要处理

```ts
const setupSyntaxIds = Object.values(SetupStoreId) as string[];
```

这里只处理项目里定义的 setup store。

### 第二步：记录初始状态

```ts
const defaultStore = jsonClone($state);
```

为什么要 `jsonClone`？

因为如果直接引用对象，后面状态变化会影响原始对象；而深拷贝后才能保留“创建时的状态快照”。

### 第三步：重写 `$reset`

```ts
context.store.$reset = () => {
  context.store.$patch(defaultStore);
};
```

意思是把当前状态恢复到最初快照。

这就是为什么后面你在 `auth store`、`route store` 里可以放心使用：

```ts
authStore.$reset();
routeStore.$reset();
```

## 十、Pinia store 的阅读方法

看这个项目的 store，不建议按文件顺序一行不漏地啃。更好的方法是按职责读。

推荐顺序：

### 1. `auth store`

先理解：

```txt
登录状态、token、用户信息
```

这是后面权限系统的入口。

### 2. `route store`

再理解：

```txt
权限路由、菜单、缓存路由、面包屑
```

这是后台项目的骨架。

### 3. `tab store`

再理解：

```txt
多标签页系统
```

这决定页面切换体验。

### 4. `theme store`

再理解：

```txt
主题色、暗色模式、布局模式、CSS 变量
```

这决定全局视觉状态。

### 5. `app store`

最后理解：

```txt
语言、移动端适配、全屏内容、侧边栏等应用交互状态
```

## 十一、`auth store`：认证状态中心

`auth store` 位于：

```txt
src/store/modules/auth/index.ts
```

它负责的不是“登录页 UI”，而是**整个项目的认证状态**。

核心状态：

```ts
const token = ref(getToken());

const userInfo: Api.Auth.UserInfo = reactive({
  userId: '',
  userName: '',
  roles: [],
  buttons: []
});
```

这里要区分两件事：

### 1. `token`

表示当前登录凭证。初始值来自本地缓存：

```ts
const token = ref(getToken());
```

而 `getToken()` 在 `shared.ts` 中定义：

```ts
export function getToken() {
  return localStg.get('token') || '';
}
```

说明 token 状态不是只存在内存里，还和本地存储联动。

### 2. `userInfo`

表示当前登录用户的详细信息：

- 用户 id
- 用户名
- 角色
- 按钮权限

这是后面权限路由、按钮权限控制的基础数据。

## 十二、`auth store` 的计算属性

`auth store` 中有两个特别重要的计算属性。

### 1. `isStaticSuper`

```ts
const isStaticSuper = computed(() => {
  const { VITE_AUTH_ROUTE_MODE, VITE_STATIC_SUPER_ROLE } = import.meta.env;

  return VITE_AUTH_ROUTE_MODE === 'static' && userInfo.roles.includes(VITE_STATIC_SUPER_ROLE);
});
```

意思是：

```txt
如果当前是静态权限路由模式，并且用户角色里包含“超级管理员角色”，
那这个用户就拥有静态路由下的超级权限。
```

### 2. `isLogin`

```ts
const isLogin = computed(() => Boolean(token.value));
```

这是最基础的登录判断，很多地方都会依赖它。

## 十三、`auth store` 的核心动作：登录

`login()` 是第一个完整的业务链路：

```ts
async function login(userName: string, password: string, redirect = true) {
  startLoading();

  const { data: loginToken, error } = await fetchLogin(userName, password);

  if (!error) {
    const pass = await loginByToken(loginToken);

    if (pass) {
      const isClear = checkTabClear();
      let needRedirect = redirect;

      if (isClear) {
        needRedirect = false;
      }
      await redirectFromLogin(needRedirect);

      window.$notification?.success({
        title: $t('page.login.common.loginSuccess'),
        message: $t('page.login.common.welcomeBack', { userName: userInfo.userName }),
        duration: 4500
      });
    }
  } else {
    resetStore();
  }

  endLoading();
}
```

你先不要陷入每一行细节，先看主线：

```txt
点击登录
  -> 调登录接口
  -> 拿到 token / refreshToken
  -> 缓存 token
  -> 再请求用户信息
  -> 初始化用户状态
  -> 判断是否要清空 tabs
  -> 登录后跳转
  -> 弹出成功提示
```

这是一个很标准的后台管理登录链路。

## 十四、`auth store` 为什么要依赖别的 store

你会看到：

```ts
const routeStore = useRouteStore();
const tabStore = useTabStore();
```

这说明 store 之间不是完全隔离的。

例如退出登录时：

```ts
async function resetStore() {
  recordUserId();

  clearAuthStorage();

  authStore.$reset();

  if (!route.meta.constant) {
    await toLogin();
  }

  tabStore.cacheTabs();
  routeStore.resetStore();
}
```

这里做了好几件事：

- 记录上次登录用户 id。
- 清理 token / refreshToken。
- 重置认证 store 自身。
- 必要时跳转登录页。
- 缓存当前 tabs。
- 重置路由 store。

说明：

```txt
认证状态变化，不只是 auth store 自己的事，
它会影响路由和标签页。
```

这是后台项目里很典型的“状态联动”。

## 十五、`route store`：权限路由和菜单中心

`route store` 位于：

```txt
src/store/modules/route/index.ts
```

它是这个项目里最复杂的 store 之一。

它负责：

- 常量路由。
- 权限路由。
- 动态添加 Vue Router 路由。
- 生成全局菜单。
- 生成全局搜索菜单。
- 计算缓存路由。
- 计算面包屑。
- 路由重置。

你可以先抓住几个关键状态。

### 1. 路由初始化标记

```ts
const { bool: isInitConstantRoute, setBool: setIsInitConstantRoute } = useBoolean();
const { bool: isInitAuthRoute, setBool: setIsInitAuthRoute } = useBoolean();
```

它们用来标记：

```txt
常量路由初始化过没有
权限路由初始化过没有
```

### 2. 权限模式

```ts
const authRouteMode = ref(import.meta.env.VITE_AUTH_ROUTE_MODE);
```

说明权限路由模式来自环境变量，可以是：

```txt
static
dynamic
```

### 3. 菜单

```ts
const menus = ref<App.Global.Menu[]>([]);
const searchMenus = computed(() => transformMenuToSearchMenus(menus.value));
```

说明菜单不是直接写死在组件里的，而是 route store 统一计算。

### 4. 缓存路由

```ts
const cacheRoutes = ref<RouteKey[]>([]);
const excludeCacheRoutes = ref<RouteKey[]>([]);
```

这会影响 KeepAlive 和路由缓存刷新。

## 十六、`route store` 的主线：初始化路由

最关键的是这两个方法：

```ts
async function initConstantRoute() { ... }
async function initAuthRoute() { ... }
```

理解主线即可。

### 常量路由初始化

```txt
先拿不需要登录也能访问的路由
例如登录页、404、403、根路由等
```

### 权限路由初始化

```txt
用户登录后，再根据权限模式获取可访问路由
静态模式：前端本地生成后按角色过滤
动态模式：向后端请求用户路由
```

最后统一走：

```ts
handleConstantAndAuthRoutes()
```

它会做这些事：

```txt
合并常量路由和权限路由
按 order 排序
转成 Vue Router 需要的路由记录
动态 addRoute 到 router
生成菜单
生成缓存路由
```

这就是为什么 route store 是权限系统的中枢。

## 十七、`route store` 和路由守卫怎么联动

在 [src/router/guard/route.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/router/guard/route.ts) 中：

```ts
const authStore = useAuthStore();
const routeStore = useRouteStore();
```

并且在进入路由前会调用：

```ts
await routeStore.initConstantRoute();
await routeStore.initAuthRoute();
```

这说明：

```txt
路由守卫负责“什么时候初始化路由”
route store 负责“如何初始化路由”
```

这是一种很好的职责分离。

路由守卫决定时机，store 管理状态和数据。

## 十八、`tab store`：多标签页系统

`tab store` 位于：

```txt
src/store/modules/tab/index.ts
```

它负责项目里的多标签页能力：

- 记录当前已打开的 tab。
- 记录首页 tab。
- 记录当前激活 tab。
- 新增 tab。
- 删除 tab。
- 清空左侧 / 右侧 / 其他 tab。
- 更新 tab 文案。
- 缓存 tab 到本地。

核心状态：

```ts
const tabs = ref<App.Global.Tab[]>([]);
const homeTab = ref<App.Global.Tab>();
const activeTabId = ref<string>('');
```

这说明 tab 系统本质上就是一组数据：

```txt
当前有哪些 tab
首页 tab 是谁
当前激活的是哪个 tab
```

## 十九、`tab store` 为什么要依赖 `route store` 和 `theme store`

它一开始就依赖：

```ts
const routeStore = useRouteStore();
const themeStore = useThemeStore();
```

原因很自然。

### 1. 依赖 `route store`

因为 tab 本质上是“路由的视图化表示”。

例如：

```ts
homeTab.value = getDefaultHomeTab(router, routeStore.routeHome);
```

首页 tab 要根据当前首页路由来算。

而且清理 tab 时，可能还要重置某些路由缓存：

```ts
routeStore.resetRouteCache(routeKey);
```

### 2. 依赖 `theme store`

因为 tab 是否缓存由主题设置决定：

```ts
if (themeStore.tab.cache && storageTabs) {
  ...
}
```

也就是说“标签页缓存”虽然作用在 tab 系统上，但配置入口在主题设置里。

这再次说明后台项目里的状态往往是交叉的，store 之间会协作。

## 二十、`theme store`：主题和布局状态中心

`theme store` 位于：

```txt
src/store/modules/theme/index.ts
```

它负责的不只是“主题色”，而是整套视觉状态：

- 主题模式：浅色 / 深色 / 跟随系统。
- 是否灰度模式。
- 是否色弱模式。
- 主色和其他颜色。
- 布局模式。
- 页签模式。
- Header / Tab / Sider / Footer 配置。
- 主题 token 到 CSS 变量的映射。

最核心的状态是：

```ts
const settings: Ref<App.Theme.ThemeSetting> = ref(initThemeSettings());
```

这说明 theme store 的本体就是一份大的主题设置对象。

### 为什么要用一份 `settings`

因为主题系统本身就适合配置化。

例如：

```txt
当前主题方案
当前布局模式
是否显示页签
页签高度
侧边栏宽度
页脚是否显示
水印是否显示
```

这些都像配置项，而不是离散变量，所以适合收拢成统一 settings。

## 二十一、`theme store` 里为什么有那么多 `computed`

例如：

```ts
const darkMode = computed(() => { ... });
const grayscaleMode = computed(() => settings.value.grayscale);
const colourWeaknessMode = computed(() => settings.value.colourWeakness);
const themeColors = computed(() => { ... });
const uiTheme = computed(() => getNaiveTheme(themeColors.value, settings.value.recommendColor));
```

这是因为 theme store 里很多状态不是直接存的，而是从 `settings` 推导出来的。

例如：

```txt
themeScheme = auto
+ 当前系统主题 = dark
=> darkMode = true
```

所以 store 不只是“存数据”，还负责把原始配置推导成可直接使用的状态。

## 二十二、`theme store` 的关键能力：监听并同步到全局环境

`theme store` 里最值得学的是一组 `watch`：

```ts
watch(
  darkMode,
  val => {
    toggleCssDarkMode(val);
    localStg.set('darkMode', val);
  },
  { immediate: true }
);

watch(
  [grayscaleMode, colourWeaknessMode],
  val => {
    toggleAuxiliaryColorModes(val[0], val[1]);
  },
  { immediate: true }
);

watch(
  themeColors,
  val => {
    setupThemeVarsToGlobal();
    localStg.set('themeColor', val.primary);
  },
  { immediate: true }
);
```

这说明 theme store 的职责不只是“保存状态”，还包括：

```txt
监听状态变化
  -> 改 html 上的 dark class
  -> 写入 localStorage
  -> 更新 CSS 变量
  -> 更新辅助模式样式
```

这是一种很典型的做法：

```txt
store 负责管理状态和副作用同步
组件只负责触发修改
```

这样业务组件就不会到处散落：

```ts
document.documentElement.classList...
localStorage.setItem(...)
```

## 二十三、为什么 `theme store` 用了 `effectScope`

你会看到：

```ts
const scope = effectScope();

scope.run(() => {
  watch(...)
  watch(...)
  watch(...)
});

onScopeDispose(() => {
  scope.stop();
});
```

这表示项目把一组 watcher 放进一个响应式作用域里管理。

这样做的好处是：

- watcher 组织更集中。
- 需要清理时可以统一 stop。
- 结构比到处散写 watch 更清晰。

在 store 逻辑复杂、watch 较多时，这种写法是很实用的。

## 二十四、`app store`：应用级交互状态中心

`app store` 位于：

```txt
src/store/modules/app/index.ts
```

它处理的是更接近“应用交互层”的状态：

- 当前语言。
- 是否移动端。
- 页面 reload 标记。
- 主题抽屉是否打开。
- 内容区是否全屏。
- 内容区是否横向滚动。
- 侧边栏是否折叠。
- 混合菜单侧边栏是否固定。

这类状态和认证、权限、主题不完全一样，但又是全应用共享的，所以单独放到 `app store` 很合理。

## 二十五、`app store` 为什么和 `theme` / `route` / `tab` 联动

开头依赖：

```ts
const themeStore = useThemeStore();
const routeStore = useRouteStore();
const tabStore = useTabStore();
```

例如语言切换时：

```ts
watch(locale, () => {
  updateDocumentTitleByLocale();
  routeStore.updateGlobalMenusByLocale();
  tabStore.updateTabsByLocale();
  setDayjsLocale(locale.value);
});
```

这个链路非常有代表性。

语言变化不是只改一个变量，它还会引发：

```txt
文档标题更新
菜单文案更新
tab 文案更新
dayjs 语言更新
```

所以语言状态放在 app store 中，由它统一协调这些副作用，非常合理。

再例如移动端适配：

```ts
watch(
  isMobile,
  newValue => {
    if (newValue) {
      localStg.set('backupThemeSettingBeforeIsMobile', {
        layout: themeStore.layout.mode,
        siderCollapse: siderCollapse.value
      });

      themeStore.setThemeLayout('vertical');
      setSiderCollapse(true);
    } else {
      ...
    }
  },
  { immediate: true }
);
```

说明：

```txt
“是否移动端”属于 app 级状态，
但它会反过来影响 theme store 的布局模式。
```

## 二十六、store 之间的真实依赖关系

你现在可以把五个 store 的关系先画成这样：

```txt
auth store
  -> 依赖 route store / tab store

route store
  -> 依赖 auth store / tab store

tab store
  -> 依赖 route store / theme store

theme store
  -> 主要独立，但会影响全局样式和缓存

app store
  -> 依赖 theme store / route store / tab store
```

注意，这里并不是鼓励你随便让 store 互相引用。

而是这个项目已经存在这样一套业务链路，你要学会识别：

```txt
谁是核心状态源
谁在协调副作用
谁在消费别人的状态
```

如果以后你自己搭建项目，store 之间依赖也要控制好，避免形成难以维护的循环关系。

## 二十七、这些 store 是在哪里被实际使用的

### 1. 路由守卫中

例如：

```ts
const authStore = useAuthStore();
const routeStore = useRouteStore();
```

用于：

- 初始化路由。
- 判断登录状态。
- 判断是否有权限。

### 2. 页面和布局组件中

用于：

- 显示当前语言。
- 控制侧边栏折叠。
- 显示当前 tab。
- 切换主题。

### 3. 初始化逻辑中

例如应用启动后，需要：

- 初始化常量路由。
- 初始化用户信息。
- 初始化主题样式。

这些都是通过 store 承接的。

## 二十八、本课最关键的工程思想

学习到这里，你要抓住三件事。

### 1. store 不是“变量仓库”

它不只是存：

```ts
const token = ref('')
```

它还负责：

- 计算派生状态。
- 封装动作。
- 处理联动副作用。
- 协调本地缓存。
- 协调路由和 UI。

### 2. store 是业务主线的承接层

例如登录链路：

```txt
登录页组件触发 login()
  -> auth store 更新 token / userInfo
  -> route store 初始化权限路由
  -> tab store 初始化首页标签
  -> 页面跳转
```

如果没有 store，这条链路会散落在多个组件和工具函数中，很难维护。

### 3. setup store 很适合 Vue 3 风格项目

因为它天然能和：

```txt
ref / computed / watch / useXxx composable
```

结合在一起。

这个仓库选择 setup store 是很合理的。

## 二十九、课堂演示

### 演示 1：从 Pinia 注册入口出发

打开：

```txt
src/store/index.ts
```

回答：

- `createPinia()` 做了什么？
- `store.use(resetSetupStore)` 做了什么？
- `app.use(store)` 为什么必须执行？

### 演示 2：解释 `$reset` 是怎么来的

打开：

```txt
src/store/plugins/index.ts
```

再打开：

```txt
src/store/modules/auth/index.ts
```

找到：

```ts
authStore.$reset();
```

解释：

```txt
setup store 默认没有 $reset
项目通过 Pinia 插件把它补上
```

### 演示 3：画出登录后的状态链路

阅读：

```txt
src/store/modules/auth/index.ts
```

整理：

```txt
login
  -> fetchLogin
  -> loginByToken
  -> localStg.set(token)
  -> getUserInfo
  -> Object.assign(userInfo, info)
  -> redirectFromLogin
```

### 演示 4：画出路由初始化链路

阅读：

```txt
src/router/guard/route.ts
src/store/modules/route/index.ts
```

整理：

```txt
beforeEach
  -> initRoute(to)
  -> routeStore.initConstantRoute()
  -> routeStore.initAuthRoute()
  -> handleConstantAndAuthRoutes()
  -> addRoute
  -> menus / cacheRoutes 更新
```

### 演示 5：观察主题状态是如何影响全局样式的

阅读：

```txt
src/store/modules/theme/index.ts
src/store/modules/theme/shared.ts
```

重点解释：

```txt
watch(darkMode)
watch(themeColors)
```

为什么 store 改动能最终影响到：

- `html.dark`
- CSS 变量
- localStorage

## 三十、本课作业

### 作业 1：整理五个 store 的职责

写一份笔记：

```md
# Pinia store 职责表

- app store：
- auth store：
- route store：
- tab store：
- theme store：
```

每个 store 用 3-5 句话说明它负责什么。

### 作业 2：解释 setup store 为什么更适合这个项目

回答：

```md
# setup store 优势

- 为什么这个项目不用 option store：
- setup store 和 Composition API 的关系：
- 哪些 store 中大量使用了 watch / composable：
```

### 作业 3：解释 `$reset` 插件

阅读：

```txt
src/store/plugins/index.ts
```

然后用自己的话回答：

```md
# resetSetupStore 分析

- 为什么需要它：
- 它处理了哪些 store：
- 它如何保存初始状态：
- 它如何重置状态：
```

### 作业 4：找一个 store 的完整调用链

建议选择：

- `auth store`
- `route store`

格式：

```md
# store 调用链

- 入口文件：
- 暴露的方法：
- 谁调用了它：
- 调用后影响了哪些状态：
- 进一步影响了哪些模块：
```

## 三十一、自检问题

完成本课后，尝试回答：

- Pinia 在这个项目中解决什么问题？
- `src/store/index.ts` 做了哪三件事？
- setup store 和 option store 的区别是什么？
- 为什么这个项目更适合 setup store？
- `SetupStoreId` 的作用是什么？
- setup store 默认为什么没有 `$reset`？
- `resetSetupStore` 是如何工作的？
- `auth store` 管理哪些核心状态？
- 登录后为什么还要请求用户信息？
- `route store` 为什么是权限系统中枢？
- `route store` 为什么要记录 `menus` 和 `cacheRoutes`？
- `tab store` 和 `route store` 是什么关系？
- `theme store` 为什么有那么多 watch？
- `app store` 为什么要协调语言、移动端和侧边栏状态？
- store 和 localStorage、router、CSS 变量是如何联动的？

如果这些问题能回答出来，就可以进入第 10 课。

## 三十二、下一课预告

第 10 课会继续进入登录、请求与状态管理主线，重点学习：

- 登录流程如何和请求层配合。
- token、refreshToken 如何存储和更新。
- 请求拦截器如何注入 token。
- 401 / token 过期时如何处理。
- 为什么后台项目经常要做刷新 token。

第 9 课理解的是“状态放在哪里、如何共享”，第 10 课会进一步理解“这些状态如何和请求系统联动”。
