---
name: xms-browser-interaction
description: XMS 系统浏览器交互技术指南。涵盖在 QoderWork Agent 中模拟鼠标点击、键盘输入、元素定位的三种核心方法及其适用场景。适用于需要在 XMS (cs-packet.i4px.com) 或类似内部系统中执行自动化操作的任务。
---

# XMS 浏览器交互技术指南

## 概述

QoderWork Agent 在 XMS 系统中模拟用户操作有**三种核心方法**，按优先级排列：

| 优先级 | 方法 | 工具 | 适用场景 |
|--------|------|------|---------|
| 1 (首选) | JavaScript DOM 操作 | `javascript_tool` | 已知元素 ID/CSS 选择器、表单填写、按钮点击 |
| 2 (辅助) | 坐标点击 | `computer` | 元素无 ID、动态渲染、Canvas/iframe 内操作 |
| 3 (定位) | 文本搜索 | `find` | 通过可见文字定位元素获取坐标 |

---

## 方法一：JavaScript DOM 操作（首选）

### 原理

通过 `javascript_tool`（action: `javascript_exec`）直接在页面上下文中执行 JavaScript 代码。可以调用 DOM API 操作任何元素，不依赖坐标、不受窗口大小影响。

### 为什么是首选

- **可靠性最高**：不依赖窗口尺寸（即使 256x116 的极小窗口也能正常工作）
- **精确**：通过 ID 或 CSS 选择器直接定位，不会点错位置
- **快速**：无需截图、无需等待渲染
- **可验证**：操作后可立即通过 JS 检查结果

### 基础用法

#### 点击按钮

```javascript
// 通过 ID 点击
document.getElementById('abnormalSearch').click()

// 通过 CSS 选择器点击
document.querySelector('.export-btn').click()

// 通过文本内容查找并点击
Array.from(document.querySelectorAll('button, a, span'))
  .find(el => el.textContent.trim().includes('导出原因'))
  ?.click()
```

#### 填写输入框

```javascript
// 直接设置值（不够！必须触发事件）
const input = document.getElementById('username');
input.value = 'myvalue';

// 必须触发事件，否则前端框架（Vue/React/Angular）不会感知变化
['input', 'change', 'keyup'].forEach(evt => {
  input.dispatchEvent(new Event(evt, { bubbles: true }));
});
```

#### 清空输入框

```javascript
// XMS 上架组织字段的清空方法
const show = document.getElementById('txtUpShelfOgCodeShow');
const hidden = document.getElementById('txtUpShelfOgCode');
if (show) { show.value = ''; show.dispatchEvent(new Event('input', {bubbles:true})); }
if (hidden) { hidden.value = ''; hidden.dispatchEvent(new Event('change', {bubbles:true})); }
```

#### 展开菜单/点击链接

```javascript
// 点击左侧菜单项（通过文本查找）
Array.from(document.querySelectorAll('.nav-item, .menu-item, a'))
  .find(el => el.textContent.includes('异常件管理'))
  ?.click()
```

#### 读取页面信息

```javascript
// 读取表格内容
const rows = document.querySelectorAll('.next-drawer table tr');
const firstRow = rows[1]; // 跳过表头
firstRow?.textContent

// 读取进度百分比
const cells = rows[1]?.querySelectorAll('td');
cells[2]?.textContent.trim() // 假设第3列是进度
```

### 事件触发的完整模式

XMS 使用的前端框架需要完整的事件链才能正确响应：

```javascript
function simulateInput(element, value) {
  // 聚焦
  element.focus();
  element.dispatchEvent(new Event('focus', { bubbles: true }));
  
  // 设置值
  element.value = value;
  
  // 触发事件链
  element.dispatchEvent(new Event('input', { bubbles: true }));
  element.dispatchEvent(new Event('change', { bubbles: true }));
  element.dispatchEvent(new Event('keyup', { bubbles: true }));
  
  // 失焦（触发校验）
  element.dispatchEvent(new Event('blur', { bubbles: true }));
}

function simulateClick(element) {
  element.dispatchEvent(new MouseEvent('mousedown', { bubbles: true }));
  element.dispatchEvent(new MouseEvent('mouseup', { bubbles: true }));
  element.dispatchEvent(new MouseEvent('click', { bubbles: true }));
}
```

### 处理异步加载

```javascript
// 等待元素出现（轮询方式）
function waitForElement(selector, timeout = 10000) {
  return new Promise((resolve, reject) => {
    const start = Date.now();
    const check = () => {
      const el = document.querySelector(selector);
      if (el) return resolve(el);
      if (Date.now() - start > timeout) return reject('Timeout');
      setTimeout(check, 500);
    };
    check();
  });
}

// 用法
await waitForElement('#myButton').then(el => el.click())
```

---

## 方法二：坐标点击（`computer` 工具）

### 原理

通过 `computer` 工具先截图，识别目标元素的屏幕坐标，再用 `left_click` 在该坐标上执行点击。模拟真实鼠标行为。

### 适用场景

- 元素无法通过 DOM 访问（如 iframe 内、Shadow DOM、Canvas）
- 需要视觉确认后再操作（如验证码、图形界面）
- 弹窗/Modal 内容无法通过 `read_page` 读取时
- 动态生成的元素没有稳定的 ID 或选择器

### 基础流程

```
1. computer(action: "screenshot", tabId: xxx)     → 获取当前页面截图
2. 分析截图，确定目标元素的像素坐标 (x, y)
3. computer(action: "left_click", coordinate: [x, y], tabId: xxx)  → 点击
4. computer(action: "screenshot", tabId: xxx)     → 验证点击结果
```

### 关键参数

| action | 用途 | 必要参数 |
|--------|------|---------|
| `screenshot` | 截图查看当前页面 | `tabId` |
| `left_click` | 左键单击 | `coordinate: [x, y]`, `tabId` |
| `double_click` | 双击 | `coordinate: [x, y]`, `tabId` |
| `right_click` | 右键点击 | `coordinate: [x, y]`, `tabId` |
| `type` | 键盘输入文字 | `text`, `tabId` |
| `key` | 按特定键 | `text: "Enter"`, `tabId` |
| `scroll` | 滚动页面 | `coordinate`, `scroll_direction`, `tabId` |
| `wait` | 等待 | `duration` (秒), `tabId` |
| `zoom` | 放大查看区域 | `region: [x0, y0, x1, y1]`, `tabId` |

### 实际案例：点击下载中心的下载按钮

```
步骤 1: computer(action: "screenshot", tabId: 123)
   → 截图显示下载中心弹窗，"下载"按钮在坐标 (850, 320) 附近

步骤 2: computer(action: "zoom", region: [800, 300, 900, 340], tabId: 123)
   → 放大确认按钮精确位置在 (855, 318)

步骤 3: computer(action: "left_click", coordinate: [855, 318], tabId: 123)
   → 点击下载

步骤 4: computer(action: "wait", duration: 3, tabId: 123)
   → 等待下载启动

步骤 5: computer(action: "screenshot", tabId: 123)
   → 验证是否触发了下载
```

### 坐标点击注意事项

1. **光标对准中心**：始终点击元素的中心位置，不要点在边缘
2. **先截图再点**：每次点击前必须截图确认坐标，页面可能已变化
3. **窗口大小影响**：坐标依赖窗口尺寸，256x116 窗口下坐标会完全不同
4. **等待加载**：点击后需等待页面响应，使用 `wait` 或多次截图确认
5. **Zoom 辅助**：对小元素使用 `zoom` 放大区域确认精确坐标

---

## 方法三：文本搜索定位（`find` 工具）

### 原理

通过 `find` 工具按文本内容搜索页面元素，返回带有 `ref` 引用的元素列表。获得 `ref` 后可用于 `computer` 的 `scroll_to` 或直接通过 `ref` 点击。

### 适用场景

- 不知道元素的 ID 或 CSS 选择器
- 需要按可见文字定位按钮或链接
- 辅助 `computer` 确定点击目标

### 用法

```
find(query: "导出原因", tabId: 123)
  → 返回匹配元素列表，包含 ref_id 和坐标信息

computer(action: "left_click", ref: "ref_5", tabId: 123)
  → 通过 ref 点击该元素
```

### 限制

- **中文支持有限**：`find` 对中文关键词匹配不稳定，有时搜不到
- **只能搜可见文本**：对 `placeholder`、`title` 等属性匹配不全
- **Modal 内搜不到**：与 `read_page` 一样，无法感知弹窗内容
- 建议：优先用 JS DOM 查询，`find` 作为辅助验证

---

## XMS 系统特有的交互模式

### 1. 左侧菜单展开

XMS 左侧菜单是折叠式的，需要先点击父菜单再点子菜单：

```javascript
// 方法1：JS 直接操作（推荐）
// 先展开父菜单
Array.from(document.querySelectorAll('.ant-menu-submenu-title, .nav-item'))
  .find(el => el.textContent.includes('异常件管理'))
  ?.click();

// 等待子菜单出现后点击
setTimeout(() => {
  Array.from(document.querySelectorAll('.ant-menu-item, a'))
    .find(el => el.textContent.includes('自有业务异常件管理'))
    ?.click();
}, 500);
```

### 2. 上架组织输入框

这是 XMS 中最容易踩坑的交互：

```javascript
// 关键点：存在两个字段
// txtUpShelfOgCodeShow = 可见输入框，显示"客户服务部"
// txtUpShelfOgCode = 隐藏字段，存储实际值"D000414"

// 方法1：直接点击输入框右侧（文字会自动消失）
// 用 computer 点击 txtUpShelfOgCodeShow 输入框内部、文字右侧

// 方法2：JS 强制清空（兜底）
const show = document.getElementById('txtUpShelfOgCodeShow');
const hidden = document.getElementById('txtUpShelfOgCode');
show.value = '';
hidden.value = '';
['input', 'change'].forEach(evt => {
  show.dispatchEvent(new Event(evt, { bubbles: true }));
  hidden.dispatchEvent(new Event(evt, { bubbles: true }));
});
```

### 3. 弹窗/Drawer 内操作

XMS 的「下载中心」是一个 Drawer 弹窗，`read_page` 看不到其内容：

```javascript
// 读取 Drawer 内表格
const drawer = document.querySelector('.next-drawer') || document.querySelector('.drawer');
const rows = drawer?.querySelectorAll('table tr');

// 第一行是表头，第二行是最新数据
const dataRow = rows?.[1];
const cells = dataRow?.querySelectorAll('td');

// 读取各列
const taskName = cells?.[0]?.textContent.trim();
const fileSize = cells?.[1]?.textContent.trim();
const progress = cells?.[2]?.textContent.trim();
const status = cells?.[3]?.textContent.trim();
const createTime = cells?.[4]?.textContent.trim();
```

### 4. 确认对话框

XMS 登录时可能弹出「已在别处登录」的确认框：

```javascript
// 查找并点击确认按钮
const confirmBtn = Array.from(document.querySelectorAll('button, .ant-btn'))
  .find(el => ['是', '确定', '确认', 'OK', 'Yes'].includes(el.textContent.trim()));
if (confirmBtn) confirmBtn.click();
```

---

## 三种方法的决策树

```
需要在 XMS 中执行操作
│
├─ 元素有已知的 ID？
│  └─ YES → 使用 javascript_tool：document.getElementById('xxx').click()
│
├─ 元素有可用的 CSS 选择器？
│  └─ YES → 使用 javascript_tool：document.querySelector('.xxx').click()
│
├─ 可以通过文本内容找到？
│  └─ YES → 使用 javascript_tool：
│           Array.from(document.querySelectorAll('*'))
│             .find(el => el.textContent.includes('目标文字'))?.click()
│
├─ 以上都不行，元素在 DOM 中不可访问？
│  └─ YES → 使用 computer：
│           1. screenshot 截图
│           2. 分析坐标
│           3. left_click 点击
│
└─ 需要先找到元素再决定？
   └─ YES → 使用 find 搜索 → 获得 ref → 用 computer 的 ref 参数点击
```

---

## 常见坑与解决方案

| 问题 | 现象 | 原因 | 解决方案 |
|------|------|------|---------|
| 点击无反应 | `.click()` 执行了但页面没变化 | 前端框架绑定了 mousedown/mouseup 而非 click | 使用 `simulateClick()` 触发完整事件链 |
| 输入框值不生效 | `.value` 设置了但表单校验报空 | 缺少 input/change 事件 | 必须 `dispatchEvent` 触发事件 |
| 弹窗内容读不到 | `read_page` 返回空 | Modal/Drawer 不在主 DOM 树可见区域 | 用 `javascript_tool` 查询 `.next-drawer` |
| 坐标点偏了 | `computer` 点击了错误位置 | 页面滚动或布局变化 | 每次操作前重新截图确认 |
| 中文搜索失败 | `find("上架组织")` 返回空 | `find` 对中文匹配不稳定 | 改用 JS: `document.querySelectorAll('*')` 遍历 |
| 小窗口下坐标不对 | 窗口 256x116 时 computer 点错 | 坐标基于可视区域，小窗口中元素被压缩 | 优先用 JS 操作（不依赖坐标）；或先 resize |
| 等待不够 | 点击后立即操作下一步失败 | 异步加载还没完成 | `computer(wait)` 或 JS `setTimeout`/轮询 |

---

## 完整交互示例：从登录到导出

```
1. tabs_context_mcp → 获取浏览器标签
2. javascript_tool: window.innerWidth + ',' + window.innerHeight → 检查窗口
3. navigate: http://cs.packet.i4px.com/ → 打开 XMS
4. javascript_tool: document.URL → 检查是否跳转到 SSO
5. javascript_tool: [注入账号密码 + 触发事件 + 点击登录] → 登录
6. computer: wait 5s → 等待跳转
7. javascript_tool: document.URL → 确认已到达首页
8. javascript_tool: [点击菜单"异常件管理"] → 展开菜单
9. computer: wait 1s
10. javascript_tool: [点击"自有业务异常件管理"] → 进入页面
11. computer: wait 2s
12. javascript_tool: [清空上架组织] → 清除筛选条件
13. javascript_tool: document.getElementById('abnormalSearch').click() → 查询
14. computer: wait 3s → 等待数据加载
15. javascript_tool: [查找并点击"导出原因(新)"按钮] → 触发导出
16. computer: wait 2s
17. javascript_tool: [点击"下载中心"] → 打开下载弹窗
18. javascript_tool: [查询 .next-drawer 表格读取进度] → 监控进度
```

---

## 关键原则

1. **JS 优先，截图兜底**：能用 `javascript_tool` 的绝不用 `computer`
2. **事件完整性**：设置值后必须触发事件，点击时考虑完整事件链
3. **操作前验证**：每一步之前确认前一步已成功（检查 URL、DOM 状态）
4. **容错等待**：异步操作后必须等待，用轮询而非固定延时
5. **小窗口兼容**：JS 操作不受窗口大小影响，是锁屏/最小化场景下的唯一可靠方案
