# PR 描述：修复主题浅拷贝污染、生命周期泄漏及页码越界问题

## Issue 标题

`[Bug] 自定义主题污染其他主题 + 多处内存泄漏`

## Issue 内容

在使用过程中发现以下几个 bug，并已定位根因和完成修复，附上 PR。

---

### Bug 1（严重）：新建/编辑自定义主题会污染其他主题

**现象：**
- 新建自定义主题时选择参考模板，修改新主题的颜色后，原参考模板的颜色也跟着变了
- 编辑已有自定义主题时，点击"取消"后改动仍然生效

**根因：**
JavaScript 的展开运算符 `{ ...obj }` 只做浅拷贝，`styles` 是嵌套对象，
多个主题会共享同一个 `styles` 对象的内存引用，修改一个就会影响所有引用它的主题。

**涉及代码：**
- `src/settings/CreateThemeModal.ts` 第 19 行：`{ ...existingTheme }` 浅拷贝
- `src/settings/CreateThemeModal.ts` 第 129 行：`{ ...selectedTheme, ... }` 浅拷贝
- `src/settings/settings.ts` 第 138 行：`addCustomTheme` 直接 push 原始引用
- `src/settings/settings.ts` 第 160 行：`updateTheme` 浅合并

---

### Bug 2（严重）：视图和插件缺少生命周期清理

**现象：**
- 长期使用后 Obsidian 内存持续增长
- 禁用再启用插件后，旧实例的事件监听器仍在运行

**根因：**
- `RedPlugin` 没有 `onunload()` 方法，插件禁用时资源不释放
- `RedView` 没有 `onClose()` 方法，关闭视图时 500ms 定时器不停止
- `createCustomSelect` 每次调用都往 `document` 上追加匿名 `click` 监听器，永不移除

---

### Bug 3（中）：编辑内容后页码显示异常

**现象：**
用户在浏览第 5 页时，若编辑笔记使页面总数减少到 3 页，
页码指示器显示 `5/3`，当前页面空白。

**根因：**
`onFileModify` 触发 `updatePreview()` 时未重置 `currentImageIndex`，
`updateNavigationState()` 也未对越界索引做保护。

---

## PR 标题

`fix: 修复主题浅拷贝污染、生命周期泄漏及页码越界问题`

## PR 正文

### 修复内容

#### 1. 主题深拷贝（修复 Bug 1）

将所有涉及主题对象复制的地方改为 `JSON.parse(JSON.stringify())` 深拷贝，
确保不同主题之间的 `styles` 对象完全隔离，互不影响。

**修改文件：**
- `src/settings/CreateThemeModal.ts`：构造函数和参考模板选择时使用深拷贝
- `src/settings/settings.ts`：`addCustomTheme` 和 `updateTheme` 使用深拷贝

#### 2. 生命周期清理（修复 Bug 2）

- `src/main.ts`：新增 `onunload()`，插件禁用时分离视图并移除事件监听
- `src/view.ts`：新增 `onClose()`，视图关闭时清除防抖定时器
- `src/view.ts`：将 `document.addEventListener` 改为 `this.registerDomEvent()`，
  由 Obsidian 框架自动管理生命周期

#### 3. 页码边界保护（修复 Bug 3）

- `src/view.ts`：`updateNavigationState()` 中增加：
  - 页数为 0 时显示 `0/0` 并隐藏导航按钮
  - 页码索引自动收敛到有效范围（`Math.min(currentIndex, length - 1)`）

### 测试

- [x] 新建自定义主题，修改颜色，验证原主题不受影响
- [x] 选择参考模板后修改样式，验证保存生效
- [x] 多页文档减少标题后，验证页码自动修正

## 提交 PR 步骤

1. 打开 https://github.com/zhangjx0101/note-to-red
2. 点击 **"Contribute" → "Open pull request"**
3. Base repository 选 `Yeban8090/note-to-red`，base 选 `main`
4. 标题填上方的 PR 标题
5. 正文粘贴上方的 PR 正文
6. 点击 **"Create pull request"**
