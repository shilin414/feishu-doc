# 飞书 iOS 原生侧滑返回异常——第二版方案 第一阶段执行报告

执行日期：2026-09-25

## 一、基线与新 Commit

| 项目 | 值 |
|---|---|
| 业务仓库 | https://github.com/shilin414/cas |
| 分支 | `test` |
| 基线 Commit | `cebf8f26bb650f2919b627bc7537942f7006e647` |
| 新 Commit | `1ac05ad0c07405bd99cfebd617e92a5cd069f7e3` |
| Commit 信息 | `fix(frontend): keep mobile history navigation user-activated` |

回滚方式（真机无效时整体恢复）：

```bash
git revert 1ac05ad0c07405bd99cfebd617e92a5cd069f7e3
```

## 二、修改文件（共 7 个）

```
frontend/src/shell/MobileAppShell.tsx
frontend/src/shell/__tests__/MobileAppShell.test.tsx
frontend/src/workbench/shell/MobileWorkbenchDrawer.tsx
frontend/src/workbench/capability/CapabilityPicker.tsx
frontend/src/workbench/capability/CapabilityPickerSheet.tsx
frontend/src/workbench/__tests__/CapabilityPicker.navigation.test.tsx
frontend/src/workbench/__tests__/WorkbenchRouteMapping.test.tsx
```

`RecentTaskList.tsx` 按方案第十八章**未修改**（其 `onNavigate ? onNavigate(path) : navigate(path)` 结构原样保留）。

## 三、删除的 afterOpenChange → navigate 逻辑

### MobileAppShell（原第七章 7.1）

删除：

- `pendingDrawerNavigation` ref（关闭动画结束后才 navigate 的暂存路径）
- `navigateAfterDrawerClose`（点击 → 关 Drawer → 暂存目标路径）
- `handleDrawerAfterOpenChange`（动画结束后从 `afterOpenChange(false)` 回调里执行 `navigate()`）
- Drawer 上的 `afterOpenChange={handleDrawerAfterOpenChange}` prop

### CapabilityPicker（原第七章 7.2）

删除：

- `pendingPath` ref
- `handleAfterOpenChange`（Sheet 关闭动画结束后才 navigate）
- 传给 `CapabilityPickerSheet` 的 `afterOpenChange` prop
- `CapabilityPickerSheet` 组件签名中的 `afterOpenChange` prop 整体移除

## 四、现在哪些 navigate 发生在 click handler 内

### MobileAppShell —— `navigateFromDrawer(targetPath)`

```tsx
const navigateFromDrawer = (targetPath: string) => {
  if (drawerNavigationPendingRef.current) return;
  if (targetPath === `${location.pathname}${location.search}`) {
    setMobileNavOpen(false);      // 同路径：只关 Drawer，不 push 重复条目
    return;
  }
  drawerNavigationPendingRef.current = true;
  navigate(targetPath);          // ← 用户点击调用栈内同步执行
  drawerCloseFrameRef.current = requestAnimationFrame(() => {
    drawerCloseFrameRef.current = null;
    drawerNavigationPendingRef.current = false;
    setMobileNavOpen(false);     // RAF 内只有关闭，绝无 navigate
  });
};
```

覆盖入口（Drawer 内所有导航统一走此函数）：

- 主导航菜单（应用中心 / 智能体中心 / 自动化 / 企业控制台…）—— `MobileWorkbenchDrawer` 的 `go(item.path)`
- 最近使用 —— `go(routeForApplication(item))`
- 最近任务 —— `RecentTaskList` 经 `onNavigate` 直传
- 查看全部任务 —— `go('/tasks')`

另含卸载清理：`useEffect` cleanup 里 `cancelAnimationFrame(drawerCloseFrameRef.current)`，防止 stale RAF。

### CapabilityPicker —— `mobileNavigate(path)`

```tsx
const mobileNavigate = (path: string) => {
  if (navigationPendingRef.current) return;
  navigationPendingRef.current = true;
  navigate(path);                // ← 用户点击调用栈内同步执行
  closeFrameRef.current = requestAnimationFrame(() => {
    closeFrameRef.current = null;
    navigationPendingRef.current = false;
    onCloseRef.current();        // RAF 内只有关闭
  });
};
```

`onClose` 通过 `onCloseRef`（useEffect 同步最新值）在 RAF 回调中调用，不依赖旧闭包。卸载清理同样 `cancelAnimationFrame`（覆盖方案第二十七章：Picker 被 Route Change 直接卸载时取消 RAF 即可，无需播放关闭动画）。

## 五、Overlay 在什么时机关闭

| 组件 | navigate 时机 | Overlay 关闭时机 |
|---|---|---|
| 左侧 Drawer（MobileAppShell） | 用户 click handler 内同步 | 下一帧 `requestAnimationFrame` → `setMobileNavOpen(false)` |
| CapabilityPickerSheet（移动端） | 用户 click handler 内同步 | 下一帧 `requestAnimationFrame` → `onClose()` |
| 桌面端 CapabilityPickerDialog | 不变（`onClose(); navigate(path)`） | 不变（同步 close） |

两处时序形成：`click → history.pushState → route commit → RAF → overlay close transition`，路由切换与遮罩退出动画永不同帧竞争。

## 六、验证结果

| 检查 | 结果 |
|---|---|
| `npm run typecheck` | ✅ 通过 |
| `npm run lint` | ✅ 变更文件 0 error；仓库另有 2 个基线即存在的无关 error（`RunChatPanel.attachments.test.tsx` 的 `fiveMb`、`compressImage.test.ts` 的 `beforeEach` 未使用），经 `git stash` 对照确认与本次修改无关 |
| `npm test` | ✅ 129 个文件 / 1192 个测试全部通过 |
| `npm run build` | ✅ 通过（chunk 大小警告为基线已有） |

### 测试改写要点（原第三十九～四十八节要求）

- `MobileAppShell.test.tsx`：点击后 location **立即**变为 `/apps` 且 Drawer 仍 open → `flushAnimationFrame()` 后 Drawer 才关闭；最近任务入口同样断言先导航后关闭；双击（同帧两个不同菜单项）只产生 1 次导航（经 `location.key` 变化计数审计，突变验证已确认能捕获 guard 失效）；点击当前路由不产生新 History 条目且立即关闭 Drawer。
- `CapabilityPicker.navigation.test.tsx`：断言调用顺序严格为 `['navigate', 'close']`（绝不能反序）；点击调用栈内 sheet 仍 open，RAF flush 后才 close；双击只 navigate 一次；外部 `onSelect` 路径保持同步不受影响。
- `WorkbenchRouteMapping.test.tsx`：prop 更名 `navigateAfterClose` → `onNavigate`，路由映射断言不变。

审查方式：独立子智能体按方案第七十六、七十七条逐项 code review（结论 PASS-WITH-NOTES，无功能缺陷）；其发现的 1 个测试质量问题（MemoryRouter 下 `window.history` 审计恒真空转）已修复并改用 `location.key` 计数；对两处 pending guard 与 navigate/close 顺序做了突变测试（注入错误实现，确认测试确实失败后再恢复）。

## 七、真机未验证的内容（必须按第五十三～六十节验收）

以下尚未在飞书 iOS 真机上验证，需按「进入来源」分 Case 测试：

- **Case A（对照组）**：首页普通卡片同步导航进入 → Native Swipe Back
- **Case B（第一重点）**：左侧 Drawer → 应用中心/智能体中心/自动化 → Swipe Back
- **Case C**：Drawer → 最近使用 → 进入智能体 → Swipe Back
- **Case D**：Drawer → 最近任务 → 进入任务 → Swipe Back
- **Case E（第二重点）**：CapabilityPicker Sheet → 切换智能体 → Swipe Back
- 每个 Case 观察三指标：① 侧滑完成后是否仍空白 2～3 秒；② 返回后立即再次侧滑是否正常；③ 返回后立即点输入框键盘/页面是否立即正常
- 程序化返回按钮与 Native Edge Swipe 分别对照测试

成功标准（第六十一节）：Drawer/CapabilityPicker 进入的页面 Swipe Back 异常明显消失或大幅降低，而其他同步导航保持正常 → 基本确认「延迟创建 History Entry」是核心诱因。若完全没有改善，**停止继续改生产逻辑**，保留或回滚本 Commit 均可，直接进入第二阶段「两页最小 BrowserRouter 复现」（`/__swipe-test/a` ↔ `/__swipe-test/b`，连做 20 次）。

## 八、明确未修改的内容

按方案第三十三、三十四、三十五节，以下全部**没有**改动：

```
VisualViewport / 100dvh（global.css、shell.css）
MobileComposer
WorkspaceHost TTL / ApplicationEntityStore / /resolve
App.tsx focus / session refresh / SSE / AssistantResponse
React Router 类型（仍为 createBrowserRouter，未切 HashRouter）
飞书 showNavBar / showBottomNavBar / slideToClose
MobileActionSheet（第一阶段明确不改）
未新增任何自定义侧滑手势、touch-action / overscroll-behavior CSS
```

## 九、一句话总结

> 用户点击 → 立即在 click 调用栈内创建浏览器 History（保持真实 user activation）→ 下一帧再关闭 Sheet/Drawer → 之后 Native Swipe 使用正常用户导航产生的 History Entry。
