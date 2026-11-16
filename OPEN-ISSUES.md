# 三国杀私服（基于无名杀 noname）未解决问题清单

> 写给外部模型分析用。所有敏感信息（域名、账号、内网地址、密钥）已脱敏。
> 日期：2026-11-16

---

## 0. 项目背景

- 私有三国杀联机服，**基于开源引擎「无名杀 noname」v1.11.6.1**（commit `0b7e4be4`）
- 引擎本身**不做任何修改**，运行时通过哈希清单校验（`UPSTREAM.json` 逐文件 sha256）
- 我们的定制全部在**外层**：
  - `v1/browser/play.css` —— 覆盖样式
  - `v1/browser/play-ui.mjs` / `native-interaction-ui.mjs` —— 运行时注入的 JS 层
  - `v1/multiroom/gateway.mjs` —— 多房间网关（Node ESM）
- 部署环境：NAS + Docker，游戏跑在**服务端无头 Chromium**（Playwright）里，客户端是 PWA
- 手机视口：**800×360 CSS px**（iPhone，devicePixelRatio 3）；iPad 1024×768 不受影响

### 引擎的关键机制（已确认）

1. **整屏缩放**（`noname/ui/index.js` 的 `updatez`）：
   ```js
   if (宽 < 900 || 高 < 500)
     zoom = min(round(宽/98)/10, round(高/50)/10);   // 800×360 → min(0.8, 0.7)=0.7?
   document.body.style.width  = 宽/zoom + 'px';
   document.body.style.height = 高/zoom + 'px';
   document.body.style.transform = 'scale(' + zoom + ')';
   ```
   （实际 844×390 时得到 0.8，800×360 时我们实测为 0.71 左右）

2. **按钮条定位**（`updatec`）：
   ```js
   var offset = -length / 2;          // length = 该行所有按钮总宽
   control.style.transform = `translateX(${offset}px)`;
   ```
   行容器 `left:50%`，所以**总宽超过屏宽时两端会跑出屏幕**。
   上游在 2026-11-10 的 #4528 修了这个问题（改成溢出时左对齐 + 横向滚动）。

3. **引擎对象不暴露到 window**：`window.game` / `window.ui` 都拿不到，
   只有 `window.zyt` 这个聚合对象（含 `zyt.ui`、`zyt.lib`、`zyt.game`）。

---

## 1. 【未解决】国战「先驱」点击后 UI 卡死、被 AI 接管

### 现象（玩家实测）
- 4 人 + 2 AI 国战，双将（周瑜 + 小乔）
- 点击「先驱」标记 → 画面闪一下 → **中间那排按钮消失，点什么都没反应**
- 过一会儿系统**自动**对某个角色使用了「反间」（玩家从未选择）
- 刷新页面后才能恢复

### 已尝试
1. 用上游 #4528 的思路做了运行时补丁：包装 `ui.updatec`，检测溢出后加 class 并把按钮条移到左边缘、允许横向滚动
   - 单元测试 6/6 通过（3/5/7 个按钮不触发，9/12/18 个按钮触发且左边界=0、可滚动）
   - **线上玩家复现，问题依旧**
2. 因此判断：**不是（或不只是）按钮溢出问题**

### 待分析的关键点
- 「先驱」是国战的**标记**（mark），点击后引擎会走 `useCard`/`chooseToUse` 流程
- 画面"闪一下"很像**对话框/控制条重建**（`ui.updatec` + `ui.update` 同时触发）
- "按钮消失"可能是：`#control` 被清空后没有重新渲染，或对话框处于 `paused` 状态
- "自动使用反间"：可能是超时（`choose_timeout`）后由 AI 托管接管

### 需要 GPT 帮判断
1. 国战里点击「先驱」标记的正确代码路径是什么？（`mode/guozhan` 已被打包成 `mode/guozhan.js`）
2. 什么情况下 `#control` 会变空且不再恢复？是否与 `_status.noupdatec` 或 `ui.dialog` 状态有关？
3. 我们的补丁包装 `ui.updatec` 会不会与引擎内部直接调用 `ui.updatec()` 的时机冲突（比如在 dialog 未就绪时测量）？

### 相关代码（我们的补丁）
```js
// v1/browser/native-interaction-ui.mjs
export function installControlOverflow(z){
  try{
    const ui=z&&z.ui;
    if(!ui||typeof ui.updatec!=='function'||ui.__zytControlOverflow)return;
    ui.__zytControlOverflow=true;
    const native=ui.updatec.bind(ui);
    ui.updatec=function(...args){
      const out=native(...args);
      try{
        const control=ui.control;
        if(!control)return out;
        const rows=[...control.children].filter(n=>!n.classList.contains('controlscrollbar'));
        if(!rows.length){control.classList.remove('zyt-control-overflow');control.style.removeProperty('--zyt-control-shift');return out;}
        const box=control.getBoundingClientRect();
        let minLeft=Infinity,maxRight=-Infinity;
        for(const row of rows){const r=row.getBoundingClientRect();if(!r.width&&!r.height)continue;minLeft=Math.min(minLeft,r.left);maxRight=Math.max(maxRight,r.right);}
        if(!Number.isFinite(minLeft))return out;
        if(minLeft<box.left-1||maxRight>box.right+1){
          control.classList.add('zyt-control-overflow');
          control.style.setProperty('--zyt-control-shift',Math.round(minLeft-box.left)+'px');
        }else{
          control.classList.remove('zyt-control-overflow');
          control.style.removeProperty('--zyt-control-shift');
        }
      }catch{}
      return out;
    };
  }catch{}
}
```

---

## 2. 【未解决】中央提示框：遮挡手牌 + 仍是不透明

### 现象
- 提示框（引擎的 `.dialog`）在屏幕中间，**压住手牌和牌堆**
- 我们给 `.dialog` 设了 `background: rgba(14,12,10,0.55) !important`，**玩家看到的仍然是不透明深色块**

### 我们的分析
- 提示框**内部还有一层 `.content-container`**，它自己画背景（我们 `.zyt-game` 层的面板贴图 `--zyt-panel`）
- 只改外层的 `background` 不会让内层透明

### 试过的没成功的做法
```css
html.zyt-game #arena > .dialog { background: rgba(14,12,10,.55) !important; backdrop-filter: blur(2px); }
```
- 效果：外框颜色变了，但视觉上仍是一块实心深色

### 需要 GPT 帮判断
1. 无名杀的 `.dialog` 结构里，真正画背景的是哪一层？（`.content-container`？`.dialog > .menubar`？）
2. 有没有官方的"提示栏/消息条"替代方案（不用大对话框），例如 `ui.create.menu` / `#historybar` / popup？
3. 如果要让提示只显示一行文字+按钮，最稳的做法是覆盖 CSS 还是替换 `ui.create.dialog`？

### 另一个相关问题
弹出的「取消」按钮**独立于对话框**（在 `#control` 里），位置在对话框上沿，**压住了对话框文字**。
- 已确认它是 `#control > .control > div`，不是对话框的子元素
- 想把它挪到对话框右侧/下方，但 `#control` 的位置是引擎按行计算的

---

## 3. 【未解决】「没能进入牌桌」兜底提示被触发

### 现象
- 手机进入游戏后弹出我们自己的提示："没能进入牌桌 / 页面加载超时…"
- 玩家**刷新几次后能进**

### 机制（我们加的）
```js
// play-ui.mjs
const ready=setInterval(()=>{ if(!window.zyt?.ready)return; booted=true; clearInterval(ready); ... },100);
const bootTimeout=setTimeout(()=>{ if(booted)return; console.error('BOOT_TIMEOUT',{...}); showBootFailure(); },25000);
```
即：引擎 25 秒内没把 `window.zyt.ready` 置真，就弹提示。

### 已排除
- 不是崩溃（刷新即好）
- 不是登录过期

### 待分析
1. 引擎在什么情况下会**长时间不 ready**？（首次加载慢？Service Worker 旧缓存？）
2. iOS PWA 里 Service Worker 更新策略会不会导致拿到旧 bundle 而一直不 ready？
3. 有没有办法在客户端判断"正在加载"和"卡住了"，而不是靠固定 25 秒？

---

## 4. 【未解决】引擎自带的「退出房间」无效 / 白屏

### 现象
- 对局结束后点引擎的「退出房间」→ 有时**白屏**（只剩我们注入的按钮）
- 点了没反应，屏幕闪一下又回到牌桌
- **设置里的「解散房间」是有效的**

### 引擎实现（已确认）
```js
// mode/connect.js
ui.exitroom = ui.create.system("退出房间", function () {
  game.saveConfig("directstartmode");
  game.reload();
}, true);
```
`game.reload()` 重新请求 `/`，而我们的网关对"名下还有房间"的账号会把 `/` **重新代理进房间** → 看起来没退出。

### 我们已做的
- 自己注入了一个「退出房间」按钮，走网关 API `POST /api/rooms/leave`（这个有效）
- 但玩家觉得它多余，要求删除
- 白屏的根因仍未定位（我们的兜底提示能避免"白屏无路可走"，但没治病）

### 待分析
1. `game.reload()` 后引擎重新初始化，什么情况下会**永远不 ready**（与第 3 条同源？）
2. 能否通过 URL 参数/标记让网关在重载时直接回大厅，而不是重新代理进房间？

---

## 5. 【未解决】诸葛亮「观星」在国战里无法调整牌序

### 现象
- 国战里选诸葛亮，观星阶段**拖不动牌**，无法改变顺序
- 身份局里是否正常未验证

### 我们的相关代码
```js
export function installGameTouchPolicy(){
  document.addEventListener('gesturestart',e=>{if(html.classList.contains('zyt-game'))e.preventDefault();},{passive:false});
  document.addEventListener('touchstart',e=>{if(html.classList.contains('zyt-game')&&e.touches.length>1){e.preventDefault();e.stopPropagation();}},{passive:false,capture:true});
  document.addEventListener('touchmove',e=>{if(html.classList.contains('zyt-game')&&e.touches.length>1)e.preventDefault();},{passive:false});
}
```
只拦截**多指**手势，但 `touchstart` 是 **capture + stopPropagation**，可能影响单指拖拽的初始化。

### 待分析
1. 观星的排序是 HTML5 drag 还是 pointer 事件？我们的 capture 监听会不会截断它？
2. 上面那段 touch 策略对单指拖拽有没有副作用？

---

## 6. 已知的引擎版本情况（供参考）

- 我们锁定 **v1.11.6.1**（2026-10-26）
- 上游 `main` 已到 **2026-11-13**（27 个提交），其中包含：
  - `feat(ui): ui.control 单行按钮放不下时支持横向滚动 (#4528)` ← 与问题 1 相关
  - `fix(connect): 规范化联机地址输入，并修复 iOS 退出报错 (#4590)` ← 与问题 4 相关
  - `feat(guozhan): 优化角色名与主副将对应关系 (#4597)`
- **无法直接升级**：运行时由哈希清单逐文件校验，且上游新构建把 `mode/guozhan/` 目录**打包成单文件 `mode/guozhan.js`**，路径变了，升级需要改校验逻辑 + 重新生成清单
- 上游 `build-output` 分支有自动构建产物（1.17GB），我们已下载验证含上述修复

---

## 7. 想问 GPT 的核心问题（按优先级）

1. **问题 1（最痛）**：国战点「先驱」后按钮消失、被 AI 接管，最可能的根因是什么？怎么在不改引擎 bundle 的前提下修复？
2. **问题 2**：无名杀的对话提示框怎么做到真正半透明/无边框（要让下面的牌可见）？是否有官方轻量提示替代方案？
3. **问题 4+3**：`game.reload()` 后引擎不 ready / 白屏的常见原因？如何优雅退出房间？
4. **问题 5**：观星拖拽失效是否与我们的 touch 拦截有关？
5. 如果把引擎从 v1.11.6.1 升到 main（2026-11-13），有哪些**兼容性陷阱**需要提前知道？（打包结构变化、`mode/guozhan.js`、`ui.updatec` 改动）

---

## 8. 附：我们的关键覆盖样式（节选）

```css
@media (max-width: 900px), (max-height: 500px) {
  html.zyt-game body {
    --zyt-zoom: 0.72;
    width: calc(100vw / var(--zyt-zoom)) !important;
    height: calc(100vh / var(--zyt-zoom)) !important;
    transform: scale(var(--zyt-zoom)) !important;
    transform-origin: 0 0 !important;
  }
  html.zyt-game #arena > .dialog {
    background: rgba(14, 12, 10, 0.55) !important;
    border-color: rgba(185, 150, 94, 0.45) !important;
    backdrop-filter: blur(2px);
  }
  html.zyt-game #control.zyt-control-overflow { overflow-x: auto !important; overflow-y: hidden !important; }
  html.zyt-game #control.zyt-control-overflow > .control { left: calc(50% - var(--zyt-control-shift, 0px)) !important; }
}
```

---

## 9. 环境与验证方式

- 所有改动都通过 Playwright 在真实页面注入 CSS/JS 验证，用 `getBoundingClientRect` + 重叠面积计算做断言
- 部署经过：CI 构建 → 五房隔离压测 → 切换 → 线上验收（大厅/房间/8人/1v1/国战）
- 现状：布局改动已上线（座位 86×144、提示框半透明参数已提交），但玩家实测仍反馈"提示框不透明、遮挡手牌"
