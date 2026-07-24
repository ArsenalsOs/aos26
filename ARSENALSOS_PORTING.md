# ArsenalsOS 定制改动全量检索与移植清单

> 检索日期 2026-07-22。基线：ArsenalsOS = LineageOS 21.0 / Android 14（AOSP tag `android-14.0.0_r25`，manifest `ArsenalsOs/aos_manifest`）。移植目标：LineageOS 23.2 / Android 16（目标树 `/root/arsenals/aos/los`，AOSP tag `android-16.0.0_r4`，release `bp4a`）。
> 源码树 `/root/arsenals/aos/aos`（旧）。下列结论由 4 路并行检索 agent + 直接读取源码交叉验证（含在 los/23.2 目标树实测比对）得出。

---

## 移植后状态（2026-07-24 逐 commit 核查后更新）

**移植已完成**。另一 agent 逐 commit 核查：38/38 已配对，~22 完整移植 / ~10 移植-适配，1 部分移植（kernelsu Minor），0 意外引入。核心定制（AOS 服务 / rebrand / releasekey / KernelSU / marble / MindTheGapps）移植充分，A16 适配有高质量改进（@hide 规避 metalava / reflection / getSystemService / shareduid allowlist / kprobes hook / releasekey 体系闭合 / rebrand 比原版更完整 / neverallow 核查通过）。

**核查结论与本文档勘误**：

- ✅ **C1 snet spoof（AosHooks / AndroidKeyStoreSpi / SetSafetyNetProps）有意不移植** —— 非遗漏。21.0 编译期 framework 篡改在 A16 keystore2 大变 + 硬件 attestation 收紧下已失效。改用 **PIF + TrickyStore 模块**（运行时 zygisk hook + resetprop + keybox）替代，与 EvolutionX/crDroid/PixelOS A16 一致。源码 grep 零命中符合预期（PIF 是模块，不在源码树）。详见记忆 `reference-pif-trickystore`。**本文档 §10 P1#7#8 标"待重写"应理解为"有意不移植 + PIF/TrickyStore 模块替代"。**
- ✅ **C2 内核 5.10 不需升级 6.x** —— LineageOS 23.2 官方 marble 就用 5.10（跟随官方），A16 不强制 6.x（5.10 GKI android13-5.10 KMI 兼容 A16 userspace）。**已实机 boot 成功**（baseline + KSU boot 5.10 内核开机正常，manager 连上）。本文档 §10 P1#9"升级 6.x 最大工程"在 23.2 **不再需要**。
- ✅ **M1 aos_service 专用 SELinux 域已补**（commit `6df6cb2c1` system/sepolicy）：`type aos_service, system_api_service, system_server_service, service_manager_type` + service_contexts `aos` 条目 + system_server/untrusted_app_all allow + prebuilts/api/202504 同步。对齐 21.0 `bdd66580`/`e1e99216`。`service_fuzzer_bindings.go` 23.2 已删不再适用。
- ⚠️ **M2 manifest 持久化待补**：ArsenalsOS commit 在各子项目本地分支，`repo sync --force-sync` 会丢。需建自有 fork（github ArsenalsOs/aos_*）+ local_manifests 覆盖（类 21.0 aos.xml，参考 `~/aos-manifest-reference.xml`）。
- **§5 勘误**：`583dfba9 merge_dtbs` 误标为 Y-D-Lu 专属，实为上游 Arian Kulmer commit，23.2 上游已含，**免移植**。vendor/lineage fork 实际专属 commit 为 4 个（非 5）。
- **§4 勘误**：device/lineage/sepolicy 两 Revert（legacy camera HAL1 `ff9318d` + ultra-legacy qcom `151f52b`）在 23.2 **不再适用/不需移植**（核查确认不移植正确）。
- **§10 P2#12 勘误**：`buildinfo.sh`（ro.arsenals.device）在 23.2 已被 `build/soong/scripts/gen_build_prop.py` 取代，移植改的是后者（非 buildinfo.sh）。
- **Minor 待确认**：vendor/xiaomi/marble thermal（`b3cdd594` 清空 thermal-*.conf）原意是禁热限流（性能调优，非移植必需，有烧机风险），23.2 未做，待用户定。

**已完成 commit 清单**（los 树各子项目本地分支）：vendor/lineage、frameworks/base（9660b93 + 69c147b）、build/soong（918286b90 + 3410717a6）、build/make、device/xiaomi/marble、device/lineage/sepolicy（546c956）、device/arsenals/sepolicy、vendor/arsenals/{aos,arsenalsos}、kernel/arsenals/kernelsu（79feeb91 + feef0dd6 + f7fb4da1，KSU_VERSION=32563，签名 0x03fd）、kernel/xiaomi/sm8450、**system/sepolicy（6df6cb2c1，本次补 aos service 域）**。

---

## 0. 总览：定制高度集中

ArsenalsOS 的全部专属代码只落在少数几个仓库/目录，且围绕三大功能主题：

1. **AOS 系统服务**（自定义 framework 服务 + 反射管道）
2. **SafetyNet / Play Integrity 规避**（Build 字段欺骗 + 硬件证书链篡改 + init 属性伪造）
3. **品牌 rebrand + 自签 releasekey + 小米 marble 设备 + KernelSU 内核 root**

| 仓库 / 目录 | 专属 commit | 专属文件 | 主题 | 23.2 是否需移植 |
|---|---|---|---|---|
| `frameworks/base` (fork `aos_frameworks_base`) | 3 | 6 (+179 -2) | snet spoof + AosService 管道 | ✅ 需重做 |
| `frameworks/native` (fork `aos_frameworks_native`) | **0** | 0 | **纯上游镜像** | ❌ **直接用上游 23.2** |
| `bionic` (fork `aos_bionic`) | **0** | 0 | **纯上游镜像** | ❌ **直接用上游 23.2** |
| `system/core` (fork `aos_system_core`) | 1 | 2 (+51) | init 属性欺骗 | ✅ 需重做 |
| `build/make` (fork `aos_build_make`) | 5 | 19 | releasekey 默认 + rebrand + check_product | ✅ 需重做（落点变） |
| `vendor/lineage` (fork `aos_vendor_lineage`) | 5 | 7 | ro.arsenals.* + OFFICIAL + merge_dtbs | ✅ 需重做 |
| `device/lineage/sepolicy` (fork) | 3 | — | 接入 arsenals sepolicy + 回退 legacy | ✅ |
| `device/qcom/sepolicy_vndr/sm8450` (fork `aos-caf-sm8450`) | 1 | — | fix hal_health avc | ✅ |
| `vendor/arsenals/aos` + `arsenalsos` | 新增 | ~8 | AOS 系统服务实现 | ✅ 需重新引入 |
| `device/arsenals/sepolicy` | 新增 | 8 .te | platform_app 读 sysfs | ✅ |
| `kernel/arsenals/kernelsu` | 新增(fork tiann) | GPL 内核 | 内核 root | ✅ 需适配 6.x |
| `device/xiaomi/marble` + `sm8450-common` | 手动 clone | — | 小米 SM8450 设备树 | ✅ 需重引入+适配 |
| `vendor/xiaomi` | 手动 clone | — | proprietary blobs（基线 Android 12） | ✅ 需重 extract |
| `vendor/gapps` | 手动 clone | — | MindTheGapps | ✅ |
| `kernel/xiaomi/sm8450`(+`-modules`) | 手动 clone | — | GKI 5.10 内核 | ✅ **最大工程（→6.x）** |

## 1. manifest 层（`.repo/manifests/snippets/aos.xml`）

- manifest = `ArsenalsOs/aos_manifest`，`default.xml` 末尾 `include snippets/lineage.xml` 再 `include snippets/aos.xml`（后置覆盖同名 project）。
- **4 个 remote**：`github`(默认,真 github.com)、`lineage`(清华 LOS 镜像)、`aosp`(清华 AOSP 镜像)、`private`(ssh)、`losul`。
- **aos.xml 15 project 全 `ArsenalsOs/aos_*`**：
  - `refs/heads/aos` ×12：`kernel/arsenals/kernelsu`、`vendor/lineage`、`bionic`、`build/make`、`frameworks/base`、`frameworks/native`、`system/core`、`hardware/qcom/wlan`、`device/lineage/sepolicy`、`device/arsenals/sepolicy`、`vendor/arsenals/aos`、`vendor/arsenals/arsenalsos`
  - `refs/heads/aos-caf-sm8450` ×2：`hardware/qcom-caf/sm8450/display`、`device/qcom/sepolicy_vndr/sm8450`
  - `refs/heads/aos-caf` ×1：`hardware/qcom-caf/wlan`
- **手动 clone（不在任何 manifest，project.list 不含，移植须自行加入）**：`device/xiaomi/marble`、`device/xiaomi/sm8450-common`、`vendor/xiaomi`、`vendor/gapps`、`kernel/xiaomi/sm8450`、`kernel/xiaomi/sm8450-modules`、`hardware/xiaomi`。

> ⚠️ **manifest remote 移植大坑（已实测 los/23.2）**：21.0 用 `<default remote="github" fetch="https://github.com/">`；**23.2 改为 `fetch=".."`（相对路径 → 指向 LOS 镜像主机），不再有 `github` remote**。故 aos.xml 里的 `ArsenalsOs/aos_*` project **不能继承 23.2 默认 remote（会去 LOS 镜像找 → 404）**。移植时必须在 snippet/local_manifest 里**显式声明一个 `fetch="https://github.com"` 的 remote** 给所有 ArsenalsOs fork project，且各 fork 仓需新建基于 `lineage-23.2` 的 `aos` 分支。

## 2. framework 层

### frameworks/base（fork，3 commit by Y-D-Lu，基线 `e9e05048 init from LineageOS lineage-21.0`）
- `4444ee59` snet spoof workaround — frameworks/base part
- `35b88746` init AosService
- `c5ef0f27` move AosService to vendor/arsenals/arsenalsos

**6 个改动文件（+179 -2）**：
| 文件 | 改动 |
|---|---|
| `core/java/com/android/internal/util/aos/AosHooks.java`（新增 98 行） | GMS/Finsky 包检测；Build 字段欺骗（伪装 Motorola griffin/XT1650-05，`SECURITY_PATCH=2016-07-01`）；`isCallerSafetyNet()` 栈查 DroidGuard；`onEngineGetCertificateChain()` 阻断证书链 |
| `core/java/android/app/Instrumentation.java`（+3） | 两个 `newApplication()` 重载调 `AosHooks.onNewApplication(app)` |
| `keystore/java/android/security/keystore2/AndroidKeyStoreSpi.java`（+53 -2） | `engineGetCertificateChain()` 起首调 AosHooks；42 字节 ASN.1 attestation 模式匹配，命中后改 offset+38=1 / offset+41=0 把硬件认证降级软件认证；检查 EAT_OID/ASN1_OID/KNOX_OID 扩展 |
| `core/java/android/content/Context.java`（+8，L6440） | `@SystemApi @hide public static final String AOS_SERVICE = "aos";` |
| `core/api/system-current.txt`（+1，L3448） | `field public static final String AOS_SERVICE = "aos";` |
| `services/java/com/android/server/SystemServer.java`（+18 净） | L951 `startAosServices(t)`（在 `startApexServices(t)` 之后），反射加载 `cn.arsenals.arsenalsos.AosServicesManager` 调其 `startAosServices()` |

反射引导代码（跨版本只需重放 hook 位置，不依赖 framework 内部 API）：
```java
private void startAosServices(@NonNull TimingsTraceAndSlog t) {
    try {
        Class<?> clazz = Class.forName("cn.arsenals.arsenalsos.AosServicesManager");
        Constructor<?> ctor = clazz.getDeclaredConstructor(Context.class); ctor.setAccessible(true);
        Object obj = ctor.newInstance(mSystemContext);
        Method m = obj.getClass().getDeclaredMethod("startAosServices"); m.setAccessible(true);
        m.invoke(obj); // 内部 ServiceManager.addService(Context.AOS_SERVICE, new AosService(ctx))
    } catch (Throwable e) { Slog.wtf(TAG, "startAosServices catch " + e); }
}
```
> 注：`35b88746` 曾在 `SystemServiceRegistry` 注册 AosManager，但 `c5ef0f27` 全部删除并移到 vendor 侧——最终 frameworks/base 只剩常量 + 反射入口。

**移植风险（A16）**：
- `AndroidKeyStoreSpi.java` **最高风险**：keystore2 在 A15/A16 大幅演进，42 字节匹配模式与 offset 修改强依赖证书编码格式，极可能失效；EAT/ASN1/Knox OID 检查需重验。
- `SystemServer.java`：`startApexServices` 之后插入点在 A16 该段重构过，需重新定位（los 的 hook 段在 `SystemServer.java:1034-1037` 附近 `startBootstrap/Core/Other/ApexServices`）。
- `Context.java`：los 现有服务常量在 L7110–7242，需在此区段加 `AOS_SERVICE`；同步 `system-current.txt`（los 仍用该文件）否则 `mmd`/API lint 报错。
- `Instrumentation.java`：`newApplication()` 两重载签名在 A16 可能变。
- `AosHooks.java`：Build 字段欺骗较稳，但 hidden-api policy 判定方式可能调整。

### frameworks/native（fork）— **0 专属改动，纯上游镜像**
分支 HEAD = `e412ad3b Merge 'android-14.0.0_r25'`（LineageOS 官方），全文 grep `arsenals|AosService|AOS_SERVICE` 零命中。**移植直接用上游 23.2 的 frameworks/native，无需任何补丁。**

### bionic（fork）— **0 专属改动，纯上游镜像**
顶部 4 commit 均为已知 LineageOS 贡献者上游补丁（crpalmer/Quallenauge/me-cafebabe/Rashed），无 init 提交，grep 零命中。**移植直接用上游 23.2 的 bionic。**

### system/core（fork，1 commit `15fb1ad`，基线 `fc1df85 init from LineageOS`）
- `init/Android.bp`（+6）：加 `SPOOF_SAFETYNET=1` 编译宏（userdebug 经 product_variables 覆盖为 0）。
- `init/property_service.cpp`（+45）：新增 `SetSafetyNetProps()` 在 `PropertyInit()`、解析 kernel cmdline 前调用，设置 ~25 个只读属性：`ro.boot.flash.locked=1`、`ro.boot.vbmeta.device_state=locked`、`ro.boot.verifiedbootstate=green`、`ro.boot.veritymode=enforcing`、`ro.build.keys=release-keys`、`ro.build.type=user`、`ro.secure=1`、`ro.debuggable=0` 等（含 oplusboot 专属）；`weaken_prop_override_security` 在 `vendor_load_properties()` 期间临时放行属性覆写。受 `SPOOF_SAFETYNET` + `IsRecoveryMode()` 双重保护。

**移植风险（A16）**：`PropertyInit()`/`PropertyLoadBootDefaults()`/`vendor_load_properties()` 签名与调用顺序可能变；`Android.bp` 的 cflags 注入点可能重组；属性清单需补 A14→A16 新增 SafetyNet 检查项。

## 3. AOS 系统服务（`vendor/arsenals/{aos,arsenalsos}`，均新增仓库）

**两个独立 boot jar**，Soong `java_library{ installable:true, dxflags:["--core-library"], srcs:["**/*.java"|"**/*.aidl"] }`，经 `PRODUCT_PACKAGES`+`PRODUCT_BOOT_JARS` 上 boot classpath。vendor 下的 `frameworks/base/...` 路径仅**目录命名约定**，不叠加进 AOSP 源码树。

| 仓库 | 文件 | 职责 |
|---|---|---|
| `vendor/arsenals/aos` | `config/common.mk` | `PRODUCT_PACKAGES += aos` + `PRODUCT_BOOT_JARS += aos`，`-include vendor/arsenals/arsenalsos/config/common.mk`（aos 依赖 arsenalsos） |
| | `frameworks/base/Android.bp` | 构建 `aos` jar |
| | `core/java/cn/arsenals/aos/AosConstants.java` | `AOS_PACKAGE_NAME="cn.arsenals.aos"` |
| | `core/java/cn/arsenals/aos/input/AosInputUtil.java` | 包装 `InputManager.getInstance().injectInputEvent(event,mode)` + 3 个 `INJECT_INPUT_EVENT_MODE_*` 常量 |
| `vendor/arsenals/arsenalsos` | `config/common.mk` | `PRODUCT_PACKAGES += arsenalsos` + `PRODUCT_BOOT_JARS += arsenalsos` |
| | `frameworks/base/Android.bp` | 构建 `arsenalsos` jar（srcs 含 .aidl） |
| | `core/java/cn/arsenals/arsenalsos/IAosService.aidl` | `@hide int getAosVersionNumber()` |
| | `core/java/cn/arsenals/arsenalsos/AosService.java` | `IAosService.Stub` 实现，返回 1 |
| | `core/java/cn/arsenals/arsenalsos/AosServicesManager.java` | `startAosServices()` → `ServiceManager.addService(Context.AOS_SERVICE, new AosService(ctx))` |
| | `core/java/cn/arsenals/arsenalsos/AosManager.java` | 客户端单例，`IAosService.Stub.asInterface(ServiceManager.getService(Context.AOS_SERVICE))` |
| | `core/java/cn/arsenals/arsenalsos/ArsenalsOsConstants.java` | `ARSENALS_OS_PACKAGE_NAME="cn.arsenals.arsenalsos"` |

**移植（A16，已实测 los）**：
- ⚠️ **`InputManager.getInstance()` 在 Android 16 已移除**（A14 `@Deprecated`，A15 TODO soft-remove，A16 删除）→ `AosInputUtil` 必改为 `ctx.getSystemService(InputManager.class).injectInputEvent(event,mode)`（需持 Context 或加参数）。
- `INJECT_INPUT_EVENT_MODE_*` 三常量与 `injectInputEvent(InputEvent,int)` 2 参重载仍在（los `InputManager.java:1047`），常量值不变——只改 `getInstance()` 调用点。
- boot jar 机制（`PRODUCT_BOOT_JARS`+`java_library`+`--core-library`）A16 仍有效，`common.mk`/`Android.bp` 可原样保留；建议核对 hiddenapi/bootclasspath allowlist 是否需登记。
- `Context.AOS_SERVICE` 需在 los 重新 patch（见 §2 frameworks/base）。

## 4. sepolicy

### `device/arsenals/sepolicy`（新增）
`sepolicy.mk` 用 `SYSTEM_EXT_PUBLIC_SEPOLICY_DIRS`(public) + `SYSTEM_EXT_PRIVATE_SEPOLICY_DIRS`(private/dynamic/system) + `BOARD_VENDOR_SEPOLICY_DIRS`(vendor) 接入。`default.te` 全空占位，3 个 `platform_app.te` 有内容：
- `private/platform_app.te`：`allow platform_app proc_stat:file { read open getattr };`
- `system/platform_app.te`：读 `sysfs_graphics`/`sysfs_battery_supply`/`sysfs_kgsl`/`sysfs_kgsl_gpuclk`（system 域类型）
- `vendor/platform_app.te`：同上但 `vendor_` 前缀类型（对应源码编译 vendor）

### `device/lineage/sepolicy` fork（3 专属 commit by Y-D-Lu）
1. `a37d305` `include arsenals sepolicy rules` — `common/sepolicy.mk` 末尾 `-include device/arsenals/sepolicy/sepolicy.mk`
2. `ff9318d` Revert "Remove legacy camera HAL1 sepolicy"（保旧 camera HAL1）
3. `151f52b` Revert "qcom: Drop support for ultra legacy platforms"（保 ultra-legacy qcom）

### `device/qcom/sepolicy_vndr/sm8450` fork（1 专属 commit `6c8d7641`）
`fix sepolicy avc: denied for hal_health` — 新增 `qva/vendor/common/hal_health.te`（`allow hal_health vendor_sysfs_battery_supply/vendor_sysfs_usb_supply`）+ `file_contexts` 标 `android.hardware.health-service.qti`。

**移植（A16）**：接入机制（`SYSTEM_EXT_*`/`BOARD_VENDOR_*`）未变，`.te` 可保留；需复核 `allow platform_app proc_stat/sysfs_*` 在 A16 的 neverallow（proc 域收紧）；`sysfs_kgsl*` 等类型须在目标 vendor sepolicy 存在；sm8450 的 `hal_health` fix 依赖 `vendor_sysfs_battery_supply/usb_supply`，新平台需重新对齐。

## 5. vendor/lineage fork（品牌，5 专属 commit by Y-D-Lu，fork 点 `8e49671f`）

| commit | 主题 |
|---|---|
| `e0a681af` | add vendor_arsenals_aos to common.mk |
| `34b0912c` | hide los props by changing name to arsenals' |
| `b6aa198c` | 不创建 addon.d 规避第三方 ROM 检测 |
| `d51e8e55` | build official by default |
| `583dfba9` | merge_dtbs: Respect miboard-id |

**7 改动文件**：
| 文件 | 改动 |
|---|---|
| `build/core/main_version.mk` | `ro.lineage.{version,releasetype,build.version,display.version,build.version.plat.sdk,build.version.plat.rev}` → `ro.arsenals.*`；`ro.lineagelegal.url` → `ro.arsenalslegal.url`（URL 仍 lineageos.org/legal） |
| `build/envsetup.sh` | `eat()` glob `lineage-*.zip` → `arsenals-*.zip` |
| `build/tasks/bacon.mk` | `LINEAGE_TARGET_PACKAGE := $(PRODUCT_OUT)/arsenals-$(LINEAGE_VERSION).zip` |
| `build/tasks/kernel.mk` | 删 "PREBUILT KERNEL NOT ALLOWED ON OFFICIAL" 逻辑（允许官方构建用预编译内核，配合 KernelSU） |
| `build/tools/merge_dtbs.py` | 新增 `xiaomi,miboard-id` 贯穿 DeviceTreeInfo（小米多板 DTB 合并） |
| `config/common.mk` | 删 `50-lineage.sh`→`addon.d` 的 COPY_FILES（但 `PRODUCT_ARTIFACT_PATH_REQUIREMENT_ALLOWED_LIST` 保留条目）；末尾 `include vendor/arsenals/aos/config/common.mk` |
| `config/version.mk` | 强制 `LINEAGE_BUILDTYPE := OFFICIAL` |

> `common_full_phone.mk`/`common_full.mk` 无 arsenals 改动；bootanimation 仍 `vendor/lineage/bootanimation/`（`bootanimation.tar` 5.9MB + `desc.txt`，未改名）。

**移植（A16/23.2）**：23.2 vendor/lineage 仍是上游，需重做上述 5 处 rebrand；注意 23.2 新增 `release/`+`vars/` 目录、`config/version.mk` 改 `23/2`、build 可能改用 Soong 模块化打包 bootanimation；`merge_dtbs.py` 路径可能移到 `vendor/lineage/build/tools/`。

## 6. build/make fork（5 专属 commit）

| commit | 主题 |
|---|---|
| `3c03dfec99` | use release key by default |
| `ac16544c59` | fix version code for userdebug |
| `3a1810c993` | move keys in security into relative path folder besides aos |
| `d3a8fac2dd` | hide los props by changing name to arsenals' |
| `f30f7aa7f2` | use default fsverity-release.x509.der |

**19 改动文件（要点）**：
| 文件 | 改动 |
|---|---|
| `core/config.mk`（L6） | `PRODUCT_DEFAULT_DEV_CERTIFICATE := build/make/target/product/security/releasekey` |
| `core/sysprop.mk` | 删 testkey/dev-keys 判定，**恒定 `BUILD_KEYS := release-keys`** |
| `core/version_util.mk` | `BUILD_NUMBER := eng.$(BUILD_USERNAME[:6]).date` → `$(BUILD_DATETIME)`（userdebug 版本号修复） |
| `envsetup.sh` | `check_product()` 新增 `^arsenals_` 前缀分支（`LINEAGE_BUILD=$(sed 's/^arsenals_//')`） |
| `tools/buildinfo.sh` | `ro.lineage.device` → `ro.arsenals.device` |
| `target/product/security/{bluetooth,cts_uicc_2021,media,networkstack,platform,sdk_sandbox,shared}.{pk8,x509.pem}` | AOSP testkey 派生证书**删除→符号链接** `../../../../../../keys/<name>.*` |
| `target/product/security/fsverity-release.x509.der` | 先 symlink 后改回真实文件（1484B，用 AOSP 默认 fsverity 证书，不自签） |

> build/make 带 5 个 linkfile（CleanSpec/buildspec/core/envsetup/target/tools → `build/`），故 `build/envsetup.sh` → `build/make/envsetup.sh`。

**移植（A16/23.2）**：23.2 build 拆为 `build/make`+`build/release`+`build/soong`+`build/blueprint`（21.0 只有 build/make）；fsverity/release 逻辑多在 `build/release`，fork 改动需重新定位落点；`BUILD_KEYS:=release-keys` 硬编码 hack 在 23.2 宜改用 release config flag（`bp4a`）；`PRODUCT_DEFAULT_DEV_CERTIFICATE` 在 23.2 的 `build/make/core/config.mk` 仍可加，但需确认是否移到 `build/release`。

## 7. 签名体系

- 自签 releasekey 体系（非 AOSP testkey、非 LOS dev-keys）。
- 证书外置：**`/root/arsenals/aos/keys/`**（源码树 `aos/` 同级目录，无 git）存全部真实私钥：`releasekey`/`platform`/`shared`/`media`/`networkstack`/`bluetooth`/`sdk_sandbox`/`cts_uicc_2021`/`testkey`（各 `.pk8`+`.x509.pem`）+ `fsverity-release.x509.der`。
- `build/make/target/product/security/` 的 7 套 key 改 symlink → `keys/`；`testkey.*` 与 `fsverity-release.x509.der` 保留真实文件。
- `PRODUCT_DEFAULT_DEV_CERTIFICATE := .../security/releasekey` → `DEFAULT_SYSTEM_DEV_CERTIFICATE`；`otacerts` 用其 `.x509.pem` 打包 OTA 验签 keystore；`BUILD_KEYS := release-keys`。

> ⚠️ **gap（移植必查）**：`security/` 里 7 套 key 都 symlink 了，**唯独漏了 `releasekey`**（`PRODUCT_DEFAULT_DEV_CERTIFICATE` 指向它，真实文件只在 `keys/`）。21.0 构建能出包说明主机手动补过该 symlink 未入库。移植 23.2 时**务必显式建** `build/make/target/product/security/releasekey.{pk8,x509.pem} -> ../../../../../../keys/releasekey.*`，否则签名找不到证书。

## 8. 设备树 / kernel / blob / gapps（手动 clone 为主）

### 8.1 marble 产品构建链
```
arsenals_marble.mk (device/xiaomi/marble/)
├─ device/xiaomi/marble/device.mk          (NFC SKU/init.marble.rc/overlay/Soong ns/gapps)
│  ├─ device/xiaomi/sm8450-common/common.mk (HAL/分区/fstab/overlay/sepolicy)
│  │  ├─ $(SRC_TARGET_DIR)/product/{core_64_bit,full_base_telephony,emulated_storage,generic_ramdisk,updatable_apex}.mk
│  │  ├─ .../virtual_ab_ota/launch_with_vendor_ramdisk.mk   (Virtual A/B + vendor ramdisk)
│  │  ├─ frameworks/native/build/phone-xhdpi-6144-dalvik-heap.mk
│  │  └─ vendor/xiaomi/sm8450-common/sm8450-common-vendor.mk (通用 blob)
│  ├─ vendor/xiaomi/marble/marble-vendor.mk (marble 专用 blob: camera/audio/sensor/firmware)
│  └─ vendor/gapps/arm64/arm64-vendor.mk   (MindTheGapps → common/common-vendor.mk)
└─ vendor/lineage/config/common_full_phone.mk  (TARGET_DISABLE_EPPE := true)
```
BoardConfig 链：`marble/BoardConfig.mk` → `sm8450-common/BoardConfigCommon.mk` → `vendor/xiaomi/sm8450-common/BoardConfigVendor.mk`(空) + `vendor/xiaomi/marble/BoardConfigVendor.mk`(仅 `AB_OTA_PARTITIONS += 20 固件分区`)。

身份：`PRODUCT_NAME=arsenals_marble`、`PRODUCT_DEVICE=marble`、Redmi `23049RAD8C`、`PRODUCT_GMS_CLIENTID_BASE=android-xiaomi`；build fingerprint = `Redmi/marble/marble:12/SKQ1.230401.001/V816.0.2.0.UMRCNXM`（**基线 Android 12**）。

### 8.2 BoardConfig 关键项
- **分区/Virtual-AB**：`AB_OTA_UPDATER:=true`；`AB_OTA_PARTITIONS += boot dtbo odm product recovery system system_ext vbmeta vbmeta_system vendor vendor_boot vendor_dlkm`；动态组 `qti_dynamic_partitions`(odm product system system_ext vendor vendor_dlkm)；`BOARD_SUPER_PARTITION_SIZE := 9126805504`。
- **AVB**：`BOARD_AVB_ENABLE:=true`，`BOARD_AVB_MAKE_VBMETA_IMAGE_ARGS += --flags 3`（**关 verity，配合 KernelSU**）；`BOARD_MOVE_GSI_AVB_KEYS_TO_VENDOR_BOOT:=true`。
- **Kernel GKI 5.10**：`BOARD_USES_GENERIC_KERNEL_IMAGE:=true`、`BOARD_INCLUDE_DTB_IN_BOOTIMG:=true`、`BOARD_RAMDISK_USE_LZ4:=true`、`TARGET_KERNEL_SOURCE := kernel/xiaomi/sm8450`（Linux 5.10.205）、`TARGET_KERNEL_CONFIG := gki_defconfig vendor/waipio_GKI.config vendor/xiaomi_GKI.config vendor/marble_GKI.config`；`BOARD_BOOT_HEADER_VERSION:=4`、`BOARD_KERNEL_IMAGE_NAME:=Image`。
- 外部模块树 `TARGET_KERNEL_EXT_MODULE_ROOT := kernel/xiaomi/sm8450-modules`，13 个 `qcom/opensource/*`（audio/camera/cvp/dataipa/datarmnet(-ext)/display/eva/mmrm/video/wlan）；DLKM 三段：`first_stage`(进 vendor_ramdisk)、`second_stage`、`vendor_dlkm_exclusive`；`BOARD_VENDOR_RAMDISK_FRAGMENTS := dlkm`；blocklist `modules.vendor_blocklist.msm.waipio`。
- marble 专属：`BOOT_KERNEL_MODULES`（qcom_pm8008-regulator/gt9916r触控/qcom-hv-haptics/fpc1540/goodix_3626指纹）、`TARGET_SCREEN_DENSITY:=440`、`TARGET_QTI_VIBRATOR_EFFECT_LIB:=libqtivibratoreffect.xiaomi`。
- 平台 `TARGET_BOARD_PLATFORM := taro`（SM8450）；`PRODUCT_SHIPPING_API_LEVEL/BOARD_API_LEVEL := 31`（Android 12 shipping）。

### 8.3 overlay + CN 变体
- marble 8 个 overlay（均 `device_specific`）：`ApertureResMarble`/`FrameworksResMarble`(含 power_profile)/`SettingsResMarble`/`SettingsProviderResMarble`/`SystemUIResMarble`/`WifiResMarble` + **CN 变体** `SettingsProviderResMarbleCN`/`WifiResMarbleCN`。
- CN 机制：CN 变体 AndroidManifest 用 `android:requiredSystemPropertyName="ro.boot.hwc" android:requiredSystemPropertyValue="CN"`（运行时条件 overlay）。基础变体="POCO F5"，CN 变体="Redmi Note 12 Turbo"。`init.marble.rc` 有 `on property:ro.boot.hwc=CN setprop bluetooth.device.default_name "Redmi Note 12 Turbo"`。
- sm8450-common overlay 通用 Sys + Xiaomi/Target（含大量 `values-mcc*-mnc*` 运营商定制）。
- NFC：`TARGET_NFC_SUPPORTED_SKUS := marble`；`DEVICE_MANIFEST_SKUS := taro diwali cape ukee`（marble 属 ukee 体系）。

### 8.4 blob / gapps
- `vendor/xiaomi/{marble,sm8450-common}/`：`proprietary/` + 自动生成的 `Android.bp`/`Android.mk`/`BoardConfigVendor.mk`/`*-vendor.mk`（头部标注 `generated by setup-makefiles.sh`）。
- 生成链：`device/xiaomi/marble/extract-files.sh`（`blob_fixup()` 4 处修补：camera motiontuning XML、pureView XML、`init.batterysecret.rc`、`citsensorservice` 二进制符号 `ClientImplGetE→ClientImpl4InitE` 重命名）→ 委托 `sm8450-common/extract-files.sh` → `setup-makefiles.sh`。
- `proprietary-files.txt`：marble 1084 行（源自 marble `V816.0.3.0.UMRMIXM`，含 ACDB/ADSP/audio/camera[Arcsoft/Mialgo/Vidhance]/fingerprint[fpc/goodix]/sensors/thermal/touch firmware/vibrator RTP/VPU/Adreno/EVAS firmware）；sm8450-common 1758 行（源自 `unicorn`(Xiaomi 13 Pro 同平台) `V816.0.3.0.ULECNXM`）。
- `proprietary-firmware.txt`：20 个 `;AB` 固件分区（abl/aop/bluetooth/cpucp/devcfg/dsp/featenabler/hyp/imagefv/keymaster/modem/qupfw/shrm/tz/uefi/uefisecapp/xbl/xbl_config/xbl_ramdump）经 `BoardConfigVendor.mk` 的 `AB_OTA_PARTITIONS` 接入 OTA。
- 接入：`marble-vendor.mk` 用 `PRODUCT_COPY_FILES` + `PRODUCT_SOONG_NAMESPACES += vendor/xiaomi/marble`。
- `vendor/gapps`（MindTheGapps）：`{arm,arm64,x86,x86_64,common,build,overlay}`；接入点 `device/xiaomi/marble/device.mk` 末尾 `inherit-product vendor/gapps/arm64/arm64-vendor.mk`；APK 走 `Android.bp` `android_app_import{presigned:true, dex_preopt.enabled:false}`，分类 product/system_ext/privileged；可 `make gapps_arm64_` 打可刷 zip。

### 8.5 KernelSU 接入
- 仓库 `kernel/arsenals/kernelsu`（`snippets/aos.xml` 引入），`kernel/` 子目录（GPL）：`ksu.c/allowlist.c/apk_sign.c/sucompat.c/throne_tracker.c/core_hook.c/ksud.c/embed_ksud.c/kernel_compat.c` + `selinux/{selinux.c,sepolicy.c,rules.c}`。致谢 tiann/KernelSU。
- **接入内核构建**（已落地）：① 软链 `kernel/xiaomi/sm8450/drivers/kernelsu → ../../../arsenals/kernelsu/kernel`；② `drivers/Makefile` `obj-y += kernelsu/`（built-in，非模块）；③ `drivers/Kconfig` `source "drivers/kernelsu/Kconfig"`；④ `kernel/Kconfig` `config KSU tristate ... depends on OVERLAY_FS default y`——依赖 `CONFIG_OVERLAY_FS=y`，而 `gki_defconfig` 已开，故 `CONFIG_KSU` 默认启用。
- 版本：`KSU_VERSION = 10000 + git rev-list --count HEAD + 200`；内嵌 manager 签名 size/hash 做 apk 校验。
- ⚠️ `setup.sh` 仍写死 `git clone https://github.com/tiann/KernelSU`——若 re-run 会拉上游而非 ArsenalsOS fork，需改为 fork 或离线拷贝。

## 9. 构建命令对比

**21.0（当前 aos）**：
```
cd /root/arsenals/aos/aos
. build/envsetup.sh
breakfast marble userdebug     # = lunch arsenals_marble-userdebug（两段式）
mka bacon                       # bacon target → out/target/product/marble/arsenals-21.0-<date>-OFFICIAL-marble.zip
eat                             # adb sideload arsenals-*.zip
```
版本串：`LINEAGE_VERSION = 21.0-<YYYYMMDD>-OFFICIAL-marble`（强制 OFFICIAL）。

**23.2 目标（los，需移植到位后）**：
```
cd /root/arsenals/aos/los
. build/envsetup.sh
breakfast marble userdebug       # 23.2 breakfast 先 source vars/aosp_target_release, 再 lunch
# = lunch arsenals_marble-bp4a-userdebug  （三段式 PRODUCT-RELEASE-variant）
mka bacon                        # 产物 arsenals-23.2-<date>-OFFICIAL-marble.zip
```
> 23.2 的 `breakfast`/`check_product` 在 `vendor/lineage/build/envsetup.sh`，**只识别 `^lineage_`**——需加 `^arsenals_` 分支，并把 `lunch lineage_$target-...` 改 `arsenals_$target-...`，或 device 的 `AndroidProducts.mk` 用 `arsenals_<dev>` product 名。

## 10. 移植工作清单（按风险/优先级）

### P0 — 阻塞编译的必改项
1. **manifest remote**：23.2 `fetch=".."` 模型下，aos fork project 不能继承默认 remote → 在 local_manifest/snippet 显式加 `fetch="https://github.com"` 的 remote（如 `name="arsenals"`），所有 `ArsenalsOs/aos_*` project 用它；revision 改基于 `lineage-23.2` 的新 `aos` 分支。
2. **check_product `^arsenals_`**：23.2 `vendor/lineage/build/envsetup.sh` 的 `check_product` + `breakfast` 加 arsenals 识别；lunch 三段式 `arsenals_marble-bp4a-userdebug`。
3. **`InputManager.getInstance()`**：`vendor/arsenals/aos/.../AosInputUtil.java` 改为 `ctx.getSystemService(InputManager.class)`（A16 已删 `getInstance()`）。
4. **releasekey symlink gap**：显式建 `build/make/target/product/security/releasekey.{pk8,x509.pem} -> keys/releasekey.*`；`keys/` 目录用 `~/.arsenals-certs/`（subject `/C=CN/ST=Shaanxi/L=Baoji/O=Arsenals/OU=luyuedong/CN=arsenals/emailAddress=460999218@qq.com`，`development/tools/make_key` 生成 5 套）或同级 `keys/`。
5. **`Context.AOS_SERVICE` + `system-current.txt`**：los `Context.java`(L7110–7242 区段) 加 `@SystemApi AOS_SERVICE="aos"` + 同步 `system-current.txt`。
6. **`SystemServer.startAosServices(t)`**：los `SystemServer.java` 在 `startApexServices` 后插入反射 hook（代码见 §2）。

### P1 — 高风险功能项
7. **`AndroidKeyStoreSpi` 证书篡改**：keystore2 在 A15/A16 大变，42 字节匹配/offset 逻辑极可能失效，需按 A16 证书编码重写（最高风险）。
8. **`system/core` init 属性欺骗**：`property_service.cpp` 的 `SetSafetyNetProps()` + `SPOOF_SAFETYNET` 宏重新定位到 A16 的 `PropertyInit` 调用链；属性清单补 A14→A16 新增项。
9. **内核 5.10 → 6.6/6.12（最大工程）**：重写 `kernel/xiaomi/sm8450` 到 GKI 6.x；重做 `gki_defconfig`+`vendor/*_GKI.config`；13 个 out-of-tree `qcom/opensource/*` 驱动对齐 6.x ABI（`class_create` 签名、`folio` API 等）；`vendor_dlkm` 三段模块符号版本与 6.x Image 重新生成；KernelSU `core_hook.c`/`sucompat.c`/`selinux/` 适配 6.x。
10. **HIDL → AIDL**：common.mk 大量 `android.hardware.*@*-vendor` HIDL（audio@7.0/bluetooth@1.0/camera@2.7/drm@1.4/gnss@2.1/keymaster@4.1/nfc@1.2/sensors@2.1/thermal@2.0/usb@1.3/vibrator 等）A16 强制 AIDL 化，需迁或确认兼容。
11. **blob 重 extract**：基线 Android 12（API 31），依赖 framework lib 符号的 .so（libc++_shared/libcrypto/libprotobuf）需匹配 A16 vndk；`libprotobuf-cpp-*-3.9.1-vendorcompat` 等 A16 大概率移除；建议从 Android 15/16 的 marble/unicorn 官固件重 `extract-files.sh` 拉新 blob 更新 hash pinning。

### P2 — 品牌/机制重做
12. **vendor/lineage 5 处 rebrand**：`main_version.mk`(ro.arsenals.*)、`bacon.mk`(arsenals- zip)、`envsetup.sh eat()`(arsenals-*.zip)、`config/common.mk`(删 50-lineage.sh + include vendor/arsenals/aos)、`config/version.mk`(强制 OFFICIAL)、`buildinfo.sh`(ro.arsenals.device)；23.2 注意路径可能移到 `vendor/lineage/build/` 或 `build/release`。
13. **build/make 签名改动**：23.2 build 拆 `make`+`release`+`soong`+`blueprint`；`PRODUCT_DEFAULT_DEV_CERTIFICATE`/`BUILD_KEYS` 落点重新定位；考虑用 release config flag 替代硬编码。
14. **`merge_dtbs.py` miboard-id**：cherry-pick 到 23.2 的 `vendor/lineage/build/tools/merge_dtbs.py`。
15. **`device/lineage/sepolicy` fork 3 commit**：接入 arsenals rules + 两处 Revert（legacy camera HAL1 / ultra-legacy qcom）重新 cherry-pick。
16. **sepolicy 类型对齐**：`device/arsenals/sepolicy` 的 `sysfs_kgsl*`/`vendor_sysfs_battery_supply` 等在目标 vendor sepolicy 确认存在；sm8450 `hal_health` fix 的依赖类型对齐新平台。

### P3 — 设备/资源
17. **CN 变体**：确认 bootloader（abl/uefi）仍传 `ro.boot.hwc=CN`；`SettingsProviderResMarbleCN`/`WifiResMarbleCN` 的 `requiredSystemPropertyValue` 机制 A16 RRO 仍支持。
18. **gapps**：MindTheGapps 机制（`android_app_import` presigned）A16 可用；GMS APK 换 A16 兼容版本（Velvet/GmsCore targetSdk 34+）。
19. **QSSI/多 SKU**：`DEVICE_MANIFEST_SKUS := taro diwali cape ukee` + `hardware/qcom-caf/common/vendor_framework_compatibility_matrix.xml` 随 qcom-caf 升级。
20. **AVB**：`--flags 3`（关 verity）对 KernelSU 必要，保留；`BOARD_MOVE_GSI_AVB_KEYS_TO_VENDOR_BOOT` 复核 A16 行为。

### 可直接免移植（重要简化）
- `frameworks/native`、`bionic`：纯上游镜像，**直接用 23.2 上游，不带任何补丁**。aos.xml 里这两个 fork 可在 23.2 manifest 中去掉（或保留为上游同名 project）。

## 11. 关键文件路径速查（源 aos 树）
- manifest：`.repo/manifests/{default.xml,snippets/aos.xml,snippets/lineage.xml}`
- framework：`frameworks/base/{core/java/android/content/Context.java(L6440),core/api/system-current.txt(L3448),services/java/com/android/server/SystemServer.java(L951,L3250),core/java/com/android/internal/util/aos/AosHooks.java,keystore/java/android/security/keystore2/AndroidKeyStoreSpi.java,core/java/android/app/Instrumentation.java}`
- system/core：`init/{Android.bp,property_service.cpp}`
- AOS 服务：`vendor/arsenals/{aos,arsenalsos}/{config/common.mk,frameworks/base/Android.bp,frameworks/base/core/java/cn/arsenals/**,frameworks/base/services/core/java/cn/arsenals/arsenalsos/AosService.java}`
- sepolicy：`device/arsenals/sepolicy/{sepolicy.mk,**/*.te}`；`device/lineage/sepolicy/common/sepolicy.mk`；`device/qcom/sepolicy_vndr/sm8450/qva/vendor/common/hal_health.te`
- 品牌：`vendor/lineage/build/{core/main_version.mk,tasks/bacon.mk,tasks/kernel.mk,tools/merge_dtbs.py,envsetup.sh}`、`vendor/lineage/config/{common.mk,version.mk}`
- build：`build/make/{core/config.mk(L6),core/sysprop.mk,core/version_util.mk,envsetup.sh,tools/buildinfo.sh,target/product/security/}`
- 签名：`/root/arsenals/aos/keys/`（同级，无 git）
- 设备：`device/xiaomi/marble/{arsenals_marble.mk,device.mk,BoardConfig.mk,AndroidProducts.mk,extract-files.sh,setup-makefiles.sh,proprietary-files.txt,proprietary-firmware.txt,properties/{system,vendor}.prop,rootdir/etc/init.marble.rc,overlay/*/Android.bp}`、`device/xiaomi/sm8450-common/{common.mk,BoardConfigCommon.mk,proprietary-files.txt,modules.list.{second_stage,vendor_dlkm}}`
- blob/gapps：`vendor/xiaomi/{marble/{marble-vendor.mk,BoardConfigVendor.mk},sm8450-common/sm8450-common-vendor.mk}`、`vendor/gapps/{arm64/arm64-vendor.mk,common/common-vendor.mk}`
- kernel：`kernel/arsenals/kernelsu/kernel/{Kconfig,Makefile,setup.sh}`、`kernel/xiaomi/sm8450/{Makefile,drivers/{Makefile,Kconfig},arch/arm64/configs/gki_defconfig}`、`kernel/xiaomi/sm8450-modules/`
