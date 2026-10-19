# Issue 3 · 功能请求（FR）

> 提交方式：仓库 → Issues → New issue → 选 **功能请求** → 标题贴下面的 Title，
> 「功能描述 / 解决方案 / 其他」三段分别贴进对应输入框。

## Title

```
[FR] 收藏夹支持多选批量下载（含分P、可选画质）
```

## 功能描述

现在要从收藏夹下载视频，只能**一个一个点进视频详情页**，再从详情页菜单里点「缓存视频」，
收藏夹里几十个视频操作成本非常高。

**期望行为**

1. 收藏夹详情页长按进入多选后，多选栏提供「下载」入口
2. 选中多个视频后可以一次性全部加入离线缓存队列
3. **多P视频要把它所有分P一起下载**，而不是只下 P1
4. 批量下载前可以选择画质（默认值跟视频页下载面板一致）
5. 结果要有反馈：成功加入多少个、多少个分P、有多少因为已在列表里被跳过、有多少解析失败
6. 选中的条目里混有音频/番剧时不要整体失败，跳过并说明

## 解决方案

改动很小，全部在 `lib/pages/fav_detail/`：

1. 新增 `lib/pages/fav_detail/batch_download.dart`（约 120 行）
   - `batchDownloadFavVideos(context, items)`：过滤出 `type == 2`（普通视频）且带 `bvid` 的条目
   - 弹一个画质选择框（`VideoQuality.values`，标出 `Pref.defaultVideoQa` 是默认值）
   - 逐个 `VideoHttp.videoIntro(bvid: ...)` 拿 `pages`，再对每个分P 调
     `DownloadService.downloadVideo(index:, page:, videoDetail:, videoQuality:)`
   - 过程用 `SmartDialog.showLoading('正在解析 3/8')` 给进度
   - 结束后按队列长度差统计实际新增，一次性 toast 结果
   - 之所以复用 `downloadVideo()`，是为了去重、断点续传、弹幕下载、封面保存这些逻辑都和单集下载一致
2. `lib/pages/fav_detail/view.dart`
   - `_selectActions()` 里加「下载」按钮（这个页面本来就有 `MultiSelectController`，不需要新写多选）
   - 顺带修一个布局问题：多选栏原来有「全选/复制/移动/删除」四个文字按钮，再加「下载」会在窄屏上
     把左侧的「已选 N」挤到重叠 → 把复制/移动/删除收进 `⋮` 溢出菜单，多选栏只留「全选 / 下载 / ⋮」

## 其他

- 已在 Flutter **3.47.7** + Android arm64 上实现并验证，`dart analyze` 零问题
- 已知限制：每个视频都要一次 `videoIntro` 请求，选很多（50+）会花十几秒（有进度提示）；
  画质是批量统一的，个别视频不支持该画质时那一个任务会在下载阶段失败，不影响其它任务
- 截图：<!-- 建议贴：1) 多选栏（已选 N / 全选 / 下载 / ⋮） 2) 画质选择框 3) 完成后的结果 toast -->

---

### 附：改动文件清单（实现参考）

**新增 1 个**

| 文件 | 作用 |
|---|---|
| `lib/pages/fav_detail/batch_download.dart` | 批量下载主逻辑：过滤 `type == 2` → 选画质 → 逐个 `VideoHttp.videoIntro` → 每个分P 调 `DownloadService.downloadVideo` → 统计并 toast |

**修改 1 个**

| 文件 | 改了什么 |
|---|---|
| `lib/pages/fav_detail/view.dart` | `_selectActions()` 加「下载」按钮；复制/移动/删除收进 `⋮` 溢出菜单（修「已选 N」重叠） |
