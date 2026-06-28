# PATCHES — 对生成文件 `support.js` 的手工改动

`support.js` 是设计工具**生成**的运行时（文件顶部标了 "do not edit"）。但本仓库对它做了
**一处必要的手工补丁**——把 React/ReactDOM 从 CDN 改成本地 `vendor/`。
这是本工具能"无 CDN、网络差也能开、可离线、部署到任何静态主机"的关键。

> ⚠️ **一旦重新生成 `support.js`，这个补丁会丢失，React 会退回从 unpkg CDN 加载（或 SRI 对不上而白屏）。**
> 重新生成后，请照下面重新打一遍。

## 补丁内容（React 18.3.1 vendoring + SRI）

`support.js` 约 **第 1424–1427 行**的四个常量，改成指向本地文件：

```js
var REACT_URL     = "./vendor/react.production.min.js";
var REACT_SRI     = "sha384-DGyLxAyjq0f9SPpVevD6IgztCFlnMF6oW/XQGmfe+IsZ8TqEiDrcHkMLKI6fiB/Z";
var REACT_DOM_URL = "./vendor/react-dom.production.min.js";
var REACT_DOM_SRI = "sha384-gTGxhz21lVGYNMcdJOyq01Edg0jhn/c22nsx0kyqP0TxaV5WVdsSH1fSDUf5YJj1";
```

- 固定 **React 18.3.1** UMD production 版，已下到 [`vendor/`](vendor/)。
- 两个 SRI 哈希与 `vendor/` 里的文件**逐字节一致**（已校验）。`support.js` 的 `loadScript()`
  会把它设到 `<script integrity>`，哈希对不上浏览器就拒绝执行 → 白屏。

## 换 React 版本时：重算 SRI

```bash
# 对每个 vendored 文件算 sha384，前面加 "sha384-" 填回上面的常量
openssl dgst -sha384 -binary vendor/react.production.min.js     | openssl base64 -A   # → REACT_SRI
openssl dgst -sha384 -binary vendor/react-dom.production.min.js  | openssl base64 -A   # → REACT_DOM_SRI
```

## 检测补丁是否丢了（重新生成后必查）

```bash
grep -n "unpkg.com\|esm.sh\|cdn" support.js     # 若 REACT_URL/REACT_DOM_URL 又变回 CDN，补丁丢了
grep -n "REACT_URL\|REACT_DOM_URL" support.js    # 应指向 ./vendor/
```

## 取舍：SRI 留还是去？

这两个文件是**同源**的（和页面同一来源），SRI 的防篡改价值有限，却带来"换版本忘了重算哈希就白屏"
的风险。两种都行：
- **保留 SRI（当前选择）** —— 多一层完整性校验，代价是换 React 版本时必须同时更新哈希（见上）。
- **去掉 SRI** —— 把 `support.js` 里 `loadScript` 设 `s.integrity` 那行去掉（或把两个 `*_SRI` 置空），
  同源脚本不再做完整性校验，彻底消除"换版本白屏"这个坑。

## 旁注：`support.js` 里的 Babel CDN（不影响本工具）

`support.js` 约第 989 行有 `BABEL_URL = "https://unpkg.com/@babel/standalone..."`，**但只有当加载
外部 `.jsx`/`.tsx` 脚本时才会去取它**。本工具用的是内联的非 JSX `<script type="text/x-dc">`
（`kind="js"`，不走 Babel），所以 Babel **永不加载**（网络抓包确认只有 react/react-dom/qrcode/support 4 个脚本）。
→ 别给本工具加 `.jsx`/`.tsx` 外部脚本，否则会引入这个 CDN 依赖、破坏离线。

## 自测

改完 `support.js` 或任何静态资源后，用 [`tests.html`](tests.html) 跑一遍核心算法
（TOTP RFC 6238 向量 / base32 往返 / 迁移码解码）。需经 `localhost` 或 HTTPS 打开（Web Crypto 要安全上下文）。
