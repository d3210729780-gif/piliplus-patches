# Issue 1 · 功能请求（FR）

> 提交方式：仓库 → Issues → New issue → 选 **功能请求** → 标题贴下面的 Title，
> 然后把「功能描述 / 解决方案 / 其他」三段分别贴进对应输入框（检查清单那三个勾自己点一下）。

## Title

```
[FR] 已下载的视频支持「听视频」与离线合集连播
```

## 功能描述

现在「听视频」（`AudioPage`）只对**在线视频**开放，已下载／离线缓存的视频没有入口：

- `lib/pages/video/view.dart` 的「更多」菜单里写死了 `if (!isFileSource && isUgc)`，本地视频直接不显示「听音频」
- `lib/pages/video/widgets/header_control.dart` 顶栏那个耳机按钮整块包在 `if (!isFileSource)` 里
- 即使本地播放页有「听视频」开关（`header_control.dart` 里把 mpv 的 `vid` 设为 `no`），那只是关掉视频轨，没有 `AudioPage` 的封面/简介/播放列表界面

而 `AudioPage` 本身完全依赖 gRPC `bilibili.app.listener.v1` 的 `audioPlayList` / `audioPlayUrl` 取列表和地址，离线时这两样都没有。

**期望行为**

1. 已下载的视频（含分离音轨的 m4s、以及合并的 mp4）可以进「听视频」页面，界面与在线版一致（封面、标题、UP、进度、倍速、定时关闭、后台播放）
2. **离线连播**：同一 B 站合集的所有已下载集数、或同一视频的所有已下载分P，自动组成播放列表，支持上一首/下一首/播完自动下一首
3. 本地播放列表支持**播放顺序**：无序 / 正序 / 倒序 / 随机；「随机 + 列表循环」时，每循环一圈重新打乱
4. 离线缓存页支持**自建文件夹**（新建 / 重命名 / 删除 / 把条目移入移出），并且文件夹要**等价于合集**：听视频列表、播放页的选集、上下集/连播都限定在文件夹内
5. 文件夹只做逻辑分组，不动磁盘上的缓存目录（挪文件会让下载记录里的路径失效）

**使用场景**：通勤/睡前只听音频、把下载的 MV 当歌单听、流媒体流量受限时。

## 解决方案

已经在本地完整实现并验证过，思路如下（改动全部在 Dart 层，不涉及原生）：

1. **新增 `lib/pages/audio/models/local_audio_item.dart`**
   - 由 `BiliDownloadEntryInfo` 构造本地播放源：`FileSource` 有分离音轨就取 `.audio.m4s`，否则直接用视频文件（media_kit 两者都能播）
   - `toDetailItem()` 合成一个最小的 protobuf `DetailItem`，让 `AudioPage` 原有的封面/标题/UP/歌单 UI 不用改
   - `playlistOf()` 决定列表分组优先级：**自建文件夹 → 同 seasonInfo 的合集 → 同一 avid 的分P**
2. **`AudioController` 加本地模式**
   - `localItems` 非空即离线模式：跳过 `_queryPlayList` / `_queryPlayUrl` / SponsorBlock
   - `_onOpenLocalMedia()` 直接 `player.open(Media(本地路径))`，不需要 UA/Referer
   - `playPrev/playNext/playIndex/loadPrev/loadNext/onChangeOrder` 全部走本地列表
   - 列表循环到底回到第一首；随机模式下回绕时重新 shuffle（以「正序」为种子）
   - 界面隐藏点赞/投币/收藏/评论等必然联网的操作，信息行改成「本地缓存 · 离线播放」
3. **入口**三处：本地播放页顶栏耳机、本地播放页「更多」菜单、下载列表条目「⋮」菜单
4. **自建文件夹**
   - 新增 `lib/services/download/download_folder_store.dart`：文件夹列表 + `cid → folderId` 归属，存在 `localCache` box 的普通 List/Map 里，不加 Hive adapter、不做数据迁移
   - `DownloadController._applyFolders()`：把归入文件夹的条目从原分组摘出来，为每个文件夹造一个 `DownloadSeasonInfo(isFolder: true)` 插到最前；`SeasonInfo.id` 用文件夹 id，点进去直接复用原有的合集详情页
   - 删除文件夹不删视频，条目回到原分组
5. **播放侧一致**：`LocalIntroController.onInit()`（本地播放页的选集列表）也按 `folderOf(cid)` 聚合，保证上下集/自动连播只在本文件夹内循环

> 注意：视频页的自动连播跟合集一样受播放器「播放模式」控制（默认 `pause`），需要在播放器里选「顺序播放 / 列表循环」；听视频页默认就是顺序播放。

## 其他

- 已在本机实现并实机验证：Flutter **3.47.7**、Android arm64 release 包（MuMu 模拟器），涉及 `lib/pages/audio/`、`lib/pages/download/`、`lib/pages/video/`、`lib/services/download/` 等 16 个文件、约 950 行（不含第三方包改动）
- 已在本地实现并实机验证；实现思路与改动文件都已列全，可照此实现（`lib/` 改动不依赖任何第三方包变更）
- 已知限制：离线模式下不提供点赞/投币/收藏/评论（离线场景保证不报错）；文件夹是纯本地数据，不参与 WebDAV 备份同步
- 截图：<!-- 建议贴：1) 本地视频进听视频页 2) 离线缓存里的文件夹卡片 3) 文件夹内的播放列表 -->

---

### 附：改动文件清单（实现参考）

**新增 3 个**

| 文件 | 作用 |
|---|---|
| `lib/pages/audio/models/local_audio_item.dart` | 下载条目 → 本地播放源；`toDetailItem()` 合成最小 `DetailItem` 复用音频页 UI；`playlistOf()` 分组（文件夹 → 合集 → 分P） |
| `lib/pages/download/folder/folder_dialogs.dart` | 新建/重命名/删除文件夹、移动条目到文件夹 |
| `lib/services/download/download_folder_store.dart` | 文件夹列表 + `cid → folderId`，存 `localCache` box，不加 Hive adapter |

**修改 13 个**

| 文件 | 改了什么 |
|---|---|
| `lib/pages/audio/controller.dart` | 本地模式（跳过 gRPC 取列表/取地址）、`_playLocalIndex`、`_reorderLocal`（排序）、列表循环回绕 + 随机重洗 |
| `lib/pages/audio/view.dart` | `toAudioPage` 增加 `localItems`；本地模式隐藏联网操作、信息行换「本地缓存 · 离线播放」 |
| `lib/pages/video/controller.dart` | `toAudioPage()` 增加 file source 分支 |
| `lib/pages/video/view.dart` | 「更多」菜单放开 `isFileSource` 的「听音频」 |
| `lib/pages/video/widgets/header_control.dart` | 本地播放页顶栏加耳机按钮 |
| `lib/pages/video/introduction/local/controller.dart` | 本地播放页选集按文件夹/合集聚合；修 `index == -1` 越界 |
| `lib/models_new/download/bili_download_entry_info.dart` | 条目菜单加「听音频」「移动到文件夹」 |
| `lib/models_new/download/download_info.dart` | `DownloadSeasonInfo` 增加 `isFolder` |
| `lib/pages/download/download/controller.dart` | `_applyFolders()`：条目从原分组摘出，造「伪合集」分组 |
| `lib/pages/download/download/view.dart` | 文件夹卡片；顶栏「新建/管理文件夹」 |
| `lib/pages/download/download/widgets/season.dart` | 角标「合集 / 文件夹」可切换 |
| `lib/pages/download/download_action_mixin.dart` | 多选「文件夹」按钮（批量移动） |
| `lib/pages/download/page/view.dart` | 接入上面的批量移动 |
