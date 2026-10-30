---
name: "onboarding-guide-tour"
description: "在 React + antd 前端项目中设计并实现新用户指引（首次登录自动弹出、遮罩高亮、边填边引导、跟随真实页面推进、重看入口）。当用户提到新用户指引、新手引导、首次使用引导、产品导览、onboarding、Tour、操作指引时使用。"
---

# 新用户指引（边填边引导）

在已有的 React + antd 5 项目里加首次使用指引。核心原则：**指引是叠加层，不是新功能**。只加遮罩、卡片和定位属性，不新增页面、接口、后端字段，也不改业务校验。用户跟着指引操作的是真实页面，指引走完，用户真的完成了任务。

## 1. 先确认需求（动手前）

用 AskUserQuestion 问清这几项，每项给推荐默认值：

- 指引覆盖哪些阶段、什么顺序。按真实依赖排序，例如「必须先有已发布的 X 才能绑定 X」。
- 谁会自动弹出：所有首登用户，还是特定角色。
- 要不要重看入口，放在哪里（顶部栏、页脚、帮助菜单）。窄屏下是否只显示图标。
- 前置条件不满足时怎么处理（如没有可绑定的资源）：按情况分支、调整阶段顺序，还是先跳过。
- 推进方式：用户操作真实按钮即推进（推荐），还是只用卡片按钮推进。

如果要动界面布局（如在顶部栏加入口），先出一张 HTML 示意图让用户确认，再改代码。

先写一份计划文档（如 `docs/plans/新用户指引计划.md`）：列步骤表（阶段、高亮元素、说明、推进条件）、改动文件清单、首登与重看规则、验证方式、待确认项。用户确认后再实现。

## 2. 关键设计决策

**不要做「演示弹窗」。** 一种直觉做法是指引自己打开一个只读的演示弹窗，切到对应步骤来讲解。用户反馈这样割裂：看完指引还要回头从头操作一遍。正确做法是让指引跟随真实页面：

- 页面级步骤（欢迎、指向「创建」按钮）加遮罩，`mask: true`。
- 弹窗或编辑页内的步骤不加遮罩（`mask: false`），用户正常输入，卡片贴在当前区域旁边说明。
- 卡片的主按钮直接 `click()` 真实按钮，页面原有校验照常生效。
- 指引按固定间隔检查真实 DOM，满足条件就自动跳到对应步骤。这样用户直接点页面上的按钮（下一步、上一步、关闭）也能跟上。
- 跳过或关闭指引只结束引导，用户已填写的内容原样保留。

**前置资源缺失要分支。** 例如绑定 Flow 时没有已发布的 Flow：引导用户点现有的「前往 X 中心」入口（要确认项目里已有草稿保留机制），发布后再点「返回」回到原表单继续。不要为此新增保存逻辑。

**阶段计数比步骤计数友好。** 步骤会分支、回退，卡片显示 `2 / 4 · 配置仓库` 这种阶段进度，而不是 `5 / 14`。

## 3. 数据模型（`src/onboarding.ts`）

```ts
import type { Page } from './routes'; // 项目已有的页面类型

export type GuideEvent = 'projectSaved' | 'flowSaved';           // 业务成功回调发出的事件
// 条件全部满足才匹配：visible/hidden 判断真实元素是否渲染，event 判断本步开始后是否发生过该事件
export type GuideRule = { visible?: string; hidden?: string; event?: GuideEvent; to: string };
export type OnboardingStep = {
  key: string;
  stage?: number;          // ONBOARDING_STAGES 下标，用于阶段计数
  title: string;
  description: string;
  targets: string[];       // 依次尝试，取第一个可见元素（兼容桌面/移动端两套 DOM）
  page?: Page;             // 进入本步时切到的现有菜单
  mask?: boolean;          // 弹窗内步骤设为 false
  noFlow?: string;         // 前置资源缺失时替换成的分支步骤
  advance?: GuideRule[];   // 自动推进规则，按顺序取第一个匹配
  nextClick?: string;      // 卡片主按钮要点击的真实控件
  next?: string;           // 没有 nextClick 时主按钮跳到的步骤；两者都没有则结束
  nextLabel?: string;
  back?: string;           // 只给不需要撤销真实改动的步骤提供「上一步」
  pauseWhile?: string;     // 该元素可见时隐藏卡片（如嵌套的选择器弹窗）
};
```

常用规则写法：

- 弹窗内每一步都带两条公共规则：「关闭弹窗回到创建步骤」，以及「点了跳转入口就到对应页面步骤」。
- `hidden` 写成选择器列表（`A, B`），表示「都不可见」才算关闭。例如弹窗被暂时隐藏去别的页面、同时顶部有「返回草稿」横幅，这种情况不算关闭。
- 用户点了表单自己的「下一步」「上一步」，靠 `visible` 某个只在对应向导步骤出现的区域来同步指引。

完成状态存 `localStorage`，键为 `app.onboarding.<版本>:<账号>`：

- 完成、跳过、关闭都记为看过。手动重看不清除状态。
- 读写都包 try/catch。存储不可用时用模块级 `Set` 兜底，保证同一页面会话只弹一次。
- 指引逻辑有大改时升版本号，看过旧版的用户会再弹一次。升之前先和用户确认。
- 导出 `onboardingStorageKey`、`shouldAutoStartOnboarding(account, storage?)`、`markOnboardingDone(account, storage?)`，storage 参数可注入，方便测试。

## 4. 组件（`src/OnboardingTour.tsx`）

基于 antd `Tour`，每次只渲染当前这一步（`steps` 只有一项，`current={0}`），步骤切换由组件自己管理：

```tsx
const STABLE_FRAMES = 3, MAX_WAIT_MS = 1000, TICK_MS = 200;

function findTarget(selectors: string[]): HTMLElement | null {
  for (const s of selectors) {
    const el = [...document.querySelectorAll<HTMLElement>(s)].find(e => e.getClientRects().length > 0);
    if (el) return el;
  }
  return null;
}
```

- `go(key)`：前置资源缺失时先替换成 `noFlow` 分支步骤。有 `page` 就调用 App 现有的菜单导航；记录当前事件计数作为基线。然后逐帧检测目标位置，连续 3 帧不变、或等满 1 秒才 `setCurrent`，避免指向弹窗动画中途的位置。用递增的 transition id 取消过期的切换。
- 轮询（`setInterval` 200ms）做三件事：
  1. 检查 `advance` 规则，匹配到就 `go(rule.to)`。
  2. 按 `pauseWhile` 决定是否隐藏卡片。
  3. 目标位置变化时 `window.dispatchEvent(new Event('resize'))`。antd Tour 只在 resize 和 target/open 变化时重新定位。
- 把事件、导航函数等最新 props 放进 `useRef`，让定时器读到最新值，不用频繁重建。
- `Tour` 属性：
  - `zIndex={1100}`，高于 antd Modal 默认的 1000。
  - `rootClassName` 区分有遮罩和无遮罩两种样式，无遮罩时卡片要加明显的阴影和边框。
  - `scrollIntoViewOptions={{ block: 'center' }}`。
  - `indicatorsRender` 显示阶段计数。
  - `actionsRender` 自定义：「跳过」（最后一步不显示）、「上一步」（仅有 `back` 时显示）、主按钮。
- 不要同时传 `onFinish` 和 `onClose` 去记录完成状态。Tour 结束时会调 onClose，再传 onFinish 会记录两次。

## 5. 接入 App

- 状态：`guideOpen`，以及事件计数器 `{ projectSaved: 0, flowSaved: 0 }`。在业务保存成功的回调里 `+1`。只加这一行，不改原逻辑。
- 自动弹出：数据加载完成、已登录、没有认证错误、`shouldAutoStartOnboarding(account)` 为真，并且当前没有打开用户自己的弹窗或未保存的编辑页，才弹出。
- 重看入口：
  - 用户自己的弹窗打开时置灰，避免指引盖住未保存的输入。
  - 入口本身加 `data-guide`，最后一步指向它，告诉用户以后从哪里重看。
- 在现有控件上加 `data-guide="xxx"` 定位属性。
  - 不要依赖 antd 内部类名，除非没有别的办法（如 Modal 底部按钮）。
  - 同一个 `data-guide` 名在页面上只能对应一处。列表行里的旧标记要删掉，否则会匹配到错误的元素。
- 可选：给 `Tour` 外层加 CSS，控制卡片宽度（如 360px，`max-width: calc(100vw - 32px)`）、说明文字行高，以及窄屏下入口只显示图标。

## 6. 测试

用项目现有的测试框架。没有的话用 `node --test` 配合 esbuild 打包 `onboarding.ts`。覆盖这几类：

- 步骤顺序与阶段单调递增，每步至少一个 target。
- 所有 `next` / `back` / `noFlow` / `advance[].to` 都指向存在的步骤（防止拼错 key）。
- 弹窗内步骤都是 `mask: false`；关闭弹窗、向导回退、保存成功事件的推进规则都存在。
- 分支路径完整：缺资源 → 前往创建 → 保存 → 返回 → 回到绑定步骤。
- 存储：首登弹出、完成后不再弹、按账号隔离、未登录不弹、存储抛错时仍只弹一次、旧版本键不影响新版本。
- 源码检查：每个 `data-guide` 选择器都能在源码中找到对应标记；重看入口在顶部栏指定位置；App 里没有残留的演示弹窗代码。
- 浏览器端到端测试会被首登自动弹出的指引挡住。在 fixture 里用 `Page.addScriptToEvaluateOnNewDocument` 预先写入 `localStorage.setItem('<当前版本键>:<测试账号>', 'done')`，升版本号时同步改这里。

## 7. 常见坑

- 删掉 `data-guide` 后，`.map((item, row) =>` 里的 `row` 变成未使用参数，开了 `noUnusedParameters` 会报 TS6133。
- 分段写大文件时，最后一次整体覆盖容易丢掉文件尾部的函数。写完要检查导出是否齐全。
- 无遮罩步骤的卡片可能挡住输入框。手测时分别用「资源齐全」和「资源缺失」两种账号走一遍。
- 指引期间不要发写请求。写入只能由用户（或卡片代用户）点击真实按钮触发。
- 交付前运行 `typecheck && build && test`。环境里跑不了的话，明确告诉用户哪些没验证，请他在本地运行。
