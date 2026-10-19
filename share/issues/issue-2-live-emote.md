# Issue 2 · 功能请求（FR）

> 提交方式：仓库 → Issues → New issue → 选 **功能请求** → 标题贴下面的 Title，
> 「功能描述 / 解决方案 / 其他」三段分别贴进对应输入框。
> 如果更想按 bug 提，也可以用 `[Bug]` 标题 + bug 模板，把「功能描述」拆成「预期行为 / 实际行为」。

## Title

```
[FR] 直播弹幕在画面上显示表情图片（目前只显示 [表情名] 文字）
```

## 功能描述

直播间里同一个表情弹幕，两个显示位置的行为不一样：

| 位置 | 表现 |
|---|---|
| 右侧聊天列表 | ✅ 正常显示表情图片（`chat_panel.dart` 的 `_buildMsg()` 同时处理 `uemote` 和 `emots`，用 `WidgetSpan` 画图） |
| **画面上飘的弹幕** | ❌ 只有 `[表情名]` 这段文字 |

原因在 `canvas_danmaku`：它是把整条弹幕排版成 `ui.Paragraph`、再栅格化成一张 `ui.Image` 贴到画布上的（`lib/utils/utils.dart` 的 `generateParagraph` + `recordDanmakuImage`），**只能 `addText`**，没有图片能力。所以 `DanmakuContentItem(msg)` 里的表情占位串就被原样画成了文字。

**期望行为**

1. 直播弹幕里的表情（普通表情 `emots`、大表情 `uemote`）在飘屏弹幕上显示为图片
2. 尺寸要跟字号协调，不能糊满屏幕（官方大表情原图可能 160px 以上）
3. 图片没加载完时不要崩、不要留空白框，可以先退化成文字
4. 没有表情的普通弹幕，行为和性能保持不变

## 解决方案

要改 `canvas_danmaku`（PiliPlus 的 git 依赖，作者自己的 fork），Dart 侧改动如下：

1. `models/danmaku_content_item.dart`
   - 新增 `DanmakuEmote { url, width, height, text }`（`text` 是它在弹幕文本里的占位串）
   - `DanmakuContentItem` 增加可选 `Map<String, DanmakuEmote>? emotes`
2. `utils/utils.dart`
   - `emoteSegments()`：按占位串切分文本（**长 key 优先**匹配），得到「文字 / 表情」片段序列
   - `generateParagraph`：文字 `addText`，表情 `addPlaceholder(width, height, alignment)`
     （注意 Flutter 3.47 起 `addPlaceholder` 是三个位置参数，**`PlaceholderDimensions` 已经被移除**）
   - `recordDanmakuImage`：`paragraph.getBoxesForPlaceholders()` 拿到每个占位矩形，用 `canvas.drawImageRect` 把表情画进去；描边用的 paragraph 用同一套占位结构，保证几何一致
   - 尺寸归一：表情高度夹在 `1.2 ~ 2.4 × fontSize`，按原图宽高比缩放
   - 新增 `preloadEmotes(emotes, {headers})`：用 `HttpClient` 预解码成 `ui.Image` 做 LRU 缓存（上限 240 张）；没解出来的那一帧退化画占位文字
3. PiliPlus 侧 `lib/pages/live_room/controller.dart`
   - `DANMU_MSG` 分支把 `extra['emots']` 和 `first[13]` 的 `uemote` 转成 `DanmakuEmote` 挂到 `DanmakuContentItem.emotes` 上
   - 顺手触发预加载（带 `Referer: https://live.bilibili.com/`）
4. `pubspec.yaml` 用 `dependency_overrides` 把 `canvas_danmaku` 指向本地 `third_party/canvas_danmaku`（或者直接把改动 PR 到包的仓库再改 ref）

## 其他

- 已在 Flutter **3.47.7** 上实现并实机验证（Android arm64），表情在飘屏正常显示，普通弹幕不受影响
- 已知限制：画布是**同步**栅格化的、图片解码是**异步**的，所以某一张表情**第一次**出现时可能还是文字，同一张再次出现就是图了。要绝对准确需要把弹幕入队改成「预加载完成后再入队」，代价是几百毫秒延迟
- 相关：`#2981`（直播间自己弹幕只有文字形式展示）应该和这个是同一个根因
- 如果包仓库不方便接收改动，也可以由 PiliPlus 侧自己实现一个带图片能力的弹幕 painter，但工作量会大不少
- 截图：<!-- 建议贴：1) 聊天列表里的表情 2) 同一时刻飘屏弹幕上的 [表情名] 文字 -->

---

### 附：改动文件清单（实现参考）

**第三方包（真正要改的只有 2 个文件）**

| 文件 | 改动 |
|---|---|
| `canvas_danmaku/lib/models/danmaku_content_item.dart` | 新增 `DanmakuEmote { url, width, height, text }`；`DanmakuContentItem` 增加 `Map<String, DanmakuEmote>? emotes` |
| `canvas_danmaku/lib/utils/utils.dart` | `emoteSegments()` 切分文本；`generateParagraph` 用 `addPlaceholder`；`recordDanmakuImage` 用 `getBoxesForPlaceholders() + drawImageRect`；`preloadEmotes()` 预解码 + LRU |

其余 `third_party/canvas_danmaku/**` 都是原样落地，未改动。

**PiliPlus 侧**

| 文件 | 改动 |
|---|---|
| `pubspec.yaml` / `pubspec.lock` | `dependency_overrides` 把 `canvas_danmaku` 指向本地包 |
| `lib/pages/live_room/controller.dart` | `DANMU_MSG` 分支把 `emots` / `uemote` 转成 `DanmakuEmote`，挂到 `DanmakuContentItem.emotes`，并触发预加载 |
