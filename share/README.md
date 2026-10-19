# PiliPlus 功能补丁（三个功能）

这份仓库里是给 [bggRGjQaUbCoE/PiliPlus](https://github.com/bggRGjQaUbCoE/PiliPlus) 的三个功能实现，
全部已在 Flutter 3.47.7 + Android ARM64 上构建并实机验证过。

## 三个功能

| # | 功能 | 补丁文件 |
|---|---|---|
| 1 | 已下载的视频支持「听视频」，含离线合集连播、自建文件夹 | `issue1-listen-audio.patch`（51 KB，16 个文件） |
| 2 | 直播弹幕在画面上显示表情图片（现在只有 `[表情名]` 文字） | `issue2-live-emote.patch`（157 KB，21 个文件） |
| 3 | 收藏夹支持多选批量下载（含分P、可选画质） | `issue3-fav-batch-download.patch`（9 KB，2 个文件） |

三个补丁互相独立，可以分别 review / 分别合并，**都不依赖另外两个**。

## 怎么用这些补丁

```bash
git clone https://github.com/bggRGjQaUbCoE/PiliPlus.git
cd PiliPlus
git checkout -b feat/xxx

# 按需应用（可以只应用其中一个）
git apply /path/to/issue1-listen-audio.patch

flutter pub get
powershell -ExecutionPolicy Bypass -File lib/scripts/patch.ps1 android
flutter build apk --release --target-platform android-arm64
```

补丁是**自包含**的：新增文件是完整内容，修改文件是精确 diff，`git apply` 之后文件会全部到位，
不需要再去找别的文件。

## 其他文件

| 文件 | 说明 |
|---|---|
| `feature-offline-audio-and-live-emote.patch` | 三个功能的**合集**（227 KB）。适合一次性构建完整包 |
| `canvas-danmaku-emote.patch` | 只含 `canvas_danmaku` 包的表情渲染改动（10 KB）。如果不想在 PiliPlus 里 vendor 整个包，可以先把这份提到 [canvas_danmaku](https://github.com/bggRGjQaUbCoE/canvas_danmaku) 仓库 |
| `source-changes.zip` | 改动后的**完整源码文件**（含 `third_party/canvas_danmaku`），不依赖 git，解压覆盖同路径即可 |
| `BUILD-NOTES.md` | 每个功能的实现说明、踩坑记录、已知限制 |
| `issues/` | 三个功能对应的 issue 原文（`[FR]` 格式），可以直接拿去提 issue |

## 对应的 issue

- #1 已下载视频支持「听视频」与离线合集连播 — `issues/issue-1-offline-listen-audio.md`
- #2 直播弹幕飘屏显示表情图片 — `issues/issue-2-live-emote.md`
- #3 收藏夹多选批量下载 — `issues/issue-3-fav-batch-download.md`
