# WeChat MP unfreeze

微信公众平台 / 小程序管理后台（`mp.weixin.qq.com`）经常一直转圈、左侧菜单出不来，最后弹出 Chrome 的 “Page Unresponsive / 页面无响应”。这个 Chrome 扩展拦掉导致卡死的页面录制脚本，顺带拦掉后台里其他纯上报 / 监控请求，让页面少等一点。

*English: a one-rule Chrome extension that blocks the session-recording script which freezes the WeChat MP admin console. See [English summary](#english).*

## 症状

- 登录后首页、登录页转圈，左侧菜单和“昨日核心数据”一直加载不出来。
- 标签页的加载按钮一直是 ✕，刷新没用。
- 活动监视器里有一个 Chrome 渲染进程一直占 100% CPU，几分钟后弹出“页面无响应”。
- 无痕窗口里几乎每次都中。

跟网络、代理、扩展都无关：在没有任何扩展的干净无头 Chrome 里也能稳定复现。

## 原因

后台页面会加载腾讯的页面录制 / 监控脚本：

```
https://dev.weixin.qq.com/platform-console/proxy/assets/tel/px.min.js
```

它启动时会把整个 DOM 建成一棵树。卡死时暂停主线程，调用栈最里层固定是：

```
getNodePos → addNode → _addTree → runTaskLoop → schedule → addTree → init → start   (px.min.js)
```

`getNodePos` 对每个节点都沿着 `previousSibling` 往回找一个已登记的兄弟节点（格式化后的原代码）：

```js
function getNodePos(node) {
  let parentId, prevId
  if (node.parentNode && idMap.has(node.parentNode)) {
    parentId = idMap.get(node.parentNode)
    let prev = node.previousSibling
    while (prev && !idMap.has(prev)) prev = prev.previousSibling
    if (prev) prevId = idMap.get(prev)
  }
  return { parentId, prevId }
}
```

兄弟节点还没登记时，每个节点都要扫一遍前面所有兄弟，整体变成平方级；再加上任务循环同步跑完、不让出主线程，页面就锁死了。

它在**每个新浏览器会话的第一次加载**时触发（同一会话里再次打开通常正常），所以无痕窗口几乎每次都卡。

## 验证（2026-09-25，Chrome 154，macOS）

用 Puppeteer 反复打开登录后的后台首页，页面主线程 3 秒内无响应即判定卡死：

| 条件 | 结果 |
|---|---|
| 不拦截，新会话第一次加载 | 4 / 4 卡死 |
| 不拦截，同一会话后续加载 | 0 / 12 卡死 |
| 拦截 `px.min.js`，含第一次加载 | 0 / 5 卡死，页面数据正常 |

扩展装上后，`px.min.js` 返回 `ERR_BLOCKED_BY_CLIENT`，其余 `mp.weixin.qq.com` 请求照常 200。

## 安装

1. 下载本仓库（`git clone` 或 Code → Download ZIP 后解压）。
2. Chrome 打开 `chrome://extensions`，打开右上角**开发者模式**。
3. 点**加载已解压的扩展程序**，选择仓库里的 `extension` 文件夹。
4. 用无痕窗口的话：扩展 → 详情 → 打开**在无痕模式下启用**。
5. 关掉所有公众平台标签页再重新打开。

Edge（`edge://extensions`）和其他 Chromium 浏览器步骤相同。

已经在用 uBlock Origin 的话，也可以不装本扩展，在“我的过滤规则”里加一行：

```
||dev.weixin.qq.com/platform-console/proxy/assets/tel/px.min.js$script
```

## 拦截清单

全部规则在 [`extension/rules.json`](extension/rules.json)，只对 `mp.weixin.qq.com` 页面发出的请求生效（`initiatorDomains`），不影响其他网站。

| 规则 | 是什么 | 为什么拦 |
|---|---|---|
| `dev.weixin.qq.com/…/tel/px.min.js` | 页面录制脚本 | 卡死元凶 |
| `res8.wxqcloud.qq.com.cn/obtelemetry*` | 录制 / 遥测 SDK（`phantom.min.js`、`wxtelsdk.min.js`），`px.min.js` 就是它拉起来的 | 源头一起断 |
| `aegis.qq.com` | 腾讯前端监控上报 | 纯上报 |
| `badjs.weixinbridge.com` | 前端错误上报 | 纯上报 |
| `cube.weixinbridge.com/cube/report/` | 业务埋点上报 | 纯上报 |
| `mp.weixin.qq.com/wxamp/cgi/reportclick`、`/cgi/report?`、`/cgi/base/mmdatareport`、`/cgi/wedata/ReportHomeData` | 后台自己的点击 / 曝光埋点 | 纯上报，和真实数据接口抢后端 |

拦截后控制台会多出几条 `ERR_BLOCKED_BY_CLIENT` 和 `Uncaught (in promise) Error: Network Error`，是上报失败的噪音，不影响功能。

实测（登录后，新会话第一次加载，各 4 轮平均）：

| | 左侧菜单 + 核心数据出现 |
|---|---|
| 只拦 `px.min.js` | 约 6.9 s |
| 全部规则 | 约 5.3 s |

首页、版本管理、成员管理、账号设置都正常显示数据。剩下的慢主要是腾讯后端本身：从海外访问，首页 HTML 首字节约 2.7 s，每个数据接口 1–4 s，这部分扩展帮不上。

## 它做了什么，没做什么

- 只有静态的 `declarativeNetRequest` 拦截规则。
- 不读取页面内容、不注入脚本、不申请任何站点权限，自己不发任何网络请求，也没有弹窗——点工具栏上的图标没反应是正常的。
- 被拦的都是监控 / 上报，不影响后台功能。
- 腾讯改了地址或修了问题，规则会自然失效或变得不需要。欢迎提 issue / PR。

## English

The WeChat MP admin console (`mp.weixin.qq.com`) freezes on load because its session-recording script `dev.weixin.qq.com/platform-console/proxy/assets/tel/px.min.js` walks `previousSibling` chains for every node while building its DOM mirror (`getNodePos`), which is quadratic and runs synchronously. It hits the first page load of every fresh browser session, so incognito windows almost always hang. This extension blocks that script plus the console's other telemetry/beacon requests (aegis, badjs, cube reports, first-party click/impression beacons) with static `declarativeNetRequest` rules scoped to `mp.weixin.qq.com`; no page access, no host permissions. Load `extension/` via `chrome://extensions` → Developer mode → Load unpacked.

## License

MIT
