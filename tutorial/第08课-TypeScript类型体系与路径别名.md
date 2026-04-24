# 第 8 课：TypeScript 类型体系与路径别名

## 课程定位

前面几课已经完成了项目认知、开发环境、启动链路、Vite 配置和插件系统。本课开始进入一个后台管理项目非常重要的基础能力：**TypeScript 类型体系**。

后台管理项目的数据很多：

- 登录返回 Token。
- 用户信息包含角色和按钮权限。
- 用户列表、角色列表、菜单列表都有分页结构。
- 路由 meta 会影响菜单、标签页、缓存和权限。
- 主题配置会影响布局、颜色、暗色模式。
- 环境变量会影响接口地址、路由模式、图标前缀和构建行为。

如果这些数据完全不加类型约束，项目越大越容易出错。这个项目通过 `src/typings`、`tsconfig.json`、接口泛型和命名空间，把很多关键数据结构固定下来，让编辑器能够提前提示错误。

本课的目标不是把 TypeScript 所有语法都学完，而是先看懂这个项目的类型组织方式。你要知道类型从哪里来、在哪里生效、怎么被业务代码使用。

## 学习目标

学完本课后，你应该能够：

- 理解 `tsconfig.json` 在项目中的作用。
- 说明 `@` 和 `~` 路径别名分别指向哪里。
- 理解为什么 Vite alias 和 TypeScript paths 都要配置。
- 理解 `src/typings` 目录的职责。
- 区分 `.d.ts` 类型声明文件和普通 `.ts` 文件。
- 理解 `Api.*`、`App.*`、`Env.*`、`CommonType.*`、`UnionKey.*` 等命名空间。
- 看懂接口请求函数里的泛型返回类型。
- 理解 `declare global` 和 `declare module` 的作用。
- 知道如何给一个新接口补充类型。
- 能够使用 `pnpm typecheck` 检查类型错误。

## 建议课时

建议用 90-120 分钟完成本课：

- 15 分钟：阅读 `tsconfig.json`。
- 15 分钟：理解 `@` 和 `~` 路径别名。
- 20 分钟：浏览 `src/typings` 目录。
- 20 分钟：理解 `Api.*` 接口类型。
- 15 分钟：理解 `App.*`、`Env.*`、`CommonType.*`、`UnionKey.*`。
- 15 分钟：理解全局类型声明和模块增强。
- 20 分钟：完成接口类型补充练习。

## 一、为什么后台管理项目需要类型体系

后台管理项目的核心不是炫酷页面，而是大量稳定的数据流：

```txt
页面表单
  -> 请求参数
  -> 后端接口
  -> 响应数据
  -> 表格渲染
  -> 编辑弹窗
  -> 再次提交
```

这条链路上的字段如果写错，很容易出现问题。

例如用户列表接口返回用户数据，项目中定义了：

```ts
type User = Common.CommonRecord<{
  userName: string;
  userGender: UserGender | undefined;
  nickName: string;
  userPhone: string;
  userEmail: string;
  userRoles: string[];
}>;
```

如果你在页面里误写成：

```ts
row.username
```

而真实字段是：

```ts
row.userName
```

TypeScript 就可以提前提示错误。

没有类型时，这类错误通常要等到页面运行时才发现。后台管理系统字段多、表单多、接口多，所以类型体系非常重要。

## 二、本项目的类型体系分成几类

这个项目的类型大致可以分成五类。

### 1. TypeScript 编译配置

主要看：

```txt
tsconfig.json
```

它决定：

- TypeScript 用什么编译目标。
- 支持哪些运行环境类型。
- 路径别名如何识别。
- 是否开启严格模式。
- 哪些文件参与类型检查。

### 2. 环境变量类型

主要看：

```txt
src/typings/vite-env.d.ts
```

它定义：

```ts
import.meta.env.VITE_APP_TITLE
import.meta.env.VITE_SERVICE_BASE_URL
import.meta.env.VITE_AUTH_ROUTE_MODE
```

这些环境变量分别是什么类型。

### 3. 后端接口类型

主要看：

```txt
src/typings/api
```

例如：

```txt
src/typings/api/auth.d.ts
src/typings/api/common.d.ts
src/typings/api/system-manage.d.ts
src/typings/api/route.d.ts
```

它们定义 `Api.*` 命名空间。

例如：

```ts
Api.Auth.LoginToken
Api.Auth.UserInfo
Api.SystemManage.User
Api.SystemManage.UserList
```

### 4. 应用内部类型

主要看：

```txt
src/typings/app.d.ts
src/typings/router.d.ts
src/typings/storage.d.ts
src/typings/ui.d.ts
```

它们定义：

- 主题配置类型。
- 菜单类型。
- 标签页类型。
- 路由 meta 类型。
- 本地存储类型。
- 表格列类型。

### 5. 第三方库补充声明

主要看：

```txt
src/typings/global.d.ts
src/typings/package.d.ts
```

它们做的事情包括：

- 给 `window` 增加全局属性类型。
- 声明构建时全局常量。
- 给某些第三方包或全局变量补类型。

## 三、先看 `tsconfig.json`

当前项目的 `tsconfig.json` 核心配置是：

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "jsx": "preserve",
    "jsxImportSource": "vue",
    "lib": ["DOM", "ESNext"],
    "baseUrl": ".",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "paths": {
      "@/*": ["./src/*"],
      "~/*": ["./*"]
    },
    "resolveJsonModule": true,
    "types": ["vite/client", "node", "unplugin-icons/types/vue", "element-plus/global"],
    "strict": true,
    "strictNullChecks": true,
    "noUnusedLocals": false,
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true
  },
  "include": ["./**/*.ts", "./**/*.tsx", "./**/*.vue"],
  "exclude": ["node_modules", "dist"]
}
```

这份配置告诉 TypeScript：这个项目是一个现代 Vue + Vite 项目，源码里会出现 `.ts`、`.tsx`、`.vue`，并且需要识别 Vite、Node、Icon 组件和 Element Plus 全局组件类型。

## 四、`target: "ESNext"` 是什么

```json
"target": "ESNext"
```

表示 TypeScript 按现代 JavaScript 语法目标来理解代码。

在 Vite 项目中，很多最终转译和打包工作由 Vite / Rolldown / esbuild 处理。TypeScript 更多负责类型检查，所以这里使用比较新的目标。

你可以先理解为：

```txt
TypeScript 不把代码强行降级成很老的 JavaScript。
```

## 五、`lib: ["DOM", "ESNext"]` 是什么

```json
"lib": ["DOM", "ESNext"]
```

表示项目可以使用这些环境里的类型。

`DOM` 提供浏览器相关类型：

```ts
window
document
HTMLElement
MouseEvent
localStorage
```

`ESNext` 提供现代 JavaScript 类型：

```ts
Promise
Map
Set
Array.prototype.at
```

如果没有 `DOM`，你写：

```ts
document.querySelector('#app')
```

TypeScript 可能就不知道 `document` 是什么。

## 六、路径别名：`@` 和 `~`

本项目配置了两个路径别名：

```json
"paths": {
  "@/*": ["./src/*"],
  "~/*": ["./*"]
}
```

含义是：

```txt
@  指向 src 目录
~  指向项目根目录
```

所以：

```ts
import { localStg } from '@/utils/storage';
```

等价于：

```ts
import { localStg } from './src/utils/storage';
```

而：

```ts
import { setupVitePlugins } from '~/build/plugins';
```

可以理解为从项目根目录开始找：

```txt
build/plugins
```

不过当前项目里更常用的是 `@`，因为业务代码主要在 `src` 下。

## 七、为什么 Vite 和 TypeScript 都要配置别名

`tsconfig.json` 里配置了：

```json
"paths": {
  "@/*": ["./src/*"],
  "~/*": ["./*"]
}
```

同时 `vite.config.ts` 里也配置了：

```ts
resolve: {
  alias: {
    '~': fileURLToPath(new URL('./', import.meta.url)),
    '@': fileURLToPath(new URL('./src', import.meta.url))
  }
}
```

这两份配置解决的是不同问题。

`tsconfig.json` 的 `paths` 是给 TypeScript 和编辑器看的：

```txt
让 VS Code / vue-tsc 知道 @/utils/storage 指向哪里。
```

`vite.config.ts` 的 `resolve.alias` 是给 Vite 构建和开发服务器看的：

```txt
让 Vite 运行项目时知道 @/utils/storage 应该加载哪个真实文件。
```

如果只配 TypeScript，不配 Vite：

```txt
编辑器不报错，但项目运行时可能找不到模块。
```

如果只配 Vite，不配 TypeScript：

```txt
项目可能能运行，但编辑器和类型检查会报找不到模块。
```

所以两边都要配置，并且指向要保持一致。

## 八、`@` 和 `~` 应该怎么用

建议：

```txt
src 内部业务代码优先用 @。
根目录级配置、构建脚本需要时再用 ~。
```

常见写法：

```ts
import { $t } from '@/locales';
import { localStg } from '@/utils/storage';
import { useThemeStore } from '@/store/modules/theme';
```

不建议在业务代码里大量写很长的相对路径：

```ts
import { localStg } from '../../../../utils/storage';
```

原因是：

- 文件移动后容易出错。
- 阅读成本高。
- 不容易看出模块来自哪里。

但同目录或相邻目录的小模块，继续用相对路径也可以。

例如：

```ts
import { request } from '../request';
```

这在 `src/service/api/auth.ts` 里是合理的，因为 `api` 和 `request` 属于同一块 service 模块。

## 九、`types` 配置是什么意思

`tsconfig.json` 中有：

```json
"types": ["vite/client", "node", "unplugin-icons/types/vue", "element-plus/global"]
```

这些是项目主动引入的全局类型。

### 1. `vite/client`

让 TypeScript 识别 Vite 特有能力。

例如：

```ts
import.meta.env
```

没有它，TypeScript 不一定知道 `import.meta.env` 是什么。

### 2. `node`

让 TypeScript 识别 Node.js 类型。

例如配置文件中使用：

```ts
import process from 'node:process';
import path from 'node:path';
```

这些类型来自 Node。

### 3. `unplugin-icons/types/vue`

让 TypeScript / Vue 能识别图标组件类型。

项目中有很多：

```vue
<icon-mdi-refresh />
<icon-ic-round-plus />
```

这类组件不是手写的 `.vue` 文件，而是插件自动生成的。这个类型声明可以减少编辑器报错。

### 4. `element-plus/global`

让 Element Plus 的全局组件在模板中有更好的类型支持。

例如：

```vue
<ElButton />
<ElCard />
<ElInput />
```

虽然项目通过插件全局注册了 Element Plus，但 TypeScript 也需要知道这些组件类型。

## 十、`include` 为什么能让 `.d.ts` 自动生效

配置里有：

```json
"include": ["./**/*.ts", "./**/*.tsx", "./**/*.vue"]
```

`.d.ts` 文件也属于 TypeScript 文件，所以会被包含进类型检查范围。

这意味着你不需要在业务代码里写：

```ts
import '@/typings/api/system-manage';
```

只要 `.d.ts` 在 `include` 覆盖范围内，里面的全局声明就会生效。

这也是为什么你可以在代码里直接写：

```ts
Api.SystemManage.UserList
```

而不用先 import `Api`。

## 十一、`.d.ts` 和 `.ts` 的区别

`.d.ts` 是类型声明文件。

它通常只告诉 TypeScript：

```txt
某个类型存在
某个全局变量存在
某个模块存在
某个库的类型应该如何补充
```

它不会写业务运行逻辑。

例如：

```ts
declare namespace Api {
  namespace Auth {
    interface LoginToken {
      token: string;
      refreshToken: string;
    }
  }
}
```

这只是声明类型，不会在浏览器里真的创建一个 `Api` 对象。

普通 `.ts` 文件则可以包含运行时代码。

例如：

```ts
export function fetchLogin(userName: string, password: string) {
  return request<Api.Auth.LoginToken>({
    url: '/auth/login',
    method: 'post',
    data: {
      userName,
      password
    }
  });
}
```

这个函数会参与打包，因为它是运行时代码。

简单区分：

```txt
.d.ts：只给 TypeScript 看，不产生运行时代码。
.ts：既可以写类型，也可以写运行时代码，可能参与打包。
```

## 十二、为什么有的类型写在 `.ts` 文件中

你之前问过 `src/service/request/type.ts` 为什么类型写在 `.ts` 文件中。

当前文件是：

```ts
export interface RequestInstanceState {
  /** the promise of refreshing token */
  refreshTokenPromise: Promise<boolean> | null;
  /** the request error message stack */
  errMsgStack: string[];
  [key: string]: unknown;
}
```

它用的是 `export interface`，说明这是一个模块内导出的类型。

这种类型适合写在 `.ts` 文件里：

- 它属于 `src/service/request` 模块内部。
- 不是全局类型。
- 需要被其他文件按需导入。

如果其他地方只做类型导入：

```ts
import type { RequestInstanceState } from './type';
```

最终构建时不会因为这个类型生成多余的业务代码。

所以：

```txt
全局共享、无需导入的类型，适合放 .d.ts。
模块内部导出的类型，适合放普通 .ts。
```

## 十三、`src/typings` 目录总览

当前 `src/typings` 大致包含：

```txt
src/typings
├── api
│   ├── auth.d.ts
│   ├── common.d.ts
│   ├── route.d.ts
│   └── system-manage.d.ts
├── app.d.ts
├── common.d.ts
├── components.d.ts
├── elegant-router.d.ts
├── global.d.ts
├── package.d.ts
├── router.d.ts
├── storage.d.ts
├── ui.d.ts
├── union-key.d.ts
└── vite-env.d.ts
```

你可以先把它分成几组：

```txt
api/*                 后端接口类型
app.d.ts              应用核心类型
vite-env.d.ts         环境变量类型
router.d.ts           路由 meta 类型增强
storage.d.ts          本地存储类型
ui.d.ts               UI 组件相关类型
global.d.ts           全局变量类型
package.d.ts          第三方包或全局库补充类型
elegant-router.d.ts   路由生成器生成的类型
components.d.ts       自动组件类型
```

阅读时不要从头到尾硬啃。更好的方式是：遇到某个类型，再跳过去看它。

## 十四、`Api.*`：后端接口类型

接口类型集中在：

```txt
src/typings/api
```

### 1. 公共分页结构

`src/typings/api/common.d.ts` 中定义了：

```ts
declare namespace Api {
  namespace Common {
    interface PaginatingCommonParams {
      current: number;
      size: number;
      total: number;
    }

    interface PaginatingQueryRecord<T = any> extends PaginatingCommonParams {
      records: T[];
    }
  }
}
```

这个结构表达的是后台常见分页返回：

```ts
{
  current: 1,
  size: 10,
  total: 100,
  records: []
}
```

其中 `records` 的具体元素类型由泛型 `T` 决定。

例如：

```ts
type UserList = Common.PaginatingQueryRecord<User>;
```

可以理解为：

```txt
用户列表 = 分页结构 + records 里放 User[]
```

### 2. 登录接口类型

`src/typings/api/auth.d.ts` 中定义：

```ts
declare namespace Api {
  namespace Auth {
    interface LoginToken {
      token: string;
      refreshToken: string;
    }

    interface UserInfo {
      userId: string;
      userName: string;
      roles: string[];
      buttons: string[];
    }
  }
}
```

对应接口函数：

```ts
export function fetchLogin(userName: string, password: string) {
  return request<Api.Auth.LoginToken>({
    url: '/auth/login',
    method: 'post',
    data: {
      userName,
      password
    }
  });
}
```

这里的重点是：

```ts
request<Api.Auth.LoginToken>
```

它告诉请求函数：

```txt
这个接口成功后返回的数据应该符合 LoginToken 类型。
```

所以后面使用登录结果时，编辑器知道结果里有：

```ts
token
refreshToken
```

### 3. 系统管理接口类型

`src/typings/api/system-manage.d.ts` 中定义了角色、用户、菜单等类型。

例如用户类型：

```ts
type User = Common.CommonRecord<{
  userName: string;
  userGender: UserGender | undefined;
  nickName: string;
  userPhone: string;
  userEmail: string;
  userRoles: string[];
}>;
```

这里用了组合类型：

```ts
Common.CommonRecord<...>
```

`CommonRecord` 提供通用字段：

```ts
id
createBy
createTime
updateBy
updateTime
status
```

用户自己的字段是：

```ts
userName
userGender
nickName
userPhone
userEmail
userRoles
```

组合起来就是完整的用户记录。

对应接口函数：

```ts
export function fetchGetUserList(params?: Api.SystemManage.UserSearchParams) {
  return request<Api.SystemManage.UserList>({
    url: '/systemManage/getUserList',
    method: 'get',
    params
  });
}
```

这里有两层类型约束：

```ts
params?: Api.SystemManage.UserSearchParams
```

约束请求参数。

```ts
request<Api.SystemManage.UserList>
```

约束响应数据。

这就是后台项目中最常见的接口类型写法。

## 十五、`CommonType.*`：通用工具类型

`src/typings/common.d.ts` 中定义：

```ts
declare namespace CommonType {
  interface StrategicPattern {
    condition: boolean;
    callback: () => void;
  }

  type Option<K = string, M = string> = { value: K; label: M };

  type YesOrNo = 'Y' | 'N';

  type RecordNullable<T> = {
    [K in keyof T]?: T[K] | undefined;
  };
}
```

这些不是某个业务独有的类型，而是项目中经常复用的小工具类型。

例如：

```ts
type YesOrNo = 'Y' | 'N';
```

用于环境变量和开关配置：

```ts
readonly VITE_HTTP_PROXY?: CommonType.YesOrNo;
```

这意味着：

```txt
VITE_HTTP_PROXY 只能是 'Y' 或 'N'
```

再看：

```ts
type RecordNullable<T> = {
  [K in keyof T]?: T[K] | undefined;
};
```

它的作用是把一个对象类型的所有字段变成可选。

例如：

```ts
type UserSearchParams = CommonType.RecordNullable<
  Pick<Api.SystemManage.User, 'userName' | 'userGender' | 'nickName'>
>;
```

意思是搜索条件里这些字段都可以不传。

## 十六、`UnionKey.*`：固定可选值

`src/typings/union-key.d.ts` 里定义了很多联合类型。

例如：

```ts
type ThemeScheme = 'light' | 'dark' | 'auto';
```

表示主题模式只能是：

```txt
light
dark
auto
```

不能写成：

```ts
'black'
```

再例如：

```ts
type ThemeLayoutMode = 'vertical' | 'horizontal' | 'vertical-mix' | 'horizontal-mix';
```

表示布局模式只能从这几个值里选。

后台项目里很多配置都适合用联合类型：

- 登录模块。
- 主题模式。
- 布局模式。
- 标签页模式。
- 动画模式。
- 菜单类型。
- 图标类型。
- 启用状态。

联合类型的价值是：**把字符串变成受约束的字符串**。

没有类型时：

```ts
const mode = 'vertcial';
```

拼错了也不一定能马上发现。

有类型后：

```ts
const mode: UnionKey.ThemeLayoutMode = 'vertcial';
```

TypeScript 会提示错误，因为正确值是：

```txt
vertical
```

## 十七、`App.*`：应用内部核心类型

`src/typings/app.d.ts` 定义的是应用内部数据结构。

例如：

```ts
declare namespace App {
  namespace Theme {
    interface ThemeSetting {
      themeScheme: UnionKey.ThemeScheme;
      grayscale: boolean;
      colourWeakness: boolean;
      recommendColor: boolean;
      themeColor: string;
      otherColor: OtherColor;
      layout: {
        mode: UnionKey.ThemeLayoutMode;
        scrollMode: UnionKey.ThemeScrollMode;
        reverseHorizontalMix: boolean;
      };
    }
  }
}
```

这个类型约束了主题配置对象。

所以 `src/theme/settings.ts` 或主题 store 中处理配置时，可以知道：

```ts
settings.value.layout.mode
```

只能是布局模式里的某一个值。

`App.Global` 里还有菜单和标签页类型。

例如菜单类型：

```ts
type Menu = {
  key: string;
  label: string;
  i18nKey?: I18n.I18nKey | null;
  routeKey: RouteKey;
  routePath: RoutePath;
  icon?: () => VNode;
  children?: Menu[];
};
```

这说明项目里的菜单不是随便一个对象，而是需要包含：

- 菜单 key。
- 菜单显示文本。
- 路由 key。
- 路由 path。
- 图标。
- 子菜单。

后面学习菜单和权限时，这些类型会反复出现。

## 十八、`Env.*`：环境变量类型

`src/typings/vite-env.d.ts` 中定义：

```ts
declare namespace Env {
  type RouterHistoryMode = 'hash' | 'history' | 'memory';

  interface ImportMeta extends ImportMetaEnv {
    readonly VITE_BASE_URL: string;
    readonly VITE_APP_TITLE: string;
    readonly VITE_APP_DESC: string;
    readonly VITE_ROUTER_HISTORY_MODE?: RouterHistoryMode;
    readonly VITE_ICON_PREFIX: 'icon';
    readonly VITE_ICON_LOCAL_PREFIX: 'icon-local';
    readonly VITE_SERVICE_BASE_URL: string;
    readonly VITE_AUTH_ROUTE_MODE: 'static' | 'dynamic';
  }
}

interface ImportMeta {
  readonly env: Env.ImportMeta;
}
```

这让项目中使用：

```ts
import.meta.env.VITE_APP_TITLE
```

时可以获得类型提示。

例如 `vite.config.ts` 中：

```ts
const viteEnv = loadEnv(configEnv.mode, process.cwd()) as unknown as Env.ImportMeta;
```

这是把 Vite 读取出来的环境变量对象断言成项目定义的环境变量类型。

为什么要这样做？

因为 Vite 原始读取到的环境变量大多是字符串，TypeScript 不知道项目具体有哪些变量。通过 `Env.ImportMeta`，项目把环境变量清单固定下来。

## 十九、`router.d.ts`：模块增强

`src/typings/router.d.ts` 中有：

```ts
import 'vue-router';

declare module 'vue-router' {
  interface RouteMeta {
    title: string;
    i18nKey?: App.I18n.I18nKey | null;
    roles?: string[];
    keepAlive?: boolean | undefined;
    constant?: boolean | undefined;
    icon?: string;
    localIcon?: string;
    order?: number | undefined;
    hideInMenu?: boolean | undefined;
  }
}
```

这是 TypeScript 的模块增强。

它的意思是：

```txt
在 vue-router 原本的 RouteMeta 类型基础上，补充本项目自己的 meta 字段。
```

Vue Router 本身不知道你的项目会使用：

```ts
meta.title
meta.i18nKey
meta.roles
meta.keepAlive
meta.icon
```

所以项目通过 `declare module 'vue-router'` 告诉 TypeScript：

```txt
这些字段是合法的。
```

后面学习路由、菜单、权限时，你会发现 `RouteMeta` 是非常关键的类型。

## 二十、`global.d.ts`：全局变量声明

`src/typings/global.d.ts` 中有：

```ts
export {};

declare global {
  export interface Window {
    NProgress?: import('nprogress').NProgress;
    $messageBox?: import('element-plus').IElMessageBox;
    $message?: import('element-plus').Message;
    $notification?: import('element-plus').Notify;
  }

  export const BUILD_TIME: string;
}
```

它做了两件事。

第一，给 `window` 增加属性类型。

项目中 `setupNProgress()` 会执行：

```ts
window.NProgress = NProgress;
```

如果不声明 `Window.NProgress`，TypeScript 可能会提示：

```txt
Property 'NProgress' does not exist on type 'Window'
```

所以这里补充：

```ts
NProgress?: import('nprogress').NProgress;
```

第二，声明全局常量：

```ts
export const BUILD_TIME: string;
```

项目在 `vite.config.ts` 中通过：

```ts
define: {
  BUILD_TIME: JSON.stringify(buildTime)
}
```

注入了构建时间。

TypeScript 本身不知道有 `BUILD_TIME` 这个全局常量，所以需要在 `global.d.ts` 中声明。

注意：

```ts
export {};
```

这行让当前 `.d.ts` 文件成为一个模块，然后再通过 `declare global` 扩展全局类型。这是常见写法。

## 二十一、`package.d.ts`：第三方库补充声明

`src/typings/package.d.ts` 用来补充某些第三方库或全局对象的类型。

例如：

```ts
/// <reference types="@amap/amap-jsapi-types" />
/// <reference types="bmapgl" />
```

表示引入高德地图、百度地图相关类型。

文件里还有：

```ts
declare namespace BMap {
  class Map extends BMapGL.Map {}
  class Point extends BMapGL.Point {}
}
```

这是给地图相关全局对象补类型。

还定义了：

```ts
declare const TMap: any;
```

表示项目里可能会使用腾讯地图的全局变量 `TMap`。

为什么有时候要手动补声明？

因为不是所有第三方库都天然提供完整的 TypeScript 类型，也不是所有虚拟模块、CSS 副作用导入都能被 TypeScript 自动识别。遇到这种情况，就可以通过 `.d.ts` 补充声明。

## 二十二、`components.d.ts`：自动生成的组件类型

`src/typings/components.d.ts` 是自动生成文件。

在 `build/plugins/unplugin.ts` 中有：

```ts
Components({
  dts: 'src/typings/components.d.ts',
  types: [{ from: 'vue-router', names: ['RouterLink', 'RouterView'] }],
  resolvers: [
    ElementPlusResolver({
      importStyle: false
    }),
    IconsResolver({ customCollections: [collectionName], componentPrefix: VITE_ICON_PREFIX })
  ]
})
```

这说明项目使用 `unplugin-vue-components` 自动导入组件，并把组件类型声明生成到：

```txt
src/typings/components.d.ts
```

所以你在 Vue 模板中可以直接写：

```vue
<ElButton />
<RouterView />
<icon-mdi-refresh />
```

而不需要每个组件都手动 import。

这个文件一般不要手动改，因为它可能会被插件重新生成。

## 二十三、`elegant-router.d.ts`：路由生成类型

`src/typings/elegant-router.d.ts` 也是生成文件。

开头写着：

```ts
// Generated by elegant-router
```

它定义了：

```ts
export type RouteMap = {
  "home": "/home";
  "manage_user": "/manage/user";
  "manage_user-detail": "/manage/user-detail/:id";
};

export type RouteKey = keyof RouteMap;
export type RoutePath = RouteMap[RouteKey];
```

这些类型的价值是：把路由名称和路径也纳入类型系统。

例如：

```ts
type RouteKey = 'home' | 'manage_user' | 'manage_role' | ...
```

这样项目里引用路由 key 时，如果写错，就有机会被 TypeScript 提前发现。

后面学习 Elegant Router 和权限路由时，会继续深入看这个文件。

## 二十四、接口类型如何约束请求函数

以用户列表为例：

```ts
export function fetchGetUserList(params?: Api.SystemManage.UserSearchParams) {
  return request<Api.SystemManage.UserList>({
    url: '/systemManage/getUserList',
    method: 'get',
    params
  });
}
```

这里有两个关键点。

第一个：

```ts
params?: Api.SystemManage.UserSearchParams
```

表示参数可传可不传。如果传，就要符合用户查询参数类型。

例如可以传：

```ts
fetchGetUserList({
  current: 1,
  size: 10,
  userName: 'admin'
});
```

如果你传了不存在的字段，TypeScript 有机会提示。

第二个：

```ts
request<Api.SystemManage.UserList>
```

表示响应数据类型是用户分页列表。

`UserList` 本质上是：

```ts
type UserList = Common.PaginatingQueryRecord<User>;
```

所以页面拿到数据后，理论上可以知道：

```ts
data.current
data.size
data.total
data.records
```

并且 `records` 中每一项是 `User`。

## 二十五、如何给一个新接口补类型

假设你要新增一个部门列表接口：

```txt
GET /systemManage/getDeptList
```

返回数据大概是：

```ts
[
  {
    id: 1,
    deptName: '研发部',
    deptCode: 'RD',
    parentId: 0
  }
]
```

可以按三步做。

### 第一步：在 API 类型文件中补充类型

可以放到：

```txt
src/typings/api/system-manage.d.ts
```

在 `namespace SystemManage` 里添加：

```ts
type Dept = Common.CommonRecord<{
  deptName: string;
  deptCode: string;
  parentId: number;
}>;

type DeptList = Dept[];
```

如果后端返回的是分页结构，就写成：

```ts
type DeptList = Common.PaginatingQueryRecord<Dept>;
```

### 第二步：在接口文件中使用类型

在：

```txt
src/service/api/system-manage.ts
```

添加：

```ts
export function fetchGetDeptList() {
  return request<Api.SystemManage.DeptList>({
    url: '/systemManage/getDeptList',
    method: 'get'
  });
}
```

### 第三步：在页面中使用接口

页面调用：

```ts
const { data } = await fetchGetDeptList();
```

后面访问字段时，编辑器就能知道：

```ts
deptName
deptCode
parentId
```

这就是接口类型的完整落点。

## 二十六、命名空间为什么不用 import

你会发现项目里很多地方直接写：

```ts
Api.Auth.UserInfo
App.Theme.ThemeSetting
Env.ImportMeta
CommonType.YesOrNo
UnionKey.ThemeLayoutMode
```

没有写：

```ts
import type { Api } from '@/typings/api';
```

原因是这些类型通过：

```ts
declare namespace Api
declare namespace App
declare namespace Env
```

声明到了全局类型空间里。

这是一种后台模板项目里常见的写法。优点是：

- 类型使用方便。
- 接口类型全局可见。
- 适合模板项目和中后台项目。

缺点是：

- 全局命名空间太多时，来源不如显式 import 清晰。
- 类型命名要避免冲突。

你学习时要知道：这是项目选择的风格，不是所有 TypeScript 项目都必须这么写。

如果你自己从零搭建小项目，也可以选择显式导入类型：

```ts
import type { UserList } from '@/types/api/system-manage';
```

两种方式都可以，关键是保持项目内部风格统一。

## 二十七、`import type` 应该什么时候用

如果你只导入类型，建议使用：

```ts
import type { Router } from 'vue-router';
```

例如项目中：

```ts
import type { Router } from 'vue-router';

export function createProgressGuard(router: Router) {
  router.beforeEach(() => {
    window.NProgress?.start?.();
  });
}
```

`Router` 只用于类型标注，不需要在运行时代码中存在。

用 `import type` 的好处是：

- 告诉 TypeScript 这是纯类型导入。
- 构建后不会生成对应运行时 import。
- 避免一些循环依赖和副作用问题。

普通值导入：

```ts
import NProgress from 'nprogress';
```

这是运行时需要的，因为代码中真的要调用：

```ts
NProgress.configure(...)
```

简单判断：

```txt
只用于类型标注：import type
运行时要调用、读取、执行：import
```

## 二十八、类型检查命令

项目中有脚本：

```json
"typecheck": "vue-tsc --noEmit --skipLibCheck"
```

运行：

```bash
pnpm typecheck
```

含义是：

```txt
使用 vue-tsc 检查 TypeScript 和 Vue 单文件组件类型。
不输出编译产物。
跳过第三方库声明文件检查。
```

`vue-tsc` 比普通 `tsc` 更适合 Vue 项目，因为它能理解 `.vue` 文件中的 `<script setup>` 和模板类型。

`--noEmit` 表示只检查，不生成文件。

`--skipLibCheck` 表示跳过 `node_modules` 中 `.d.ts` 的深度检查，减少第三方类型导致的噪音。

学习阶段建议你经常运行：

```bash
pnpm typecheck
```

它能帮你发现很多编辑器可能没有及时暴露的问题。

## 二十九、阅读类型文件时不要陷入细节

本课涉及很多类型文件，但你现在不需要背下来。

建议按这个层级理解：

```txt
第一层：知道类型文件在哪里。
第二层：知道 Api、App、Env 分别负责什么。
第三层：能从接口函数跳转到对应类型。
第四层：能照着已有写法给新接口补类型。
第五层：后面学习路由、权限、主题时再深入对应类型。
```

现阶段最重要的是掌握使用路径：

```txt
看到 Api.SystemManage.UserList
  -> 去 src/typings/api/system-manage.d.ts 找

看到 App.Theme.ThemeSetting
  -> 去 src/typings/app.d.ts 找

看到 Env.ImportMeta
  -> 去 src/typings/vite-env.d.ts 找

看到 RouteMeta
  -> 去 src/typings/router.d.ts 找
```

这样你就不会在类型系统里迷路。

## 三十、课堂演示

### 演示 1：从别名跳转到真实文件

在任意文件中找到：

```ts
import { localStg } from '@/utils/storage';
```

然后对照：

```json
"@/*": ["./src/*"]
```

说明：

```txt
@/utils/storage -> src/utils/storage
```

再对照 `vite.config.ts`：

```ts
'@': fileURLToPath(new URL('./src', import.meta.url))
```

说明 TypeScript 和 Vite 都知道 `@` 指向 `src`。

### 演示 2：从接口函数跳转到接口类型

打开：

```txt
src/service/api/system-manage.ts
```

找到：

```ts
fetchGetUserList(params?: Api.SystemManage.UserSearchParams)
```

然后跳转到：

```txt
src/typings/api/system-manage.d.ts
```

找到：

```ts
type UserSearchParams = ...
type UserList = ...
```

说明请求参数和响应数据分别由哪个类型约束。

### 演示 3：故意写错环境变量

临时写一行：

```ts
import.meta.env.VITE_APP_TITL
```

正确字段是：

```ts
VITE_APP_TITLE
```

观察编辑器或 `pnpm typecheck` 是否提示。

实验完成后恢复代码。

### 演示 4：故意写错联合类型

临时写：

```ts
const mode: UnionKey.ThemeLayoutMode = 'vertcial';
```

正确值是：

```ts
'vertical'
```

观察 TypeScript 是否提示错误。

实验完成后恢复代码。

### 演示 5：理解 `Window.NProgress`

打开：

```txt
src/plugins/nprogress.ts
```

看到：

```ts
window.NProgress = NProgress;
```

再打开：

```txt
src/typings/global.d.ts
```

看到：

```ts
interface Window {
  NProgress?: import('nprogress').NProgress;
}
```

说明如果没有这个声明，TypeScript 不知道 `window` 上可以挂 `NProgress`。

## 三十一、本课作业

### 作业 1：整理类型命名空间职责

新建学习笔记，整理：

```md
# 类型命名空间职责

- Api：
- App：
- Env：
- CommonType：
- UnionKey：
- StorageType：
- UI：
```

要求每个命名空间用一句话说明。

### 作业 2：找出用户列表接口对应类型

根据以下路径整理：

```txt
src/service/api/system-manage.ts
src/typings/api/system-manage.d.ts
src/typings/api/common.d.ts
```

回答：

```md
# 用户列表接口类型分析

- 接口函数：
- 请求参数类型：
- 响应数据类型：
- 单个用户记录类型：
- 分页结构类型：
- 通用记录字段来自哪里：
```

### 作业 3：给一个测试接口补充类型

不用真正提交代码，可以在草稿中设计：

```txt
接口：/systemManage/getDeptList
返回：部门列表
```

写出：

```ts
type Dept = ...
type DeptList = ...

export function fetchGetDeptList() {
  return request<Api.SystemManage.DeptList>({
    url: '/systemManage/getDeptList',
    method: 'get'
  });
}
```

重点是照着项目已有类型风格写。

### 作业 4：运行类型检查

执行：

```bash
pnpm typecheck
```

记录：

```md
# 类型检查记录

- 是否通过：
- 如果报错，报错文件：
- 报错原因：
- 如何修复：
```

## 三十二、自检问题

完成本课后，尝试回答：

- `tsconfig.json` 是给谁看的？
- `@` 和 `~` 分别指向哪里？
- 为什么 `tsconfig.json` 和 `vite.config.ts` 都要配置 alias？
- `types: ["vite/client", "node"]` 是什么作用？
- `.d.ts` 和 `.ts` 有什么区别？
- 为什么 `Api.SystemManage.UserList` 不需要 import？
- `declare namespace Api` 是运行时代码吗？
- `declare module 'vue-router'` 是什么作用？
- `declare global` 是什么作用？
- `Window.NProgress` 的类型在哪里声明？
- `Env.ImportMeta` 约束了什么？
- `CommonType.YesOrNo` 为什么比普通 `string` 更安全？
- `request<Api.Auth.LoginToken>` 中的泛型是什么意思？
- `import type` 什么时候使用？
- `pnpm typecheck` 做什么？

如果这些问题能回答出来，就可以进入第 9 课。

## 三十三、下一课预告

第 9 课会进入 Pinia 状态管理入门，重点学习：

- `defineStore` 的基本写法。
- setup store 和 option store 的区别。
- 本项目 store 的目录组织。
- `useAuthStore`、`useRouteStore`、`useThemeStore` 等模块职责。
- store 如何和本地缓存、路由、请求联动。

第 8 课理解的是类型如何约束数据，第 9 课会理解这些数据如何被放进状态管理中，并在页面之间共享。
