# 第 13 课：Token 过期与刷新机制

## 课程定位

第 12 课我们已经把请求层的总体结构看清楚了，知道了：

```txt
请求发出前会自动加 Authorization
响应回来后会先判断 HTTP 成功，再判断业务 code 成功
业务失败时会进入 onBackendFail
最终错误会进入 onError
```

但后台管理项目里还有一条最关键、也最容易绕的链路没有彻底讲透：

```txt
token 过期了怎么办？
为什么项目不是直接跳登录页？
为什么有时候页面看起来只是“卡了一下”，请求又成功了？
为什么多个请求同时过期时，不会疯狂刷新 token？
```

这就是本课要解决的问题。

在真实后台项目里，token 过期是高频场景。如果每次过期都强制用户重新登录，体验会很差。于是项目通常会设计成：

```txt
token 过期
  -> 用 refreshToken 换新的 token
  -> 刷新成功后自动重发原请求
  -> 用户几乎无感
```

这个仓库就实现了这样一套机制，而且代码量不大，非常适合拿来学习。

## 学习目标

学完本课后，你应该能够：

- 理解 token 过期为什么不一定要立刻跳登录页。
- 理解项目如何识别“这是 token 过期”。
- 理解 `handleExpiredRequest()` 的职责。
- 理解 `handleRefreshToken()` 如何刷新 token。
- 理解刷新成功后为什么要把新的 token 写回本地存储。
- 理解为什么刷新成功后还要重发原请求。
- 理解项目如何避免多个并发请求重复刷新 token。
- 理解刷新失败后为什么要执行 `resetStore()`。
- 理解 `onBackendFail` 和 `onError` 在过期场景里的分工。
- 知道如何在项目演示页面里手动触发一次 token 过期流程。

## 建议课时

建议用 90-120 分钟完成本课：

- 20 分钟：阅读 `src/service/request/index.ts` 中的过期逻辑。
- 20 分钟：阅读 `src/service/request/shared.ts`。
- 15 分钟：阅读 `src/service/api/auth.ts` 中的 `fetchRefreshToken`。
- 15 分钟：回顾 `auth store` 的 `resetStore()`。
- 15 分钟：阅读 `.env` 中的过期相关 code。
- 15 分钟：在演示页面中理解如何手动触发 token 过期。

## 一、本课重点文件

本课重点阅读：

- [src/service/request/index.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/service/request/index.ts)
- [src/service/request/shared.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/service/request/shared.ts)
- [src/service/request/type.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/service/request/type.ts)
- [src/service/api/auth.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/service/api/auth.ts)
- [src/store/modules/auth/index.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/store/modules/auth/index.ts)
- [.env](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/.env)
- [src/views/function/request/index.vue](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/views/function/request/index.vue)

## 二、先明确：token 过期到底是什么意思

在后台项目里，登录后浏览器通常会持有：

- `token`
- `refreshToken`

其中：

### `token`

用于每次正常请求时放进请求头：

```txt
Authorization: Bearer xxx
```

它通常有效期较短。

### `refreshToken`

用于在 `token` 失效时申请新的 `token`。

它通常有效期更长。

所以 token 过期不是：

```txt
用户彻底登出了
```

而是：

```txt
当前工作凭证过期了，但系统可能还有资格帮用户续期
```

这就是为什么很多后台项目不会一过期就立刻跳登录页。

## 三、这个项目怎么识别“token 过期”

项目不是靠 HTTP 状态码直接判断，而是看后端业务 code。

在 `.env` 中：

```env
VITE_SERVICE_EXPIRED_TOKEN_CODES=9999,9998,3333
```

这表示：

```txt
如果后端响应 code 是 9999 / 9998 / 3333，
就把它视为 token 过期场景。
```

对应逻辑在 [src/service/request/index.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/service/request/index.ts)：

```ts
const expiredTokenCodes = import.meta.env.VITE_SERVICE_EXPIRED_TOKEN_CODES?.split(',') || [];
if (expiredTokenCodes.includes(responseCode)) {
  const success = await handleExpiredRequest(request.state);
  if (success) {
    const Authorization = getAuthorization();
    Object.assign(response.config.headers, { Authorization });

    return instance.request(response.config) as Promise<AxiosResponse>;
  }
}
```

所以真正的识别条件是：

```txt
HTTP 请求成功返回了响应
但后端 code 告诉前端：当前 token 已过期
```

## 四、为什么项目不是直接跳登录页

如果每次 token 过期都立刻跳登录页，会有几个明显问题：

- 用户正在编辑表单，体验会非常差。
- 用户正在翻页、搜索、切换页面，会频繁被中断。
- 即便用户仍然持有有效的 `refreshToken`，也没机会自动续期。

所以更合理的策略是：

```txt
优先尝试 refreshToken
  -> 成功：自动恢复会话，重发原请求
  -> 失败：再重置认证状态并跳登录页
```

这正是本项目采用的策略。

## 五、token 过期时会进入哪条代码链路

先把整条主线写出来：

```txt
业务调用 request(...)
  -> 后端返回过期 code（例如 9999）
  -> onBackendFail 识别为 expiredTokenCodes
  -> handleExpiredRequest(request.state)
  -> handleRefreshToken()
  -> fetchRefreshToken(refreshToken)
  -> 刷新成功则更新 localStorage
  -> 用新 token 重写原请求头
  -> instance.request(response.config) 重发原请求
```

这就是整套刷新机制的骨架。

## 六、入口：`onBackendFail` 中的过期处理

在 `src/service/request/index.ts` 中：

```ts
const expiredTokenCodes = import.meta.env.VITE_SERVICE_EXPIRED_TOKEN_CODES?.split(',') || [];
if (expiredTokenCodes.includes(responseCode)) {
  const success = await handleExpiredRequest(request.state);
  if (success) {
    const Authorization = getAuthorization();
    Object.assign(response.config.headers, { Authorization });

    return instance.request(response.config) as Promise<AxiosResponse>;
  }
}
```

这里可以拆成三步。

### 第一步：判断当前错误码是不是过期码

```ts
expiredTokenCodes.includes(responseCode)
```

如果不是，就走别的错误处理分支。

### 第二步：尝试刷新 token

```ts
const success = await handleExpiredRequest(request.state);
```

这里不是直接调刷新接口，而是交给统一封装函数处理。

### 第三步：刷新成功后重发原请求

```ts
Object.assign(response.config.headers, { Authorization });
return instance.request(response.config);
```

这说明刷新 token 的目的不是单纯把本地 token 改掉，而是：

```txt
让刚才失败的那次请求再成功跑一遍
```

## 七、`handleExpiredRequest()` 是做什么的

在 [src/service/request/shared.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/service/request/shared.ts)：

```ts
export async function handleExpiredRequest(state: RequestInstanceState) {
  if (!state.refreshTokenFn) {
    state.refreshTokenFn = handleRefreshToken();
  }

  const success = await state.refreshTokenFn;

  setTimeout(() => {
    state.refreshTokenFn = null;
  }, 1000);

  return success;
}
```

它的职责不是“直接刷新 token”，而是：

```txt
管理 token 刷新动作的并发和复用
```

你可以把它理解成：

```txt
如果当前没有刷新动作在进行，就发起一次刷新；
如果已经有刷新动作在进行，就等那一次刷新结果。
```

这是整个刷新机制里最关键的并发控制点。

## 八、为什么不能每个过期请求都各自去刷 token

想象一个场景：

页面一进入就并发发了 5 个请求。

此时 token 正好过期，后端这 5 个请求全都返回：

```txt
9999
```

如果没有并发控制，就会发生：

```txt
5 个请求各自去调 refreshToken
```

这会带来很多问题：

- 多次重复刷新，浪费请求。
- 后端刷新接口压力变大。
- 新旧 token 竞争覆盖，状态可能混乱。
- 有的请求拿到旧 token，有的拿到新 token，时序容易乱。

所以项目要保证：

```txt
同一时刻只发起一次 refreshToken 请求
其他过期请求都等待这一次结果
```

这就是 `handleExpiredRequest()` 的存在意义。

## 九、并发控制是怎么实现的

看这一段：

```ts
if (!state.refreshTokenFn) {
  state.refreshTokenFn = handleRefreshToken();
}

const success = await state.refreshTokenFn;
```

这表示：

### 第一个过期请求进来时

```txt
state.refreshTokenFn 为空
-> 发起 handleRefreshToken()
-> 把这个 Promise 存起来
```

### 后续同时到来的过期请求

```txt
发现 state.refreshTokenFn 已经存在
-> 不再重复发 refresh 请求
-> 直接 await 同一个 Promise
```

所以你可以把 `state.refreshTokenFn` 理解成：

```txt
“当前正在进行中的 refreshToken Promise”
```

## 十、`handleRefreshToken()` 真正负责刷新 token

在同一个文件里：

```ts
async function handleRefreshToken() {
  const { resetStore } = useAuthStore();

  const rToken = localStg.get('refreshToken') || '';
  const { error, data } = await fetchRefreshToken(rToken);
  if (!error) {
    localStg.set('token', data.token);
    localStg.set('refreshToken', data.refreshToken);
    return true;
  }

  resetStore();

  return false;
}
```

这里才是真正去调用刷新接口的地方。

它做了四件事：

### 1. 读取本地 `refreshToken`

```ts
const rToken = localStg.get('refreshToken') || '';
```

### 2. 请求刷新接口

```ts
const { error, data } = await fetchRefreshToken(rToken);
```

### 3. 刷新成功时，更新本地 token

```ts
localStg.set('token', data.token);
localStg.set('refreshToken', data.refreshToken);
```

### 4. 刷新失败时，重置认证状态

```ts
resetStore();
return false;
```

## 十一、为什么刷新成功后一定要写回本地存储

这一步非常关键：

```ts
localStg.set('token', data.token);
localStg.set('refreshToken', data.refreshToken);
```

原因有两个。

### 原因 1：后续请求要用新的 token

请求头里的 Authorization 来自：

```ts
getAuthorization()
```

而它又是从：

```ts
localStg.get('token')
```

读取的。

如果你不把新的 token 写回本地存储，那后面请求还是会继续用旧 token，过期问题就无法真正解决。

### 原因 2：刷新后的会话要可持续

不仅是当前这一个失败请求，后面整个会话都要继续工作，所以 token 刷新不能只存在某个临时变量里，必须更新正式存储。

## 十二、为什么刷新成功后还要重发原请求

这是很多人第一次看会忽略的一点。

刷新 token 成功，只代表：

```txt
认证状态恢复了
```

但刚才那个失败请求本身还没有成功。

例如用户本来在请求：

```txt
/systemManage/getUserList
```

因为 token 过期失败了。你现在刷新出新 token，如果不重发原请求，用户还是拿不到列表数据。

所以项目一定要做：

```ts
return instance.request(response.config)
```

意思是：

```txt
拿着刚才那次失败请求的原配置，再发一遍
```

这就是“自动恢复”的关键。

## 十三、为什么要先重写原请求头里的 Authorization

刷新成功后代码是：

```ts
const Authorization = getAuthorization();
Object.assign(response.config.headers, { Authorization });

return instance.request(response.config) as Promise<AxiosResponse>;
```

为什么不直接重发？

因为原来的 `response.config` 里很可能还带着旧 token。

所以要先：

```txt
重新从 localStorage 读取刚刷出来的新 token
覆盖到原请求头
```

再重发原请求。

这一步保证了：

```txt
重发的这次请求用的是新 token，而不是旧 token
```

## 十四、为什么刷新失败后要 `resetStore()`

如果 refreshToken 也失败了，说明当前会话已经没法继续维持。

这时继续留在“已登录”状态只会导致更混乱：

- 页面显示还像已登录
- 请求却一直失败
- 菜单和权限状态也可能不正确

所以项目选择：

```ts
resetStore();
```

它会做完整认证重置：

- 记录上一个用户 id
- 清 token / refreshToken
- 重置 auth store
- 跳登录页
- 重置 route store
- 处理 tabs 缓存

这是一种非常合理的失败兜底策略。

## 十五、为什么 `onError` 里不再提示过期错误

在 [src/service/request/index.ts](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/service/request/index.ts) 的 `onError` 中：

```ts
const expiredTokenCodes = import.meta.env.VITE_SERVICE_EXPIRED_TOKEN_CODES?.split(',') || [];
if (expiredTokenCodes.includes(backendErrorCode)) {
  return;
}
```

意思是：

```txt
如果这次错误本质上是 token 过期，
就不要在这里再弹普通错误提示了
```

为什么？

因为 token 过期不是最终错误结果，它还有“刷新 -> 重发”这条恢复路径。

如果这里直接提示：

```txt
token 已过期
```

用户可能看到一堆无意义的报错，体验很差。

所以项目做了这样的分工：

- `onBackendFail` 负责尝试恢复
- `onError` 在过期场景里尽量静默

## 十六、为什么刷新 token 的接口不能再返回“过期码”

源码注释里写得很清楚：

```ts
// the api `refreshToken` can not return error code in `expiredTokenCodes`,
// otherwise it will be a dead loop, should return `logoutCodes` or `modalLogoutCodes`
```

这句话非常重要。

如果刷新 token 的接口本身也返回：

```txt
9999
```

那流程会变成：

```txt
请求过期 -> 去刷新 token
刷新接口也说 token 过期 -> 再去刷新 token
再过期 -> 再刷新
...
```

这就会形成死循环。

所以刷新接口一旦失败，应该返回：

- 直接登出类 code
- 弹窗登出类 code

而不是继续返回“过期码”。

## 十七、`setTimeout(() => { state.refreshTokenFn = null }, 1000)` 是干什么的

在 `handleExpiredRequest()` 里：

```ts
setTimeout(() => {
  state.refreshTokenFn = null;
}, 1000);
```

它的意图是：

```txt
当前这次刷新完成后，不要立刻把缓存 Promise 清掉，
稍微延迟一点，再允许下一轮新的刷新开始
```

你可以把它理解成一个很轻量的缓冲区。

这样做的直觉是：

- 刚刷新成功后，短时间内后续等待中的请求都能复用同一个结果
- 避免过于紧凑的时序下反复创建新的刷新流程

虽然这里的 1000ms 是经验性写法，但思路很好理解：

```txt
让“同一波过期请求”尽量共享同一轮 refresh 流程
```

## 十八、请求实例上的 `state` 为什么适合放刷新 Promise

在请求实例创建时有：

```ts
defaultState: {
  errMsgStack: [],
  refreshTokenPromise: null
} as RequestInstanceState,
```

虽然当前实现里在 `shared.ts` 中实际用了 `refreshTokenFn` 这个字段名来挂载 Promise，但从设计意图上看，请求实例上的 `state` 就是用来放这类“请求层自己的运行状态”的。

为什么这个状态不放到 Pinia？

因为：

```txt
“当前是否正在刷新 token”
这是请求层内部机制，不是页面状态，不是业务状态，也不需要响应式展示
```

所以把它挂在请求实例自身最合适。

## 十九、这套刷新机制和登录流程的关系

你可以把登录与刷新关系这样理解：

### 登录流程

```txt
账号密码 -> login -> 拿 token / refreshToken -> 拉 userInfo
```

### 刷新流程

```txt
已有会话 -> token 过期 -> refreshToken -> 更新 token / refreshToken -> 重发原请求
```

两者不同点在于：

- 登录流程是“建立会话”
- 刷新流程是“延续会话”

但它们都会操作：

- `token`
- `refreshToken`
- `authStore.resetStore()`

所以它们是同一条认证主线里的两段逻辑。

## 二十、如何手动触发一次 token 过期流程

项目里专门准备了演示页面，在：

[src/views/function/request/index.vue](/Users/codepencil/Documents/project/个人/soybean-admin-element-plus/src/views/function/request/index.vue)

其中：

```ts
async function refreshToken() {
  await fetchCustomBackendError('9999', $t('request.tokenExpired'));
}
```

也就是说，这个页面会主动制造一个：

```txt
后端 code = 9999
```

从而触发：

```txt
expiredTokenCodes -> handleExpiredRequest -> refreshToken 流程
```

这是你学习这套机制最好的实验入口。

## 二十一、整条刷新链路总图

到这里你应该能把整个流程写成：

```txt
业务请求发出
  -> 后端返回 code = 9999
  -> request.onBackendFail()
  -> 识别为 expiredTokenCodes
  -> handleExpiredRequest(request.state)
    -> 如果当前没有刷新动作
      -> handleRefreshToken()
        -> 读取 localStorage.refreshToken
        -> fetchRefreshToken(refreshToken)
        -> 刷新成功
          -> localStg.set('token', newToken)
          -> localStg.set('refreshToken', newRefreshToken)
          -> return true
        -> 刷新失败
          -> authStore.resetStore()
          -> return false
    -> 如果当前已有刷新动作
      -> await 同一个刷新 Promise
  -> 如果刷新成功
    -> 重新读取 Authorization
    -> 覆盖原请求头
    -> 重发原请求
  -> 如果刷新失败
    -> 认证状态重置，用户回到未登录状态
```

如果你能把这条流程讲顺，第 13 课就算真正学会了。

## 二十二、这一课最容易误解的几个点

### 误区 1：token 过期就一定要跳登录页

不一定。只要 `refreshToken` 还有效，就应该优先尝试续期。

### 误区 2：刷新成功就结束了

不对。刷新成功后还必须重发原请求。

### 误区 3：多个过期请求会各自刷新一次 token

这个项目专门通过缓存 Promise 避免了这种情况。

### 误区 4：刷新逻辑应该写在页面里

不对。它属于请求层和认证层的横切逻辑，必须统一处理。

## 二十三、课堂演示

### 演示 1：从过期 code 追到刷新逻辑

打开：

```txt
src/service/request/index.ts
```

找到：

```ts
if (expiredTokenCodes.includes(responseCode)) {
  const success = await handleExpiredRequest(request.state);
  ...
}
```

把这个入口位置记住。

### 演示 2：解释并发控制

打开：

```txt
src/service/request/shared.ts
```

重点看：

```ts
if (!state.refreshTokenFn) {
  state.refreshTokenFn = handleRefreshToken();
}

const success = await state.refreshTokenFn;
```

用自己的话解释为什么这样就能避免重复刷新。

### 演示 3：解释刷新成功后为什么要重发原请求

对照：

```ts
Object.assign(response.config.headers, { Authorization });
return instance.request(response.config);
```

说明如果不重发，会发生什么。

### 演示 4：解释刷新失败后为什么要 `resetStore()`

打开：

```txt
src/store/modules/auth/index.ts
```

回顾 `resetStore()` 会做哪些事。

### 演示 5：在演示页手动触发过期流程

打开：

```txt
src/views/function/request/index.vue
```

点击触发：

```txt
refreshToken
```

观察整个流程。

## 二十四、本课作业

### 作业 1：画 token 刷新流程图

至少包含这些节点：

```txt
onBackendFail
expiredTokenCodes
handleExpiredRequest
handleRefreshToken
fetchRefreshToken
localStg.set(token)
localStg.set(refreshToken)
instance.request(response.config)
resetStore
```

### 作业 2：解释每个函数的职责

写一份笔记：

```md
# token 刷新相关函数职责

- getAuthorization：
- handleRefreshToken：
- handleExpiredRequest：
- fetchRefreshToken：
- resetStore：
```

### 作业 3：解释并发刷新为什么会有问题

写 5-10 行说明：

```txt
如果没有缓存 Promise，而是每个过期请求都自己刷新 token，会发生什么
```

### 作业 4：解释为什么 refresh 接口不能再返回 expired code

写一段自己的理解，说明为什么会死循环。

## 二十五、自检问题

完成本课后，尝试回答：

- token 过期为什么不一定立刻跳登录页？
- 项目是通过什么识别“这是 token 过期”？
- `expiredTokenCodes` 从哪里来？
- `handleExpiredRequest()` 的职责是什么？
- `handleRefreshToken()` 的职责是什么？
- 为什么多个并发过期请求不会重复刷新 token？
- 刷新成功后为什么要更新本地存储？
- 刷新成功后为什么必须重发原请求？
- 为什么要重写原请求头里的 Authorization？
- 刷新失败后为什么要 `resetStore()`？
- 为什么 `onError` 里对过期码选择静默？
- 为什么 refreshToken 接口不能返回 expired code？
- 请求实例上的 `state` 为什么适合存刷新 Promise？

如果这些问题能回答出来，就可以进入第 14 课。

## 二十六、下一课预告

第 14 课会进入路由、菜单与权限体系，重点学习：

- 常量路由和权限路由的区别。
- `route store` 如何初始化路由。
- 动态 addRoute 和 removeRouteFns 的作用。
- 菜单如何从路由生成。
- 为什么 404、403、权限判断都放在路由守卫链路里。

第 13 课理解的是“认证如何续期”，第 14 课会进入“认证完成后，系统如何决定你能访问哪些页面”。
