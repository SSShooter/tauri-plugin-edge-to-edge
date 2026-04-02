# Tauri Plugin Edge-to-Edge: iOS 实现分析分析

本文档详细分析了 `tauri-plugin-edge-to-edge` 插件在 iOS 平台上的实现机制，以及如何通过 JavaScript (JS) 和 CSS 与该插件进行交互。

## 1. iOS 实现方式概览

该插件的 iOS 实现位于 `ios/Sources/EdgeToEdgePlugin.swift`。其核心策略是**禁用系统自动处理，改为手动管理 Safe Area 和 Keyboard 行为**，从而实现真正的全屏沉浸式体验。

### 核心技术点：

- **Webview 透明化**：将 `WKWebView` 及其内部 `UIScrollView` 的背景色设为透明，并禁用不透明度 (`isOpaque = false`)。
- **禁用系统自动缩进**：
  ```swift
  webview.scrollView.contentInsetAdjustmentBehavior = .never
  webview.scrollView.automaticallyAdjustsScrollIndicatorInsets = false
  ```
  这是实现 Edge-to-Edge 的关键，它阻止了 iOS 系统在状态栏或键盘出现时自动挤压页面内容。
- **键盘监听与手动管理**：
  - 移除了 WebView 默认的键盘监听器，避免系统默认的页面滚动和跳动行为。
  - 注册了自定义的键盘监听器（`keyboardWillShow`, `keyboardDidShow` 等）。
  - 在键盘弹出时，手动调整页面边距，并通过 JS 注入最新的 Safe Area 数据。
- **防止页面跳动 (Scroll Lock)**：
  - 通过实现 `UIScrollViewDelegate`，在键盘弹出期间强行将 `contentOffset` 锁定为 `(0, 0)`，有效解决了 iOS WebView 常见的“键盘弹出导致页面上移并留下空白”的问题。
- **主动式数据注入**：
  - 插件启动后的前 5 秒会进行周期性注入，确保无论页面加载快慢，CSS 变量和事件都能及时到位。

---

## 2. JavaScript (JS) 如何使用事件

插件通过 `window.dispatchEvent` 在 JS 层级分发自定义事件，使开发者能够实时监听屏幕和键盘状态的变化。

### 被触发的事件：

- **事件名称**：`safeAreaChanged`
- **携带数据**：`event.detail` 对象包含以下字段：
  - `top`: 顶部安全区域高度（px）
  - `right`: 右侧安全区域（px）
  - `bottom`: 底部安全区域（px，包含键盘高度）
  - `left`: 左侧安全区域（px）
  - `keyboardHeight`: 当前键盘高度（px）
  - `keyboardVisible`: 键盘是否可见（boolean）

### JS 使用示例：

```javascript
window.addEventListener("safeAreaChanged", (event) => {
  const { top, bottom, keyboardHeight, keyboardVisible } = event.detail;

  console.log("安全区域变化:", { top, bottom });

  if (keyboardVisible) {
    console.log("键盘已弹出，高度为:", keyboardHeight);
  } else {
    console.log("键盘已隐藏");
  }
});
```

---

## 3. CSS 如何使用变量

插件直接将数据注入到 HTML 根层级 (`document.documentElement.style`) 的 CSS 变量中，这意味着你可以在 CSS 中直接通过 `var()` 函数引用它们，且具有全局响应性。

### 注入的常用变量：

| 变量名                     | 说明                                                               |
| :------------------------- | :----------------------------------------------------------------- |
| `--safe-area-inset-top`    | 顶部安全区域（通常是状态栏高度/刘海高度）                          |
| `--safe-area-inset-bottom` | 底部安全区域（无键盘时为 Home Indicator 区域，有键盘时为键盘高度） |
| `--keyboard-height`        | 键盘当前高度                                                       |
| `--keyboard-visible`       | 键盘可见性（`1` 表示可见，`0` 表示隐藏）                           |
| `--content-bottom-padding` | 推荐的底部填充值，已计算好 Safe Area 和 Keyboard 的补偿            |

### CSS 使用示例：

#### A. 处理顶部状态栏（沉浸式导航栏）

```css
.navbar {
  padding-top: var(--safe-area-inset-top);
  background-color: #ffffff;
}
```

#### B. 处理底部输入框（防止被键盘遮挡）

```css
.footer-input-bar {
  /* 自动适配 Home Indicator 和 键盘高度 */
  padding-bottom: var(--safe-area-bottom-computed);
}
```

#### C. 根据键盘状态切换布局

```css
.bottom-ad-banner {
  /* 当键盘弹出时，通过 opacity 或 display 隐藏元素 */
  opacity: calc(1 - var(--keyboard-visible));
  pointer-events: var(--keyboard-visible) == 1 ? none : auto;
}
```

---

## 总结

该插件通过 iOS 原生代码强力干预了 WebView 的渲染布局逻辑，将控制权从系统交还给了前端开发者。通过监听 `safeAreaChanged` 事件和引用相关的 CSS 变量，前端开发者可以极其精确地控制页面在刘海屏、灵动岛以及键盘弹出时的表现，从而打造出原生级别的移动端体验。
