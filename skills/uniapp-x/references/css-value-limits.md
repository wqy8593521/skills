---
name: css-value-limits
description: uvue 样式的值域限制、css reset 默认值、以及会触发编译告警的写法
---

# 样式值域与 css reset

uvue 的 CSS 是 Web 子集，**属性名存在 ≠ 值合法**。写错值不会静默降级，而是产出编译告警（`uni:app-uvue-css`，plugin 为 `@dcloudio/uni-nvue-styler`），且该声明被丢弃。Agent 应优先按值域写，而不是先写完再等告警。

## 值域（App 端实际接受的值）

| 属性 | 接受值 | 常见错写（编译报 ERROR） |
|------|--------|--------------------------|
| `width` / `height` | `number` \| `pixel` \| `percent` \| `auto` | `max-content`、`fit-content`、`min-content` |
| `min-width` / `max-width` / `min-height` / `max-height` | `number` \| `pixel` | `100%`、`auto`、`max-content`、`fit-content` |
| `overflow` | `hidden`（App 端实际生效值） | `visible`（仅 view 组件支持，scroll-view 明确不支持） |
| `display` | `flex` \| `none` | `block`、`inline`、`inline-flex` |

注意：文档里 `min-width` 的属性值表列了 `auto` / `fit-content` / `max-content`，但兼容表对 App 端全部是 `x`——**这些值只在 Web 生效**，写进 uvue 一律告警。`min-width` 与 `width` 的值域不一致，这是最易踩的一条。

`width` 的 `max-content` 同样只属于 Web。App 端要让盒子按内容撑开，用 flex 手段（见下），不要靠 `max-content`。

## css reset（与 W3C 默认值不同，必须知道）

| 属性 | uvue-app | W3C | 影响 |
|------|----------|-----|------|
| `flex-direction` | `column` | `row` | 横排必须显式写 `flex-direction: row` |
| `flex-shrink` | `0` | `1` | **App 端子项默认不压缩**，显式写 `flex-shrink: 0` 是冗余（H5 端才需要） |
| `overflow` | `hidden` | `visible` | 溢出内容默认被裁掉，这是"角标/徽标被切一半"的根因 |
| `position` | `relative` | `static` | 绝对定位的参照系默认已建立 |
| `display` | `flex` | `inline` | 容器天然是 flex |
| `box-sizing` | `border-box` | `content-box` | 宽高含 padding/border |
| `min-width` / `min-height` | `0px` | `auto` | flex 子项可被压缩到 0 |
| `white-space` | `keep` | `normal` / `pre-line` | App 端默认保留连续空白 |

`overflow: hidden` 是 uvue 的默认值，所以**任何 absolute 定位、需要溢出父盒的装饰元素（徽标、角标、气泡、tooltip）都会被父层裁掉**。修法是把直接父层的 `overflow` 显式设为 `visible`（仅 view 组件支持），或改用不越界的布局。同理，父层若是 `scroll-view`，`overflow` 无法设为 `visible`，装饰元素必须收在滚动内容之内。

## 已废弃写法 → 替代

| 废弃 | 替代 | 出处 |
|------|------|------|
| `lines: <integer>`（CSS） | `<text max-lines="N">`（**组件属性**，不是 CSS） | 蒸汽模式（Vapor）已废弃 `lines`；Web 端需 `-webkit-line-clamp` |
| `scroll-y` / `scroll-x`（布尔属性） | `direction="vertical"` / `"horizontal"` / `"all"` / `"none"` | 4.0 起废弃 |
| `type="nested"`（scroll-view） | 内层设 `associative-container="nested-scroll-view"` | 蒸汽模式已废弃 `type` |
| `space` / `decode`（text 属性） | `white-space` CSS / 运行期自行解码 | 蒸汽模式已废弃 |

`direction` 默认值就是 `vertical`，竖向滚动**不需要写任何属性**；写了 `scroll-y` 反而告警。

`lines` → `max-lines` 的迁移要点：`max-lines` 是 `<text>` 的属性，且必须配合 `text-overflow: ellipsis` 才出现省略号。多行截断（`max-lines: 2`）在 App 端同样走组件属性，不要指望 `-webkit-line-clamp`（uvue 不支持 `display: -webkit-box`）。

## 横滑轨道：不要用 max-content 撑开

横向滚动行容器要"按内容撑开"，否则被压成 100% 宽 → `scrollWidth == clientWidth` → `scrollLeft` 恒为 0。但 `width: max-content` 与 `min-width: 100%` 在 App 端都非法。

跨端唯一可行写法是取消纵向拉伸：

```css
.rail-row {
  align-self: flex-start;   /* 父 scroll-view 是 column，取消 stretch → 宽度即内容宽 */
  flex-direction: row;
}
```

注意 `align-self` 只在父容器 `flex-direction: column` 时作用于交叉轴（宽度）。若父 scroll-view 显式写了 `flex-direction: row`，`align-self` 作用于高度、对宽度无效——此时应改用 `flex-shrink: 0`（App 端本就是 0，H5 端需显式写）。

## 字体图标

App 端**不支持伪元素**，`::before` / `::after` 的 `content` 图标方案不可用。字体图标必须走 **unicode 直显**：`@font-face` 引入字体文件，再把码点作为文本放进 `<text>`：

```vue
<text class="icon">{{ String.fromCharCode(0xe600) }}</text>
```

```css
.icon { font-family: myIcon; font-size: 22px; }
```

自定义图标组件通常暴露 `name` → 码点映射表，渲染时输出 unicode 字符，而不是靠 class + 伪元素。

## 其他常见限制

- `z-index` 仅在**同层兄弟节点**间生效，不能脱离 DOM 树任意调层。
- `sticky` 属性值 App 端不支持，吸顶用 list-view 的 `sticky-header` 或 scroll-view + transform。
- 关系选择器（`.a .b`、`.a > .b`）在蒸汽模式不支持；选择器只支持 class，且 class 名只允许 `A-Z a-z 0-9 _ -`。
- 非 Web 平台不支持 `css scoped`，样式隔离走"样式隔离策略 2.0"。
- `!important` 只能写在 class 里，内联 `style` 属性里无效。
- App 端 `scroll-view` 的 `overflow` 不支持 `visible`。

<!--
Source references:
- docs/css/README.md (ucss 子集、flex 方向、样式不继承、css reset 清单、字体图标注意、z-index、css 方法)
- docs/css/width.md / min-width.md / max-width.md / overflow.md / display.md / lines.md / align-self.md / position.md
- docs/.vuepress/utils/cssJson.json (值限制与兼容表原始数据)
- docs/component/scroll-view.md (direction / scroll-y 废弃说明)
- docs/component/text.md (max-lines 组件属性)
-->
