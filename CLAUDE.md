# CellHelper —— v2rayNG 定制 fork 维护手册

本仓库是 [2dust/v2rayNG](https://github.com/2dust/v2rayNG) 的定制 fork，对外名为 **CellHelper**。
本文件给后续维护者（含 AI）说明：**改了什么、为什么、怎么跟随上游更新、怎么本地构建验证**。

> ⚠️ 下面列的改动都是**有意为之**，不是 bug。合并上游时遇到这些点的冲突，要保留本 fork 的行为，不要"改回"上游写法。

---

## 1. 本 fork 的定制清单（合并上游时的冲突热点）

源码包名仍是 `com.v2ray.ang`（`namespace` 没改），只有 `applicationId` 改了——所以大部分上游改动能干净合并，只有下面这些文件是定制热点：

| 定制点 | 位置 | 说明 |
|---|---|---|
| **applicationId** | `app/build.gradle.kts` | `com.cloudorz.v2ng`（与官方共存）。`namespace` 保持 `com.v2ray.ang` 不要动 |
| **isXray() 恒 true** | `util/Utils.kt` | 本 fork 恒为 Xray 核心，解耦 applicationId。上游是 `startsWith("com.v2ray.ang")` |
| **默认 SOCKS 端口 2080** | `AppConfig.PORT_SOCKS`、`assets/v2ray_config.json`、`assets/v2ray_config_with_tun.json`、`res/xml/pref_settings.xml`(summary) | 上游为 10808 |
| **强制全局** | `handler/SettingsManager.kt` `routingRulesetsBypassLan()` 恒 `false`；`initRoutingRulesets()` 用 `RoutingType.GLOBAL`；`assets/custom_routing_global` 为单条 proxy-all | tun 全量接管，连 LAN 都不绕过 |
| **删除路由/分应用 UI** | 已删 `ui/PerAppProxyActivity`、`PerAppProxyAdapter`、`AppPickerActivity`、`AppSelectorAdapter`、`RoutingSettingActivity`、`RoutingEditActivity`、`RoutingSettingRecyclerAdapter` 及对应 layout/menu；`MainActivity` 抽屉分支、`menu_drawer.xml`、`pref_settings.xml`(RouteOnly/bypass_lan) 已移除 | 后端路由函数保留 |
| **保活/防泄漏** | `service/CoreVpnService.kt` + `core/CoreServiceManager.kt` | `onTaskRemoved` 不停服、`PARTIAL_WAKE_LOCK`、core 看门狗重启不拆 tun、`userStopRequested` 区分主动停止 vs 崩溃。设置里 `pref_always_on_vpn` 深链系统 VPN 设置 |
| **砍掉推广/外链** | 已删推广菜单项、About 的源码/反馈/TG/隐私链接、mode-help wiki、check-for-update（含 `CheckUpdateActivity`/`UpdateCheckerManager`/`CheckUpdateResult`）；`AppConfig` 删了 6 个 promo/社区 URL 常量 | 保留 OSS 许可证、版本号；保留功能性的 geo 下载 URL |
| **品牌/外观** | `res/values/strings.xml` `app_name=CellHelper`；`res/drawable/ic_launcher_foreground.xml`（信号格矢量图标）；`mipmap-anydpi-v26/ic_launcher*.xml`；`values/ic_launcher_background.xml`(#1565C0)；`values/colors.xml` + `values-night/colors.xml` 背景加浅蓝 | API26+ 用新自适应图标；API25- 用旧 PNG 兜底 |
| **CI 免 secret 构建** | `.github/workflows/build.yml` "Prepare signing keystore" 步骤 | 无 `APP_KEYSTORE_BASE64` 时现场生成临时自签 keystore |

> 完整设计文档见 `~/.claude/plans/twinkling-petting-blanket.md`（如还在）。

---

## 2. 跟随上游更新（同步流程）

远端已配置：`origin`=你的 fork，`upstream`=2dust/v2rayNG（push 已禁用，防误推）。

```bash
# 1. 拉上游
git fetch upstream

# 2. 在干净工作区，基于 master 开一个同步分支（不要直接在 master 上合，便于回退）
git switch master
git switch -c sync-upstream

# 3. 合并上游（也可用 rebase，团队习惯二选一；merge 更安全、保留历史）
git merge upstream/master
#   ↑ 冲突大概率只出现在"第 1 节定制清单"里的那几个文件。
#   解决原则：保留本 fork 的定制行为（端口/包名/全局/保活/去推广/品牌），
#   其余上游改动照单全收。

# 4. 注意上游可能"重新引入"被我们删掉的东西：
#   - 新增的路由/分应用入口、推广位、检查更新、外链 → 按第 1 节的方式再砍掉
#   - 新的设置项默认值若破坏"强制全局" → 跟进修正
#   - libv2ray/hevtun 子模块指针更新 → 见第 3 节重新拉/编

# 5. 本地构建验证（见第 3 节），跑通后：
git switch master
git merge --no-ff sync-upstream
git push origin master
git branch -d sync-upstream
```

**版本号**：上游会改 `app/build.gradle.kts` 的 `versionCode`/`versionName`，一般直接接受上游的即可。

**子模块**：上游若更新了 `AndroidLibXrayLite` / `hev-socks5-tunnel` 的指针，`git submodule update --init --recursive` 后按第 3 节重新获取 aar / 重编 hevtun。

---

## 3. 本地构建 & 安装（已验证可用）

环境：macOS，JDK 17，Android SDK（含 platform android-37）。Gradle 用 wrapper（9.5.1）。
**本仓库不含原生依赖**（`libv2ray.aar`、hevtun `.so`、`local.properties` 均被 .gitignore），首次构建需手动准备：

```bash
ROOT=/Users/mt/project/v2rayNG-fork          # 仓库根
export ANDROID_HOME="$HOME/Library/Android/sdk"
export JAVA_HOME="$(/usr/libexec/java_home -v 17)"

# (a) 子模块
cd "$ROOT" && git submodule update --init --recursive

# (b) 下载 libv2ray.aar（Xray 核心；编译期必需，否则 libv2ray.* 无法解析）
#     用 2dust/AndroidLibXrayLite 的某个 release（日期 tag，如 v26.6.14）
mkdir -p "$ROOT/V2rayNG/app/libs"
gh release download v26.6.14 --repo 2dust/AndroidLibXrayLite \
  --pattern 'libv2ray.aar' --dir "$ROOT/V2rayNG/app/libs" --clobber

# (c) 编 hevtun .so（运行期必需，因为 isUsingHevTun 默认 true）
#     NDK 28 即可（纯 C 库，不必非 29）
export NDK_HOME="$HOME/Library/Android/sdk/ndk/28.2.13676358"
cd "$ROOT" && bash compile-hevtun.sh
cp -r "$ROOT/libs/"* "$ROOT/V2rayNG/app/libs/"     # 4 个 ABI 的 .so 进 app/libs

# (d) local.properties
printf 'sdk.dir=%s\n' "$ANDROID_HOME" > "$ROOT/V2rayNG/local.properties"

# (e) 构建（无 secret 时用临时 keystore，模拟 CI）
cd "$ROOT/V2rayNG"
KS="$PWD/temp_keystore.jks"; rm -f "$KS"
keytool -genkeypair -v -keystore "$KS" -alias androidkey -keyalg RSA -keysize 2048 \
  -validity 10000 -storepass android -keypass android \
  -dname "CN=v2ng, OU=fork, O=cloudorz, L=NA, ST=NA, C=NA"
./gradlew :app:assemblePlaystoreRelease --no-daemon \
  -Pandroid.injected.signing.store.file="$KS" \
  -Pandroid.injected.signing.store.password=android \
  -Pandroid.injected.signing.key.alias=androidkey \
  -Pandroid.injected.signing.key.password=android
rm -f "$KS"

# (f) 安装（设备/模拟器，arm64 示例）
adb install -r app/build/outputs/apk/playstore/release/v2rayNG_*_arm64-v8a.apk
```

调试期想快速验证编译，可用 `:app:assemblePlaystoreDebug`（不需要 keystore）。
产物 APK 路径：`V2rayNG/app/build/outputs/apk/playstore/release/`。

---

## 4. 改动后的验证清单

- 包名 `com.cloudorz.v2ng`（`aapt dump badging` 看 `package:`）、可与官方 v2rayNG 共存
- App 名 = CellHelper；蓝底信号格图标；界面有淡蓝背景
- SOCKS 监听 2080；连上后全局代理，抽屉/设置里**无**路由、分应用、检查更新、推广入口
- About 只剩 OSS 许可证 + 版本号；设置里有"始终连接 VPN 与防泄漏"入口
- 防泄漏：连上后从最近任务划掉 → VPN 不断、通知还在；core 崩溃期间不漏流量、自动重启
- CI：fork 未配 secret 时 `build.yml` 仍能出可安装 APK

---

## 5. 约定

- 提交/推送前确认在 `origin`（你的 fork），不要推 `upstream`（已禁用 push）
- 默认分支 `master`；同步上游走临时分支，验证通过再并回
- 改动涉及"第 1 节"任一定制点时，更新本文件对应条目
