# 本次改动说明（PiliPlus 2.1.6 / main）

已实现的功能（Android 端为主、桌面端顺带可用）：

1. **已下载（离线缓存）的视频可以用「听视频」**，并支持同一合集/分P 已下载条目的离线连播、文件夹聚合
2. **离线缓存支持自建文件夹**，且文件夹等价于「合集」（听视频列表、选集、连播都在文件夹内）
3. **直播弹幕能显示表情图片**（之前画面上飘的弹幕只有一个 `[表情名]` 文字）
4. **收藏夹支持多选批量下载**（含分P，可选画质）
5. 直播弹幕发送失败不再静默（csrf 缺失/错误码/是否真的上屏都会提示），并固定设备指纹、增加 App 签名通道

---

## 零、收藏夹多选批量下载

之前只能一个个点进视频详情页、再从菜单里下载。现在：

1. 收藏夹里长按任意视频进多选 → 勾选（可「全选」）
2. 点多选栏的「下载」→ 弹出画质选择（标注了设置里的默认画质）
3. 逐个解析后加入离线缓存队列，结束提示
   `已加入下载队列：5 个视频（共 12 个分P），2 个已在列表中跳过`

实现：新增 `lib/pages/fav_detail/batch_download.dart`（约 120 行）

- 只处理 `type == 2` 的普通视频，音频/番剧条目跳过并在提示里说明
- 每个视频调 `VideoHttp.videoIntro(bvid)` 拿分P，**所有分P一起入队**
- 复用 `DownloadService.downloadVideo()`，去重/断点/弹幕下载/封面都走原逻辑
- 解析时显示「正在解析 3/8」

顺带修了一个布局问题：多选栏原来有「全选/复制/移动/删除」四个文字按钮，加上「下载」之后
在窄屏上会把左侧的「已选 N」挤到重叠；现在把复制/移动/删除收进 `⋮` 溢出菜单。

---

## 一、离线「听视频」

### 原来为什么不行

「听视频」是独立页面 `AudioPage`（`lib/pages/audio/`），播放列表和播放地址全部来自
B 站 gRPC 的 `bilibili.app.listener.v1` 接口（`audioPlayList` / `audioPlayUrl`），
离线时这两样都拿不到；入口还被写死成只对在线视频开放：

- `lib/pages/video/view.dart`：`if (!isFileSource && isUgc)`
- `lib/pages/video/widgets/header_control.dart`：顶部耳机按钮整块包在 `if (!isFileSource)` 里

### 改法

新增 `lib/pages/audio/models/local_audio_item.dart`：

- 把一个下载条目 `BiliDownloadEntryInfo` 转成可播放的本地源：
  有分离音轨（m4s）就放 `.audio.m4s`，没有就放视频文件本身（mp4，media_kit 照样能播）
- `toDetailItem()` 造一个最小的 protobuf `DetailItem`，让音频页原有的封面/标题/UP/歌单 UI 直接复用
- `playlistOf()` 取当前条目所在「下载页」（同一视频的分P、同一合集的各集）里所有已下载完成的条目，按分P顺序组成离线播放列表

`AudioController` 增加「本地模式」：

- `localItems` 非空即本地模式，跳过 `_queryPlayList` / `_queryPlayUrl` / SponsorBlock 请求
- `_onOpenLocalMedia()` 直接 `player.open(Media(localPath))`，不需要 UA/Referer
- `playPrev/playNext/playIndex/loadPrev/loadNext/onChangeOrder` 全部走本地列表

UI 上本地模式：

- 隐藏 点赞/投币/收藏/评论/分享/三连（这些必然联网），保留播放控制、倍速、定时关闭、音量均衡、后台播放
- 顶部信息行把「播放量 · 日期 · BV号」换成「本地缓存 · 离线播放」
- 播放列表弹层仍然可用，可以点选切换同一个合集里的其它分P
- 排序按钮保留：本地模式下 无序/正序/倒序/随机 都在客户端重排，不请求服务端

播放列表的分组规则跟离线缓存列表保持一致（这里踩过一次坑，见下面的修复记录）：

- 下载时带了 `seasonInfo`（合集）→ 整个合集的所有已下载集数
- 普通多P视频 → 同一 avid 的所有已下载分P
- 合集内按 `seasonInfo.index`（集数）升序，同序号再按下载时间兜底

入口加了三处：

| 位置 | 文件 |
|---|---|
| 本地播放页顶栏耳机按钮 | `lib/pages/video/widgets/header_control.dart` |
| 本地播放页「更多」菜单 → 听音频 | `lib/pages/video/view.dart` |
| 下载列表条目「⋮」菜单 → 听音频 | `lib/models_new/download/bili_download_entry_info.dart` |

---

## 二、直播弹幕表情图片

### 原来为什么不行

分两个显示面：

- **右侧聊天列表**：本来就能显示表情（`lib/pages/live_room/widgets/chat_panel.dart` 的 `_buildMsg()`
  同时处理 `uemote` 和 `emots`，用 `WidgetSpan` 画图）
- **画面上飘的弹幕**：`canvas_danmaku` 是把文字排版成 `ui.Paragraph`、再栅格化成一张
  `ui.Image` 贴到画布上的（`lib/utils/utils.dart` 的 `generateParagraph` + `recordDanmakuImage`），
  只能 `addText`，没有图片能力 → 表情弹幕就只剩 `[表情名]` 这段文字

### 改法

把 `canvas_danmaku` 从 git 依赖改成仓库内 `third_party/canvas_danmaku`（`pubspec.yaml` 里加
`dependency_overrides` 指过去），在包里加表情渲染：

- `models/danmaku_content_item.dart`：新增 `DanmakuEmote`（url / width / height / 占位文本），
  `DanmakuContentItem` 增加可选 `emotes` 映射
- `utils/utils.dart`：
  - `emoteSegments()` 按表情占位串切分弹幕文本（长 key 优先匹配）
  - `generateParagraph` 用 `ParagraphBuilder.addPlaceholder(width, height, alignment)` 给表情预留位置
    （Flutter 3.47 的 API 是三个位置参数，没有 `PlaceholderDimensions` 了）
  - `recordDanmakuImage` 用 `getBoxesForPlaceholders()` 拿到每个占位矩形，`drawImageRect` 把表情画进去
  - 字号归一：表情高度夹在 `1.2~2.4 × fontSize`，避免官方大表情糊满屏幕
  - `preloadEmotes()` 用 `HttpClient` 提前把图解码成 `ui.Image` 做 LRU 缓存（上限 240 张）；
    图还没到的那一帧退化成画占位文字，不会崩

`lib/pages/live_room/controller.dart`：

- `DANMU_MSG` 分支把 `extra['emots']` 和 `first[13]` 的 `uemote` 一起转成 `DanmakuEmote` 挂到弹幕上，
  并顺手触发预加载（带 `Referer: live.bilibili.com`）

---

## 三、构建环境（这次踩到的坑，按顺序）

1. Flutter 必须是 **3.47.7**（`pubspec.yaml` 里 `flutter: 3.47.7` 是硬约束），`lib/scripts/patch.ps1`
   会给 Flutter SDK 自己的 material/cupertino 源码打 30 多个补丁，所以不能随便换版本
2. `patch.ps1` 后半段还会给 pub 缓存里的 **`material_ui` / `cupertino_ui` 包**打补丁，它按
   `~/.pub-cache/hosted/pub.dev` 硬编码查找，所以 pub 缓存必须能被这个路径看到（用的是 junction）
3. Windows 底下 `flutter pub get` 会因为插件需要符号链接而报
   “Building with plugins requires symlink support”，要么开开发者模式，要么构建前把
   `windows/` `linux/` `macos/` 目录挪走（本次是后者，出 APK 后已还原）
4. `flutter_inappwebview` 的 Java 文件路径超过 260 字符，pub 缓存必须在短路径下
   （本次用 `D:\work\dsh\pc`），否则 clone 缺文件、编译报「找不到符号」
5. `compileSdk = 37` 但 Google 仓库只有 `platforms;android-37.0`（没有 `android-37`），
   需要给 SDK 目录建一个 `android-37 -> android-37.0` 的 junction
6. 网络：`github.com` 直连不通，需要走系统代理；`git` 要用 `http.proxy` 配置，
   Gradle 要用 `systemProp.https.proxyHost`，dart/flutter 的下载走国内镜像
   （`PUB_HOSTED_URL=https://pub.flutter-io.cn`、`FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn`）

### 复现构建

```powershell
$env:JAVA_HOME='...\jdk21'          # JDK 17+
$env:ANDROID_HOME='...\android-sdk' # platform-tools/platform 36+37.0/build-tools 37/NDK 28.2.13676358
$env:FLUTTER_ROOT='...\flutter'     # 3.47.7
$env:GITHUB_WORKSPACE='<repo>'
$env:GITHUB_ENV='<随便一个可写文件>'
[Environment]::SetEnvironmentVariable('PUB_CACHE','<短路径>', 'Process')

flutter pub get
powershell -ExecutionPolicy Bypass -File lib/scripts/patch.ps1 android
powershell -ExecutionPolicy Bypass -File lib/scripts/build.ps1 android
flutter build apk --release --target-platform android-arm64 --dart-define-from-file=pili_release.json
```

---

## 四、修复记录（v2）

第一版装到 MuMu 上实测反馈了两个问题，都已修：

1. **「只出现了一首歌」** —— 原来按 `pageId` 聚合，而 `pageId = seasonId ?? avid`，同一合集里
   每集 avid 不同 → 每集各自成一个「页」→ 只剩当前那一首。
   现在按 `seasonInfo`（合集）分组，并且用 `seasonInfo.index` 排序，顺序就是合集里的集数顺序。
2. **少了播放顺序（无序/正序/倒序/随机）** —— 第一版把排序按钮在本地模式隐藏了（原来它是服务端
   排序）。现在本地模式也能排序，在客户端重排 `localItems` + `playlist`，不打断当前播放，
   当前曲目在重排后会被重新定位。

---

## 五、新增：离线缓存自建文件夹（v3）

### 怎么用

- **新建**：离线缓存页右上角 →📁 图标 →「新建文件夹」
- **归入**：任意条目「⋮」→「移动到文件夹」→ 选文件夹 / 新建 / 移出；
  也可以长按进多选，点顶部「文件夹」批量移动
- **管理**：📁 →「管理文件夹」→ 重命名 / 删除
- 文件夹在列表里以卡片呈现（角标显示「文件夹」而不是「合集」），点进去是跟合集页一样的列表
- 删除文件夹**不会删视频**，里面的条目会回到原来的分组
- **文件夹等价于合集**：播放/连播都限定在同一个文件夹里（v4 起）
  - 听视频的播放列表 = 整个文件夹
  - 本地播放页的「选集」列表 = 整个文件夹，上下集/自动连播都只在文件夹内循环
  - 不在文件夹里的视频，播放列表不会再混进已归入文件夹的条目

### 实现要点

- `lib/services/download/download_folder_store.dart`：文件夹 + `cid → folderId` 归属，直接存在
  `localCache` box 里的普通 List/Map，不加 Hive adapter、不做数据迁移
- `DownloadController._applyFolders()`：先把归入文件夹的条目从原分组里摘出来，再为每个文件夹造一个
  「伪合集」`DownloadSeasonInfo(isFolder: true)` 插到最前；`SeasonInfo.id` 用文件夹 id，
  所以点进文件夹详情页时按 id 匹配能直接复用原有的 `DownloadPagePage`
- 播放侧一致的聚合规则：`LocalAudioItem.playlistOf()`（听视频）和
  `LocalIntroController.onInit()`（本地播放页选集）都先看 `DownloadFolderStore.folderOf(cid)`
- 只做逻辑分组，**不动磁盘上的缓存目录**（挪文件会让下载记录里的路径失效）
- 文件夹是纯本地数据，不跟 WebDAV 备份同步

> 注意：视频页的自动连播跟合集一样，受播放器里的「播放模式」控制
> （默认是 `pause`，要在播放器菜单里选「顺序播放」或「列表循环」才会自动下一集）；
> 听视频页默认就是顺序播放。

---

## 六、修复记录（v2）

第一版装到 MuMu 上实测反馈了两个问题，都已修：

1. **「只出现了一首歌」** —— 原来按 `pageId` 聚合，而 `pageId = seasonId ?? avid`，同一合集里
   每集 avid 不同 → 每集各自成一个「页」→ 只剩当前那一首。
   现在按 `seasonInfo`（合集）分组，并且用 `seasonInfo.index` 排序，顺序就是合集里的集数顺序。
2. **少了播放顺序（无序/正序/倒序/随机）** —— 第一版把排序按钮在本地模式隐藏了（原来它是服务端
   排序）。现在本地模式也能排序，在客户端重排 `localItems` + `playlist`，不打断当前播放，
   当前曲目在重排后会被重新定位。

## 七、修复记录（v3）：本地听视频的列表循环

- 播放到最后一首时，「下一首」现在会按播放模式处理：列表循环 → 回到第一首；单曲/顺序模式 → 保持不动
- 随机 + 列表循环时，**每循环一圈都会重新洗牌整个列表**（以「正序」为种子重新 shuffle）
- 第一首按「上一首」也会按列表循环跳到最后一首

---

## 八、已知限制

- 本地「听视频」下**没有**点赞/投币/收藏/评论，这是有意的（离线场景保证不报错）
- 直播表情首帧可能退化为文字：画布是同步栅格化的，图片解码是异步的；同一条表情再次出现时就是图了
  （想要绝对准确需要把弹幕入队改成「预加载完成后再入队」，会引入一点延迟）
- APK 是 **debug 签名**（仓库里没有 `android/key.properties`），装到已有官方包/其他签名的手机上
  需要先卸载旧版本，或者自备签名文件
- 只在 arm64 单 ABI 上验证构建通过；`--split-per-abi` 三 ABI 包未逐个跑，但代码路径一致
