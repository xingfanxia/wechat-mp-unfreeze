# WeChat MP unfreeze

微信公众平台 / 小程序管理后台（`mp.weixin.qq.com`）经常一直转圈、左侧菜单出不来，最后弹出 Chrome 的 “Page Unresponsive / 页面无响应”。这个 Chrome 扩展只做一件事：拦掉导致卡死的页面录制脚本。

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

## 它做了什么，没做什么

- 只有一条 [`declarativeNetRequest`](extension/rules.json) 拦截规则，只拦这一个脚本 URL。
- 不读取页面内容、不注入脚本、不申请任何站点权限，没有网络请求。
- 被拦的是监控 / 录制上报，不影响后台功能。
- 腾讯如果改了脚本地址或修了这个问题，这条规则会自然失效或变得不需要。欢迎提 issue / PR 更新规则。

## English

The WeChat MP admin console (`mp.weixin.qq.com`) freezes on load because its session-recording script `dev.weixin.qq.com/platform-console/proxy/assets/tel/px.min.js` walks `previousSibling` chains for every node while building its DOM mirror (`getNodePos`), which is quadratic and runs synchronously. It hits the first page load of every fresh browser session, so incognito windows almost always hang. This extension blocks that one script with a single `declarativeNetRequest` rule; no page access, no host permissions. Load `extension/` via `chrome://extensions` → Developer mode → Load unpacked.

## License

MIT
