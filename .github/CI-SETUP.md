# dart_simple_live-X 自动化配置说明

给 fork 仓库 [`qwqqvq123456/dart_simple_live-X`](https://github.com/qwqqvq123456/dart_simple_live-X) 加两个 GitHub Actions 工作流：

| 文件 | 作用 | 触发方式 |
| --- | --- | --- |
| `.github/workflows/sync-upstream.yml` | 自动把上游 `xiaoyaocz/dart_simple_live` 的 `master` 合并进本仓库，合并成功后自动触发构建 | 每天 09:00、21:00（北京时间）定时 + 手动 |
| `.github/workflows/build-android-release.yml` | 在 GitHub 的机器上编译 `simple_live_app` 的 release APK，产出可分架构下载的安装包并更新 Release | push 到 master、手动、被同步工作流调用 |

全程在 GitHub 云端完成，不需要本地 Android SDK / Flutter 环境。

---

## 一、安装（约 3 分钟，纯网页操作）

### 第 1 步：创建同步工作流

打开下面链接（会自动进入"新建文件"页面并填好路径）：

<https://github.com/qwqqvq123456/dart_simple_live-X/new/master?filename=.github/workflows/sync-upstream.yml>

把本目录下 `.github/workflows/sync-upstream.yml` 的全部内容粘贴进去 → 点 **Commit changes** → 直接提交到 `master`。

### 第 2 步：创建构建工作流

<https://github.com/qwqqvq123456/dart_simple_live-X/new/master?filename=.github/workflows/build-android-release.yml>

同样粘贴 `.github/workflows/build-android-release.yml` 的内容并提交。

> 提交到 `master` 本身就会触发第一次 Android 构建，可以先让它跑起来。

### 第 3 步：启用 Actions

fork 仓库的 Actions 默认可能是关闭的。打开：

<https://github.com/qwqqvq123456/dart_simple_live-X/actions>

如果看到绿色按钮 **I understand my workflows, go ahead and enable them**，点它。
然后在左侧列表里点 **同步上游主线** → 右侧 **Enable workflow**（fork 的定时任务默认不启用，必须手动启用一次，否则定时同步不会跑）。

### 第 4 步（可选，强烈建议）：配置固定的签名密钥

不配置也能出可安装的 APK，但每次构建的签名密钥是临时生成的，**新旧版本签名不一致，覆盖安装会失败**。配置一次即可长期固定。

```bash
# 本机执行（需要 JDK，任意目录）
bash scripts/gen-keystore.sh
```

脚本会生成 `android-release.jks`，并打印三样东西。到仓库
**Settings → Secrets and variables → Actions → New repository secret** 添加：

| Name | Value |
| --- | --- |
| `ANDROID_KEYSTORE_BASE64` | 脚本打印的那一大串 base64 |
| `ANDROID_KEYSTORE_PASSWORD` | 你设置的密码 |
| `ANDROID_KEY_ALIAS` | 默认 `simplelive` |

没有本地环境也可以在线生成：任意支持 `keytool` 的机器/容器执行同一条命令即可。
备份好 `android-release.jks` 和密码——丢了就无法对已装应用做覆盖升级。

---

## 二、怎么用

### 自动同步上游

- 每天北京时间 **09:00 / 21:00** 自动执行一次；
- 有上游更新时，会以 merge 方式合并到 `master`（**保留本仓库自己的改动**），推送后立刻触发 Android 构建；
- 没有更新时直接结束，不产生提交。

手动立即同步：
**Actions → 同步上游主线 → Run workflow**，可选参数：

| 参数 | 说明 |
| --- | --- |
| `force` | 丢弃本仓库所有改动，直接对齐上游 `master`（会保留这两个自定义工作流文件）。仅在合并冲突无法解决时使用 |
| `trigger_build` | 同步成功后是否自动触发构建，默认开启 |

### 自动构建 APK

触发条件：

- 任何推送到 `master` 的提交（纯文档改动会被跳过）；
- 被同步工作流调用；
- 手动：**Actions → 构建 Android Release → Run workflow**，可选 Flutter 版本、是否额外构建通用包、是否更新 Release。

构建完成后可以在两处拿到 APK：
1. **Release 页面（推荐，手机直接下载）**：
   <https://github.com/qwqqvq123456/dart_simple_live-X/releases/tag/android-latest>
   这个 Release 每次构建都会覆盖更新，地址固定不变。
2. **Actions 运行记录的 Artifacts**：保留 30 天，需登录 GitHub 才能下载。

产物命名：

| 文件 | 适用设备 |
| --- | --- |
| `SimpleLive-<版本>-arm64-v8a.apk` | 绝大多数手机（首选） |
| `SimpleLive-<版本>-armeabi-v7a.apk` | 老设备 / 32 位设备 |
| `SimpleLive-<版本>-x86_64.apk` | 模拟器 |
| `SimpleLive-<版本>-universal.apk` | 不确定机型时用这个 |

安装时需允许「未知来源应用」。若提示签名冲突，先卸载旧版本再装。

### 关于 `pubspec_overrides.yaml`（重要）

仓库里新增了一个 `simple_live_app/pubspec_overrides.yaml`，固定了两个依赖：

```yaml
dependency_overrides:
  auto_orientation_v2: 2.3.6
  dynamic_color: 1.8.1
```

原因（都是上游 `pubspec.yaml` 里 `^` 范围自动升级到新版本导致的，与工作流无关）：

| 包 | 解析到 | 问题 | 处理 |
| --- | --- | --- | --- |
| `auto_orientation_v2` | 2.4.7 | 改成 AGP 9 + `kotlin {}`，报 `Could not find method kotlin()` | 钉 2.3.6 |
| `auto_orientation_v2` | 2.4.6 | 由他人接手改版，`android/src` 是 Kotlin 源码但没 apply `kotlin-android`，类根本没被编译 → 报 `cannot find symbol class AutoOrientationPlugin` | 同上，钉 2.3.6（原始 Java 实现） |
| `dynamic_color` | 1.9.0 | `build.gradle.kts` 顶部直接写 `kotlin { compilerOptions {} }` 却没 apply Kotlin 插件 → 报 `Unresolved reference: compilerOptions` | 钉 1.8.1 |

用 `pubspec_overrides.yaml` 而不是改 `pubspec.yaml` 的好处：上游以后改这行不会和你的分支冲突，也不影响 Dart 侧代码。

等上游把 AGP 升到 9.x 后，删掉这个文件即可。以后再遇到同类 `Could not find method kotlin()` / `cannot find symbol` 报错，处理方式一样：在 `dependency_overrides` 里钉回上一个可用版本。

---

## 三、常见问题

**Q：同步工作流报 `合并冲突` 怎么办？**
它不会写坏仓库——检测到冲突会自动 `git merge --abort` 并失败退出，同时把处理建议写进该次运行的 Summary。三种处理：重跑并勾选 `force`（丢本地改动）、网页上手动解决、或本地合并后推送。

**Q：定时任务没跑？**
fork 仓库的 schedule 需要手动启用一次（第 3 步）；另外仓库连续 60 天无提交时 GitHub 会自动停用定时任务，去 Actions 页面点一下重新启用即可。

**Q：为什么同步上去的提交没有触发构建？**
GitHub 规定用内置 `GITHUB_TOKEN` 推送的提交不会触发 `push` 类工作流。所以同步工作流在推送后主动调用构建工作流，效果一样。若你希望同步后用个人身份推送（并在贡献图上显示），可加一个 `SYNC_TOKEN` secret（有 `repo` + `workflow` 权限的 PAT），工作流已优先使用它。

**Q：上游的 `publish_app_release.yml` 等旧工作流会不会报错？**
它们只在推送 `v*` / `tv_*` 这类标签时触发，平时不会运行；它们依赖上游作者自己的签名密钥，在你的仓库里跑会失败，不影响本方案。不想看到可以删掉。

**Q：想同步上游的 `dev` 分支 / 换一个上游？**
改 `sync-upstream.yml` 里的 `UPSTREAM_REPO` 和 `UPSTREAM_BRANCH` 两个环境变量即可（注意 `ref: master` 也要一起改）。

**Q：想同时构建 Android TV 版？**
把构建工作流里的 `APP_DIR` 改成 `simple_live_tv_app` 复制一份即可，依赖和流程完全一样。

**Q：要 Apache 风格签名 / 上架商店？**
本项目是 GPL-3.0，且 release 配置里开启了 R8 混淆；如需上架请自行在 `simple_live_app/android` 下调整签名与打包目标（`flutter build appbundle`）。

---

## 四、目录内容

```
dart_simple_live-X-actions/
├─ .github/workflows/sync-upstream.yml          # 自动同步上游
├─ .github/workflows/build-android-release.yml  # 自动构建 APK
├─ scripts/gen-keystore.sh                      # 生成固定签名密钥
└─ 使用说明.md
```
