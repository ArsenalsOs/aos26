# CLAUDE.md — ArsenalsLOS (LineageOS 23.2) 工作树

> 本文件供 Claude（与维护者）快速理解本仓库的性质、构建方式与版本管理边界。
> 所有结论基于 2026-07-22 对 `/root/arsenals/aos/los` 的实际分析。

## 1. 项目概述

| 项 | 值 |
|---|---|
| 仓库性质 | **LineageOS 23.2** 自定义构建工作树（项目内部代号 ArsenalsLOS） |
| Android 版本 | **Android 16 (Baklava)**，AOSP tag `android-16.0.0_r4`（2025-12, Android 16 QPR2；上一版 `android-16.0.0_r3` / 2025-09 QPR1） |
| OS 分支 | `lineage-23.2` |
| Release config | `BP4A`（`vendor/lineage/vars/aosp_target_release` → `aosp_target_release=bp4a`） |
| Manifest 上游 | `https://github.com/LineageOS/android.git`（本树实际用清华镜像，见下） |
| 当前使用的 manifest remote | `https://mirrors.tuna.tsinghua.edu.cn/git/lineageOS/LineageOS/android.git` |
| 子项目数量 | **1164** 个 git 项目（`.repo/project.list`） |
| 自定义 local_manifests | **无**（当前为纯上游 LineageOS 23.2，未叠加自定义设备/补丁 manifest） |
| 工作目录 | `/root/arsenals/aos/los` |
| 编译机资源 | 16 核 / 49 GB RAM / 16 GB swap |

版本号组成：`LINEAGE_VERSION = 23.2-<date>-<buildtype><extra>-<build>`（见 `vendor/lineage/config/version.mk`）。
`LINEAGE_BUILDTYPE`：`RELEASE`/`NIGHTLY`/`SNAPSHOT`/`EXPERIMENTAL`，其余归为 `UNOFFICIAL`（默认）。

## 2. ⚠️ 版本管理边界（最重要）

本目录存在**两套独立的版本管理**，互不混淆：

### 2.1 `repo` 工具 —— 管理上游 Android/LineageOS 源码树
- 由 `.repo/` 目录 + 1164 个子项目（各自通过 `.repo/projects`、`.repo/project-objects` 管理其 `.git`）构成。
- 同步用 `repo sync`，分支用 `repo start <br> <project>`，拣选用 `lineage/scripts/repopick`。
- **这 1164 个项目的源码绝不纳入顶层 `.git`。** `.gitignore` 已用白名单将其全部忽略。

### 2.2 顶层 `.git` —— 仅管理用户自定义内容
- 本仓库根目录的 git，**只跟踪 `CLAUDE.md`、`.gitignore` 及手动放行的自定义文件**。
- 新增自定义内容（脚本/补丁/文档）时，在 `.gitignore` 添加对应 `!路径` 例外。
- **严禁** `git add .` / `git add -A`（会扫描 1164 项目树，极慢且违反边界）。只 `git add <显式文件>`。
- 修改上游源码的正确做法：在对应子项目内 `repo start <branch>` 后提交，或用 `repopick` 从 LineageOS Gerrit 拉取 patch —— 不要把上游改动塞进顶层 git。

## 3. 顶层目录结构

32 个顶层目录 + 1 个 repo 生成文件 `lk_inc.mk`（内容仅 `include trusty/vendor/google/aosp/lk_inc_aosp.mk`，由 repo link-files 产生）。

| 目录 | 说明 |
|---|---|
| `lineage/` | **LineageOS 专属**：`scripts/`(repopick 等)、`wiki/`、`crowdin/`、`hudson/`(CI)、`charter/`、`website/`、`mirror/` |
| `lineage-sdk/` | LineageOS SDK（`lineage` 平台 API、Java/Native 接口） |
| `vendor/lineage/` | **LineageOS 产品定义核心**：`config/`(版本/构建配置)、`vars/`(设备/release 变量)、`release/`(release config)、`overlay/`、`build/`(envsetup/soong/core/target/tasks)、`product/`、`bootanimation/`、`charger/`、`audio/`、`spn/`、`tools/` |
| `build/` | AOSP 构建系统。`build/envsetup.sh` → 软链 `build/make/envsetup.sh`；含 `make/`、`soong/`、`blueprint/`、`core/`→`make/core`、`target/`、`tools/`、`release/` |
| `device/` | 设备树。当前仅通用：`generic/`、`google/`、`google_car/`、`sample/`、`qcom/`(sepolicy)、`lineage/`(atv/car/sepolicy)。**无具体手机 OEM 设备树**，需通过 local_manifests 按需引入 |
| 其他标准 AOSP 树 | `art` `bionic` `bootable` `cts` `dalvik` `developers` `development` `external`(480) `frameworks` `hardware` `kernel` `libcore` `libnativehelper` `packages` `pdk` `platform_testing` `prebuilts` `sdk` `system` `test` `toolchain` `tools` `trusty` `vendor` `android` |

## 4. 版本与 release 机制

- **版本**：`vendor/lineage/config/version.mk` —— `PRODUCT_VERSION_MAJOR=23`、`MINOR=2`，派生 `LINEAGE_VERSION` / `LINEAGE_DISPLAY_VERSION` 与 `ro.lineage.version` 等属性。
- **TARGET_RELEASE（Android 16 新引入的 release config 机制）**：
  - 默认值 `bp4a` 来自 `vendor/lineage/vars/aosp_target_release`。
  - release config 声明于 `vendor/lineage/release/release_config_map.mk`：`$(call declare-release-config, bp4a, .../build_config/bp4a.scl)`，具体 flag 值在 `vendor/lineage/release/flag_values/` 与 `release_configs/bp4a.textproto`。
  - `breakfast` 会先 `source vendor/lineage/vars/aosp_target_release` 再 `lunch lineage_<dev>-bp4a-<variant>`。
- **`vars/common`**：`android_version=16`、`os_branch=lineage-23.2`、`topic=BP4A`、`common_aosp_tag=android-16.0.0_r4`、`prev_common_aosp_tag=android-16.0.0_r3`。

## 5. 设备支持现状

- **Pixel 系列**：`vendor/lineage/vars/` 下每个代号一个文件 —— `akita`(Pixel 8a) `comet` `felix` `husky` `komodo` `lynx` `oriole` `panther` `raven` `redfin` `shiba` `tokay` `tangorpro` `tegu` `caiman` `cheetah` 等。每个含 `build_id`(如 `BP4A.260205.001`)、`build_number`、`image_url`/`ota_url`(Google factory/OTA) 及 sha256，供脚本比对上游。`vars/pixels` 为支持的 Pixel 列表。
- **高通平台**：`vendor/lineage/vars/qcom` 以 `qcom_group_revision[<platform>]` 定义各平台 CAF revision —— `qssi`(LA.QSSI.16.0.r1)、`msm8953`、`sdm660`、`sdm845`、`msmnile`、`kona`、`lahaina`、`waipio`/`waipio-6.6-vendor` 等。
- **`device/` 现状**：仅有通用树与 atv/car，**没有具体 OEM 手机设备树**。引入新设备需在 `.repo/local_manifests/*.xml` 添加该设备的 manifest（vendor/device/kernel 仓库），并按需补充对应 `vars/<device>`。

## 6. 构建流程

```bash
# 0) 进入工作目录
cd /root/arsenals/aos/los

# 1) 加载构建环境（build/envsetup.sh -> build/make/envsetup.sh，会链式 source vendor/lineage/build/envsetup.sh，
#    从而注册 breakfast/brunch/eat 等函数）
. build/envsetup.sh

# 2) 选择设备与变体（任选其一）
breakfast <device>                 # 等价 lunch lineage_<device>-bp4a-userdebug（默认变体 userdebug）
breakfast <device> user            # 指定变体
brunch <device>                    # = breakfast + mka bacon（一键编译并打 OTA）
lunch lineage_<device>-bp4a-userdebug   # 手动完整形式：lineage_<dev>-<release>-<variant>

# 3) 编译
mka bacon                          # 编译并打包 OTA zip（LineageOS 的 bacon target）
mka                                # 仅编译系统镜像，不打包 OTA
m <module>                         # 单模块

# 4) 产物
#    out/target/product/<device>/lineage-23.2-<date>-<buildtype>-<device>.zip

# 5) 安装到已连接并进入 sideload 的设备
eat                                # 自动 adb sideload 最新 lineage-*.zip（需设备 ro.lineage.device 匹配）
# 或手动：adb reboot sideload && adb sideload <zip>
```

构建函数定义见 `vendor/lineage/build/envsetup.sh`（`breakfast`/`brunch`/`eat`/`check_product`/`bib` 别名）。
`mka` 为 envsetup 提供的并行 make 包装（带 schedtool）；`bacon` 为 LineageOS 的 OTA 打包 target。

## 7. 编译环境（参考 ArsenalsOs 搭建文档，通用于本树）

- **系统**：WSL2 / Ubuntu 22.04；`sudo ln -s /usr/bin/python3 /usr/bin/python`。
- **apt 依赖**：`bc bison build-essential ccache curl flex g++-multilib gcc-multilib git gnupg gperf imagemagick lib32ncurses5-dev lib32readline-dev lib32z1-dev libelf-dev liblz4-tool libncurses5 libncurses5-dev libsdl1.2-dev libssl-dev libxml2 libxml2-utils lzop pngcrush rsync schedtool squashfs-tools xsltproc zip zlib1g-dev`
- **repo 工具**：`~/bin/repo`；`export REPO_URL='http://mirrors.tuna.tsinghua.edu.cn/git/git-repo'`；`git config --global url.http://mirrors.tuna.tsinghua.edu.cn/git/AOSP/.insteadof https://android.googlesource.com`
- **代理**：`export ALL_PROXY="http://<host_ip>:10811"`（host_ip 见 `/etc/resolv.conf`）
- **swap**：16 GB（`.wslconfig` 的 `[wsl2] swap=16GB`，或 `/swapfile` + `mkswap`/`swapon`）
- **ccache**：`export USE_CCACHE=1`、`export CCACHE_EXEC=/usr/bin/ccache`、`export CCACHE_DIR=~/ccache`、`ccache -M 50G`、`ccache -s` 查看状态

## 8. 代码同步

```bash
# 清华镜像（本树当前所用）
repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/lineageOS/LineageOS/android.git -b lineage-23.2 --git-lfs

# 或 GitHub 上游
repo init -u https://github.com/LineageOS/android.git -b lineage-23.2 --git-lfs

repo sync -c -j$(nproc --all) --force-sync --no-clone-bundle --no-tags
```

自定义覆盖（添加设备/移除项目）放 `.repo/local_manifests/*.xml`，不随 repo 的 default.xml 被覆盖。

## 9. 签名

LineageOS 23.2 采用 Android 16 的 **release config** 机制（`vendor/lineage/release/`），但生成自签名 release 证书的方法仍通用：

```bash
# 1) 生成证书（development/tools/make_key）
subject='/C=CN/ST=Shaanxi/L=Baoji/O=Arsenals/OU=luyuedong/CN=arsenals/emailAddress=460999218@qq.com'
mkdir -p ~/.arsenals-certs
for x in releasekey platform shared media networkstack; do
    ./development/tools/make_key ~/.arsenals-certs/$x "$subject"
done

# 2) 转换为 keystore（pk8 -> pem -> pk12 -> keystore）
cd ~/.arsenals-certs
openssl pkcs8  -in releasekey.pk8 -inform DER -outform PEM -out releasekey.priv.pem
openssl pkcs12 -export -in releasekey.x509.pem -inkey releasekey.priv.pem \
    -out releasekey.pk12 -name ReleaseKey
keytool -importkeystore -deststorepass '***REDACTED***' -destkeypass '***REDACTED***' \
    -destkeystore releasekey.keystore -srckeystore releasekey.pk12 \
    -srcstoretype PKCS12 -srcstorepass '***REDACTED***' -alias releasekey

# 3) 让构建使用自签证书：在 build/make/core/config.mk 设置
#    PRODUCT_DEFAULT_DEV_CERTIFICATE := <你的证书目录>/releasekey
#    （LineageOS 23.2 也可通过 release config 的 flag 覆盖签名指向，见 vendor/lineage/release/）
```

## 10. 关键脚本（`lineage/scripts/`）

| 脚本 | 用途 |
|---|---|
| `repopick` | 从 LineageOS Gerrit (`review.lineageos.org`) 拣选 patch/topic 到本地工作分支 |
| `lineage-priv-template` | priv-app 签名模板 |
| `key-migration` | 密钥迁移 |
| `device-deps-regenerator` | 设备依赖重生 |
| `aosp-merger` | 合并上游 AOSP 改动 |
| `best-caf-kernel` | CAF 内核选取 |
| `build-webview` | 构建 WebView |
| `emoji-updater` | 更新 emoji 字体 |
| `carriersettings-extractor` | 提取运营商配置 |
| `pixel/update-device-vars.sh`、`pixel/update-any-var.sh` | 更新 Pixel 设备 vars（factory image/OTA sha、build_id 等） |
| `add-repo`、`lineage-push`、`git-push-merge-review`、`shipper`、`set-default-branch` | 仓库/Gerrit 协作 |

## 11. 给 Claude 的工作指引

- **改源码前先判性质**：若目标属上游 1164 项目之一，优先 `repo start` 在子项目内改/提交，或 `repopick` 拉 patch；不要把上游改动塞进顶层 git。
- **顶层 git 只动自定义文件**：新增自定义内容需同步在 `.gitignore` 加 `!` 例外；提交只 `git add <显式路径>`。
- **编译前**：确保已 `. build/envsetup.sh` 且已 `breakfast`/`lunch`；否则 `mka` 找不到 target。
- **查设备代号/变量**：`vendor/lineage/vars/<device>`（Pixel 各代号、`common`、`qcom`、`pixels`）。
- **查版本/release**：`vendor/lineage/vars/common`、`vars/aosp_target_release`、`vendor/lineage/config/version.mk`、`vendor/lineage/release/`。
- **引入新设备**：写 `.repo/local_manifests/<device>.xml` 引入 device/vendor/kernel 仓库，必要时补 `vars/<device>`；`device/` 当前无具体 OEM 树。
- **构建产物路径**：`out/target/product/<device>/`；`out/` 已被 `.gitignore` 忽略。
- **参考资料局限**：会话内 ArsenalsOs 搭建文档的示例 lunch（`aosp_haydn`/`evolution_marble`）属 PixelExperience/EvolutionX，本树为 LineageOS 23.2，请用 `breakfast`/`brunch`/`lineage_<dev>-bp4a-<variant>`。

## 12. 常用命令速查

```bash
# 环境
. build/envsetup.sh

# 选设备
breakfast <device>                          # = lunch lineage_<device>-bp4a-userdebug
brunch <device>                             # breakfast + mka bacon
lunch lineage_<device>-bp4a-userdebug

# 编译
mka bacon                                   # 编译 + 打 OTA zip
mka                                         # 仅镜像
mka installclean                            # 清安装产物
make clobber                                 # 删 out/（完全清理）

# ccache
ccache -s && ccache -M 50G

# repo
repo status                                  # 各项目状态
repo forall -c 'git status -s'              # 全项目执行
repo start <branch> <project...>            # 开工作分支
repo sync -c -j$(nproc --all)               # 同步

# 拣选 / 安装
repopick <topic或commit>                    # 从 Gerrit 拉 patch
eat                                         # adb sideload 最新 lineage-*.zip
adb reboot sideload && adb sideload <zip>   # 手动刷入
```

## 13. 参考资源

- LineageOS Wiki：https://wiki.lineageos.org/
- LineageOS Gerrit：https://review.lineageos.org/
- 构建状态：https://buildkite.com/lineageos
- 下载：https://download.lineageos.org/
- 编译环境搭建：会话内《ArsenalsOs 环境搭建》文档（WSL2/apt/repo/代理/swap/ccache/签名）
