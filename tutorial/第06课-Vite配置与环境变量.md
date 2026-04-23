# 第 6 课：Vite 配置与环境变量

## 课程定位

第 5 课学习了应用如何从 `src/main.ts` 启动。第 6 课要往启动链路的上游看：Vite 是如何决定项目的开发端口、路径别名、环境变量、代理、插件和构建行为的。

后台管理项目通常有多个环境：本地开发、测试环境、生产环境。不同环境可能有不同接口地址、部署路径、代理开关、路由模式和构建配置。如果不了解 Vite 配置和环境变量，后续接真实后端、部署到子路径、排查接口代理都会很困难。

本课重点是读懂 `vite.config.ts`、`.env`、`.env.test`、`.env.prod` 和 `build/config`。

## 学习目标

学完本课后，你应该能够：

- 说明 `vite.config.ts` 在项目中的作用。
- 理解 `defineConfig` 的意义。
- 理解 `loadEnv(configEnv.mode, process.cwd())` 如何读取环境变量。
- 区分 `.env`、`.env.test`、`.env.prod` 的职责。
- 说明 `VITE_BASE_URL`、`VITE_SERVICE_BASE_URL`、`VITE_HTTP_PROXY` 等变量的作用。
- 理解 `resolve.alias` 中 `@` 和 `~` 的配置方式。
- 理解 `server`、`preview`、`build` 配置分别影响什么。
- 理解开发代理 `createViteProxy` 的基本链路。
- 能够完成修改端口、修改标题、修改接口 baseURL 的练习。

## 建议课时

建议用 90-120 分钟完成本课：

- 15 分钟：完整阅读 `vite.config.ts`。
- 20 分钟：理解 `.env`、`.env.test`、`.env.prod`。
- 20 分钟：理解 base、alias、server、preview、build。
- 20 分钟：理解代理配置链路。
- 20 分钟：完成修改端口、标题、接口地址实验。
- 10 分钟：整理环境变量说明表。

## 一、`vite.config.ts` 是什么

`vite.config.ts` 是 Vite 项目的主配置文件。

它决定：

- 应用部署基础路径。
- 路径别名。
- CSS 预处理器配置。
- Vite 插件。
- 全局常量替换。
- 开发服务器端口。
- 是否自动打开浏览器。
- 开发代理。
- 预览服务器端口。
- 构建 sourcemap。
- 构建兼容选项。

当前项目的配置入口：

```ts
import process from 'node:process';
import { URL, fileURLToPath } from 'node:url';
import { defineConfig, loadEnv } from 'vite';
import { setupVitePlugins } from './build/plugins';
import { createViteProxy, getBuildTime } from './build/config';

export default defineConfig(configEnv => {
  const viteEnv = loadEnv(configEnv.mode, process.cwd()) as unknown as Env.ImportMeta;

  const buildTime = getBuildTime();

  const enableProxy = configEnv.command === 'serve' && !configEnv.isPreview;

  return {
    base: viteEnv.VITE_BASE_URL,
    resolve: {
      alias: {
        '~': fileURLToPath(new URL('./', import.meta.url)),
        '@': fileURLToPath(new URL('./src', import.meta.url))
      }
    },
    css: {
      preprocessorOptions: {
        scss: {
          api: 'modern-compiler',
          additionalData: `@use "@/styles/scss/global.scss" as *;`
        }
      }
    },
    plugins: setupVitePlugins(viteEnv, buildTime),
    define: {
      BUILD_TIME: JSON.stringify(buildTime)
    },
    server: {
      host: '0.0.0.0',
      port: 9527,
      open: true,
      proxy: createViteProxy(viteEnv, enableProxy)
    },
    preview: {
      port: 9725
    },
    build: {
      reportCompressedSize: false,
      sourcemap: viteEnv.VITE_SOURCE_MAP === 'Y',
      commonjsOptions: {
        ignoreTryCatch: false
      }
    }
  };
});
```

本课会逐块拆开。

## 二、`defineConfig` 是什么

`defineConfig` 来自 Vite：

```ts
import { defineConfig } from 'vite';
```

它的作用是帮助你写 Vite 配置，并获得更好的类型提示。

简单写法可以是：

```ts
export default {
  server: {
    port: 9527
  }
};
```

但更推荐：

```ts
export default defineConfig({
  server: {
    port: 9527
  }
});
```

好处：

- 编辑器有配置项提示。
- TypeScript 能检查配置类型。
- 支持返回函数，根据 command 和 mode 动态生成配置。

本项目使用的是函数形式：

```ts
export default defineConfig(configEnv => {
  return {};
});
```

这说明配置会根据当前命令和模式动态生成。

## 三、`configEnv` 是什么

`defineConfig` 的函数参数：

```ts
configEnv
```

通常包含：

- `command`
- `mode`
- `isPreview`

在本项目中用到了：

```ts
configEnv.mode
configEnv.command
configEnv.isPreview
```

### 1. `configEnv.mode`

表示当前运行模式。

例如：

```bash
pnpm dev
```

对应脚本：

```json
"dev": "vite --mode test"
```

所以：

```ts
configEnv.mode === 'test'
```

如果执行：

```bash
pnpm build
```

对应：

```json
"build": "vite build --mode prod"
```

所以：

```ts
configEnv.mode === 'prod'
```

### 2. `configEnv.command`

表示当前是开发服务还是构建。

常见值：

- `serve`：开发服务器或 preview。
- `build`：生产构建。

本项目用它判断是否启用代理：

```ts
const enableProxy = configEnv.command === 'serve' && !configEnv.isPreview;
```

### 3. `configEnv.isPreview`

表示当前是否是 `vite preview`。

本项目不希望 preview 阶段启用开发代理，所以加了：

```ts
!configEnv.isPreview
```

## 四、`loadEnv` 如何读取环境变量

本项目读取环境变量：

```ts
const viteEnv = loadEnv(configEnv.mode, process.cwd()) as unknown as Env.ImportMeta;
```

拆开看：

```ts
loadEnv(mode, envDir)
```

这里：

- `configEnv.mode`：当前模式，例如 `test` 或 `prod`。
- `process.cwd()`：当前执行命令的工作目录，通常是项目根目录。

所以：

```ts
loadEnv('test', process.cwd())
```

会读取：

- `.env`
- `.env.test`

```ts
loadEnv('prod', process.cwd())
```

会读取：

- `.env`
- `.env.prod`

读取结果被强制断言成：

```ts
Env.ImportMeta
```

这个类型在 `src/typings/vite-env.d.ts` 中声明，用于给环境变量提供类型提示。

## 五、`.env`、`.env.test`、`.env.prod` 的职责

本项目有三个环境文件：

```txt
.env
.env.test
.env.prod
```

### 1. `.env`

公共环境变量。

无论 `test` 还是 `prod` 模式，都会读取它。

示例：

```bash
VITE_BASE_URL=/
VITE_APP_TITLE=SoybeanAdmin
VITE_AUTH_ROUTE_MODE=static
VITE_HTTP_PROXY=Y
VITE_ROUTER_HISTORY_MODE=history
VITE_SERVICE_SUCCESS_CODE=0000
VITE_SOURCE_MAP=N
```

适合放：

- 应用标题。
- 路由模式。
- 权限路由模式。
- 成功 code。
- token 过期 code。
- 是否开启代理。
- 是否开启 sourcemap。

### 2. `.env.test`

测试模式环境变量。

执行：

```bash
pnpm dev
```

或者：

```bash
pnpm build:test
```

会读取 `.env.test`。

当前内容：

```bash
VITE_SERVICE_BASE_URL=https://mock.apifox.cn/m1/3109515-0-default

VITE_OTHER_SERVICE_BASE_URL= `{
  "demo": "http://localhost:9528"
}`
```

适合放：

- 测试环境接口地址。
- 测试环境其他服务地址。

### 3. `.env.prod`

生产模式环境变量。

执行：

```bash
pnpm dev:prod
```

或者：

```bash
pnpm build
```

会读取 `.env.prod`。

当前内容：

```bash
VITE_SERVICE_BASE_URL=https://mock.apifox.cn/m1/3109515-0-default

VITE_OTHER_SERVICE_BASE_URL= `{
  "demo": "http://localhost:9529"
}`
```

适合放：

- 生产环境接口地址。
- 生产环境其他服务地址。

当前项目 test 和 prod 的主接口地址都指向 mock，但真实项目中通常会不同。

## 六、为什么变量都以 `VITE_` 开头

Vite 有一个规则：只有以 `VITE_` 开头的环境变量，才会暴露给客户端代码。

例如：

```bash
VITE_APP_TITLE=SoybeanAdmin
```

可以在前端代码中通过：

```ts
import.meta.env.VITE_APP_TITLE
```

读取。

如果你写：

```bash
APP_SECRET=xxx
```

默认不会暴露给前端。

这样做是为了避免把敏感信息错误打包到浏览器代码里。

注意：即使使用 `VITE_`，也不要把真正的密码、数据库账号、私钥放到前端环境变量里。前端变量最终可能被打包到浏览器代码中。

## 七、`base` 配置

`vite.config.ts` 中：

```ts
base: viteEnv.VITE_BASE_URL,
```

对应 `.env`：

```bash
VITE_BASE_URL=/
```

`base` 表示应用部署的基础路径。

如果项目部署在域名根路径：

```txt
https://example.com/
```

通常配置：

```bash
VITE_BASE_URL=/
```

如果项目部署在子路径：

```txt
https://example.com/admin/
```

应该配置：

```bash
VITE_BASE_URL=/admin/
```

注意注释里强调：

```txt
如果使用子目录，必须以 / 结尾，例如 /admin/，不要写 /admin
```

`base` 会影响：

- 构建后静态资源路径。
- router history base。
- 版本更新检测读取 index.html 的路径。

## 八、路径别名 `resolve.alias`

当前配置：

```ts
resolve: {
  alias: {
    '~': fileURLToPath(new URL('./', import.meta.url)),
    '@': fileURLToPath(new URL('./src', import.meta.url))
  }
}
```

含义：

- `~` 指向项目根目录。
- `@` 指向 `src` 目录。

所以你可以写：

```ts
import { localStg } from '@/utils/storage';
```

而不是：

```ts
import { localStg } from '../../utils/storage';
```

### 为什么使用 `fileURLToPath(new URL(...))`

项目是 ESM：

```json
"type": "module"
```

ESM 中没有 CommonJS 的 `__dirname`。

所以使用：

```ts
fileURLToPath(new URL('./src', import.meta.url))
```

它的含义是：

```txt
以当前 vite.config.ts 文件所在位置为基准，找到 ./src，并转换成文件系统路径
```

最终得到类似：

```txt
/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src
```

## 九、CSS 预处理配置

当前配置：

```ts
css: {
  preprocessorOptions: {
    scss: {
      api: 'modern-compiler',
      additionalData: `@use "@/styles/scss/global.scss" as *;`
    }
  }
}
```

它影响 SCSS 文件。

### 1. `api: 'modern-compiler'`

表示使用 Sass 的现代编译 API。

你不需要深入它的内部实现，只需要知道它是 Sass 编译方式配置。

### 2. `additionalData`

表示每个 SCSS 文件编译前，都会自动追加这段内容：

```scss
@use "@/styles/scss/global.scss" as *;
```

这样在组件的 `<style lang="scss">` 中，可以直接使用 `global.scss` 中暴露的变量、mixin 或函数，而不用每个文件都手动写：

```scss
@use "@/styles/scss/global.scss" as *;
```

注意：这只影响 SCSS，不影响普通 CSS，也不影响 UnoCSS class。

## 十、Vite 插件配置

当前配置：

```ts
plugins: setupVitePlugins(viteEnv, buildTime),
```

具体实现位于：

```txt
build/plugins/index.ts
```

内容：

```ts
export function setupVitePlugins(viteEnv: Env.ImportMeta, buildTime: string) {
  const plugins: PluginOption = [
    vue(),
    vueJsx(),
    setupDevtoolsPlugin(viteEnv),
    setupElegantRouter(),
    setupUnocss(viteEnv),
    ...setupUnplugin(viteEnv),
    progress(),
    setupHtmlPlugin(buildTime)
  ];

  return plugins;
}
```

这些插件大致负责：

- `vue()`：支持 `.vue` 单文件组件。
- `vueJsx()`：支持 Vue JSX/TSX。
- `setupDevtoolsPlugin`：Vue DevTools。
- `setupElegantRouter`：Elegant Router 路由生成。
- `setupUnocss`：UnoCSS。
- `setupUnplugin`：自动组件、图标、SVG sprite。
- `progress()`：构建进度。
- `setupHtmlPlugin`：HTML 处理和构建时间注入。

本课只需要知道插件入口在这里。第 7 课和后续工程化章节会继续深入。

## 十一、`define` 配置和 `BUILD_TIME`

当前配置：

```ts
define: {
  BUILD_TIME: JSON.stringify(buildTime)
}
```

`define` 用于在构建时定义全局常量。

这里定义了：

```ts
BUILD_TIME
```

它的值来自：

```ts
const buildTime = getBuildTime();
```

`getBuildTime` 位于：

```txt
build/config/time.ts
```

它使用 Dayjs 生成上海时区的构建时间：

```ts
const buildTime = dayjs.tz(Date.now(), 'Asia/Shanghai').format('YYYY-MM-DD HH:mm:ss');
```

所以项目代码中可以直接使用：

```ts
BUILD_TIME
```

例如版本更新检测中会用它判断应用是否有新版本。

类型声明在：

```txt
src/typings/global.d.ts
```

```ts
export const BUILD_TIME: string;
```

## 十二、开发服务器 `server`

当前配置：

```ts
server: {
  host: '0.0.0.0',
  port: 9527,
  open: true,
  proxy: createViteProxy(viteEnv, enableProxy)
}
```

### 1. `host: '0.0.0.0'`

表示开发服务器监听所有网络地址。

好处：

- 本机可以通过 `localhost:9527` 访问。
- 同一局域网设备可以通过你的局域网 IP 访问。

### 2. `port: 9527`

开发服务器端口。

启动后通常访问：

```txt
http://localhost:9527/
```

### 3. `open: true`

启动后自动打开浏览器。

### 4. `proxy`

开发代理配置。

它来自：

```ts
createViteProxy(viteEnv, enableProxy)
```

下面单独讲。

## 十三、预览服务器 `preview`

当前配置：

```ts
preview: {
  port: 9725
}
```

它影响：

```bash
pnpm preview
```

也就是：

```bash
vite preview
```

preview 用于预览构建产物，不是开发服务器。

通常流程：

```bash
pnpm build
pnpm preview
```

然后访问：

```txt
http://localhost:9725/
```

## 十四、构建配置 `build`

当前配置：

```ts
build: {
  reportCompressedSize: false,
  sourcemap: viteEnv.VITE_SOURCE_MAP === 'Y',
  commonjsOptions: {
    ignoreTryCatch: false
  }
}
```

### 1. `reportCompressedSize: false`

关闭 gzip 压缩体积报告。

好处：

- 构建速度略快。
- 构建日志更简洁。

### 2. `sourcemap`

```ts
sourcemap: viteEnv.VITE_SOURCE_MAP === 'Y'
```

对应 `.env`：

```bash
VITE_SOURCE_MAP=N
```

如果改成：

```bash
VITE_SOURCE_MAP=Y
```

构建时会生成 sourcemap。

sourcemap 的作用：

- 方便线上调试。
- 能从压缩代码映射回源码。

但也有风险：

- 可能暴露源码结构。
- 构建产物更大。

生产环境是否开启要根据团队策略决定。

### 3. `commonjsOptions.ignoreTryCatch`

这是 Rollup/Vite 的 CommonJS 处理选项。

本课不需要深入，只需要知道它影响 CommonJS 模块转换行为。

## 十五、开发代理的判断条件

当前代码：

```ts
const enableProxy = configEnv.command === 'serve' && !configEnv.isPreview;
```

意思是：

```txt
只有开发服务 serve 且不是 preview 时，才允许启用代理
```

然后：

```ts
proxy: createViteProxy(viteEnv, enableProxy)
```

在 `createViteProxy` 中还会判断：

```ts
const isEnableHttpProxy = enable && env.VITE_HTTP_PROXY === 'Y';
```

也就是说，代理真正启用需要两个条件：

```txt
当前是 dev server，不是 preview
并且 VITE_HTTP_PROXY=Y
```

如果是：

```bash
pnpm build
```

不会启用开发代理。

如果是：

```bash
pnpm preview
```

也不会启用开发代理。

## 十六、代理配置链路

代理链路涉及几个文件：

```txt
.env
.env.test / .env.prod
vite.config.ts
build/config/proxy.ts
src/utils/service.ts
src/service/request/index.ts
```

### 1. 环境变量提供真实接口地址

`.env.test`：

```bash
VITE_SERVICE_BASE_URL=https://mock.apifox.cn/m1/3109515-0-default
```

`.env`：

```bash
VITE_HTTP_PROXY=Y
VITE_PROXY_LOG=Y
```

### 2. `createServiceConfig` 生成代理模式

`src/utils/service.ts` 中：

```ts
function createProxyPattern(key?: App.Service.OtherBaseURLKey) {
  if (!key) {
    return '/proxy-default';
  }

  return `/proxy-${key}`;
}
```

默认接口代理前缀是：

```txt
/proxy-default
```

其他服务例如 `demo` 会生成：

```txt
/proxy-demo
```

### 3. `createViteProxy` 生成 Vite proxy

`build/config/proxy.ts` 中：

```ts
proxy[item.proxyPattern] = {
  target: item.baseURL,
  changeOrigin: true,
  rewrite: path => path.replace(new RegExp(`^${item.proxyPattern}`), '')
};
```

意思是：

```txt
请求 /proxy-default/xxx
转发到 VITE_SERVICE_BASE_URL/xxx
```

并且去掉 `/proxy-default` 前缀。

### 4. 请求层根据是否代理决定 baseURL

`src/service/request/index.ts` 中：

```ts
const isHttpProxy = import.meta.env.DEV && import.meta.env.VITE_HTTP_PROXY === 'Y';
const { baseURL, otherBaseURL } = getServiceBaseURL(import.meta.env, isHttpProxy);
```

如果启用代理：

```ts
baseURL = '/proxy-default'
```

如果不启用代理：

```ts
baseURL = VITE_SERVICE_BASE_URL
```

这样页面代码不用关心代理细节。

## 十七、代理示例

假设：

```bash
VITE_HTTP_PROXY=Y
VITE_SERVICE_BASE_URL=https://mock.apifox.cn/m1/3109515-0-default
```

请求层 baseURL 是：

```txt
/proxy-default
```

页面调用接口：

```ts
request({
  url: '/auth/login'
});
```

浏览器实际请求：

```txt
http://localhost:9527/proxy-default/auth/login
```

Vite dev server 转发到：

```txt
https://mock.apifox.cn/m1/3109515-0-default/auth/login
```

这就是开发代理。

它主要解决：

- 本地开发跨域问题。
- 本地接口地址统一管理。
- 多服务代理。

## 十八、`import.meta.env` 的使用

Vite 在前端代码中通过：

```ts
import.meta.env
```

暴露环境变量。

例如：

```ts
import.meta.env.VITE_AUTH_ROUTE_MODE
import.meta.env.VITE_SERVICE_SUCCESS_CODE
import.meta.env.VITE_HTTP_PROXY
import.meta.env.DEV
import.meta.env.PROD
import.meta.env.MODE
```

本项目中常见用法：

- 路由模式：`VITE_ROUTER_HISTORY_MODE`
- 接口成功 code：`VITE_SERVICE_SUCCESS_CODE`
- 权限路由模式：`VITE_AUTH_ROUTE_MODE`
- 是否启用代理：`VITE_HTTP_PROXY`
- 当前是否开发环境：`import.meta.env.DEV`
- 当前是否生产环境：`import.meta.env.PROD`
- 当前 mode：`import.meta.env.MODE`

注意：

- `VITE_` 开头的是自定义环境变量。
- `DEV`、`PROD`、`MODE` 是 Vite 内置变量。

## 十九、修改开发端口实验

当前端口：

```ts
server: {
  port: 9527
}
```

你可以临时改成：

```ts
port: 9528
```

然后重启：

```bash
pnpm dev
```

观察终端和浏览器地址变化。

实验结束后建议改回：

```ts
port: 9527
```

## 二十、修改应用标题实验

`.env` 中：

```bash
VITE_APP_TITLE=SoybeanAdmin
```

你可以改成：

```bash
VITE_APP_TITLE=MyAdmin
```

然后重启开发服务器。

注意：环境变量通常需要重启 Vite 才会生效。

如果页面标题或系统名称没有变化，可以搜索：

```txt
VITE_APP_TITLE
```

或者搜索：

```txt
system.title
```

因为这个项目有国际化，页面显示的标题不一定直接来自 `VITE_APP_TITLE`。

这个实验的重点不是一定改出视觉效果，而是理解：

```txt
环境变量修改后需要重新启动开发服务器
```

## 二十一、修改接口 baseURL 实验

`.env.test` 中：

```bash
VITE_SERVICE_BASE_URL=https://mock.apifox.cn/m1/3109515-0-default
```

你可以临时改成一个测试地址，例如：

```bash
VITE_SERVICE_BASE_URL=https://example.com/api
```

然后重启：

```bash
pnpm dev
```

打开浏览器 Network 面板，观察请求是否转发到新的目标。

如果 `VITE_HTTP_PROXY=Y`，浏览器看到的可能仍然是：

```txt
/proxy-default/xxx
```

但终端代理日志中会显示真实请求地址。

如果你想在浏览器中直接看到真实 baseURL，可以临时改：

```bash
VITE_HTTP_PROXY=N
```

然后重启。

注意：实验结束后改回原配置。

## 二十二、课堂演示建议

### 演示 1：从 scripts 推导 mode

打开 `package.json`：

```json
"dev": "vite --mode test",
"build": "vite build --mode prod"
```

讲解：

- `pnpm dev` 读取 `.env` 和 `.env.test`。
- `pnpm build` 读取 `.env` 和 `.env.prod`。

### 演示 2：修改端口

把 `server.port` 改成 `9528`。

重启开发服务器，观察访问地址。

### 演示 3：解释 alias

展示：

```ts
'@': fileURLToPath(new URL('./src', import.meta.url))
```

再打开业务代码中的：

```ts
import { localStg } from '@/utils/storage';
```

讲解 `@` 如何指向 `src`。

### 演示 4：观察代理日志

确保：

```bash
VITE_HTTP_PROXY=Y
VITE_PROXY_LOG=Y
```

启动项目并触发接口请求。

观察终端中的：

```txt
[proxy url]
[real request url]
```

### 演示 5：切换是否使用代理

临时修改：

```bash
VITE_HTTP_PROXY=N
```

重启项目，对比 Network 面板中的请求地址。

## 二十三、常见误区

### 误区 1：改 `.env` 不重启 Vite

环境变量在启动时读取。

修改 `.env` 后通常需要重启：

```bash
pnpm dev
```

### 误区 2：以为 `.env.prod` 只在服务器上生效

不是。

只要你执行：

```bash
vite --mode prod
```

就会读取 `.env.prod`。

例如：

```bash
pnpm dev:prod
```

也是开发服务器，但读取 prod 环境变量。

### 误区 3：把敏感信息写进 `VITE_` 变量

`VITE_` 变量会暴露给前端代码。

不要写：

```bash
VITE_DB_PASSWORD=xxx
VITE_SECRET_KEY=xxx
```

### 误区 4：分不清 dev server 和 preview

`server` 配置影响：

```bash
pnpm dev
```

`preview` 配置影响：

```bash
pnpm preview
```

它们不是一回事。

### 误区 5：以为代理在生产也生效

Vite dev server proxy 只服务本地开发。

生产环境接口代理通常由：

- Nginx。
- 网关。
- 后端服务。
- 部署平台。

来处理。

## 二十四、本课源码导读

### 1. `vite.config.ts`

重点看：

- `defineConfig`。
- `loadEnv`。
- `base`。
- `resolve.alias`。
- `css.preprocessorOptions`。
- `plugins`。
- `define`。
- `server`。
- `preview`。
- `build`。

### 2. `.env`

重点看：

- `VITE_BASE_URL`。
- `VITE_APP_TITLE`。
- `VITE_AUTH_ROUTE_MODE`。
- `VITE_HTTP_PROXY`。
- `VITE_ROUTER_HISTORY_MODE`。
- `VITE_SOURCE_MAP`。

### 3. `.env.test`

重点看：

- `VITE_SERVICE_BASE_URL`。
- `VITE_OTHER_SERVICE_BASE_URL`。

### 4. `.env.prod`

重点看：

- `VITE_SERVICE_BASE_URL`。
- `VITE_OTHER_SERVICE_BASE_URL`。

### 5. `build/config/proxy.ts`

重点看：

- `createViteProxy`。
- `createProxyItem`。
- `target`。
- `rewrite`。
- `VITE_PROXY_LOG`。

### 6. `src/utils/service.ts`

重点看：

- `createServiceConfig`。
- `getServiceBaseURL`。
- `createProxyPattern`。

它连接了环境变量、Vite 代理和请求层 baseURL。

## 二十五、本课作业

### 作业 1：整理 Vite 配置说明表

```md
# vite.config.ts 配置说明

- base：
- resolve.alias：
- css.preprocessorOptions：
- plugins：
- define：
- server.host：
- server.port：
- server.open：
- server.proxy：
- preview.port：
- build.sourcemap：
```

### 作业 2：整理环境变量说明表

```md
# 环境变量说明

- VITE_BASE_URL：
- VITE_APP_TITLE：
- VITE_AUTH_ROUTE_MODE：
- VITE_HTTP_PROXY：
- VITE_ROUTER_HISTORY_MODE：
- VITE_SERVICE_BASE_URL：
- VITE_OTHER_SERVICE_BASE_URL：
- VITE_SOURCE_MAP：
- VITE_PROXY_LOG：
```

### 作业 3：修改开发端口

把端口从 `9527` 改为 `9528`，重启项目，记录访问地址。

实验完成后改回 `9527`。

### 作业 4：修改接口地址并观察请求

临时修改 `.env.test` 中的：

```bash
VITE_SERVICE_BASE_URL
```

重启项目，观察：

- 浏览器 Network 面板。
- 终端代理日志。

实验完成后恢复原值。

### 作业 5：画代理流程图

按下面格式画：

```txt
页面调用 request('/auth/login')
→ baseURL = /proxy-default
→ 浏览器请求 /proxy-default/auth/login
→ Vite proxy 命中 /proxy-default
→ rewrite 去掉 /proxy-default
→ 转发到 VITE_SERVICE_BASE_URL/auth/login
```

## 二十六、自检问题

完成本课后，尝试回答：

- `vite.config.ts` 的作用是什么？
- `defineConfig` 有什么好处？
- `loadEnv(configEnv.mode, process.cwd())` 会读取哪些文件？
- `pnpm dev` 使用什么 mode？
- `pnpm build` 使用什么 mode？
- `.env` 和 `.env.test` 的区别是什么？
- 为什么自定义前端环境变量要以 `VITE_` 开头？
- `VITE_BASE_URL` 影响什么？
- `@` 路径别名是如何配置的？
- `server.port` 和 `preview.port` 分别影响哪个命令？
- 什么条件下会启用 Vite 开发代理？
- `/proxy-default` 是从哪里来的？
- `VITE_HTTP_PROXY=Y` 时，请求层的 baseURL 会变成什么？
- `build.sourcemap` 由哪个环境变量控制？

如果这些问题能回答出来，就可以进入第 7 课。

## 二十七、下一课预告

第 7 课会进入插件系统与全局能力注册，重点学习：

- `src/plugins/index.ts` 如何统一导出插件。
- `setupUI` 如何注册 Element Plus。
- `setupLoading` 如何生成首屏 loading。
- `setupNProgress` 如何配合路由守卫。
- `setupIconifyOffline` 如何配置图标服务。
- `setupDayjs` 如何设置日期语言。
- `setupAppVersionNotification` 如何检测版本更新。

第 6 课理解的是 Vite 如何配置项目，第 7 课会理解项目内部插件如何初始化。
