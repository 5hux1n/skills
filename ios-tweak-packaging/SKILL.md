---
name: ios-tweak-packaging
version: 1.1.0
description: >-
  构建、打包、验证、排查 iOS 越狱插件（Theos / Logos）时使用，尤其是 roothide（RootHide）
  与巨魔 / TrollStore 注入场景。
  覆盖 rootless 与 roothide 双方案构建命令、roothide 专属的 rpath / jbroot / 包名 /
  LaunchDaemon plist 坑、巨魔注入的弱链接与 Substrate 缺失处理、工具链坑（LC_ALL=C、
  class-dump 失效、UTF-16 字符串）、Logos 代码坑（构造器安全模式、%ctor 里不要碰目标 App 的类、
  接口声明签名必须精确、Swift 类 hook 不生效），以及交付前验证清单和「症状 → 病因」速查表。
  Use when building, packaging, debugging, or triaging launch crashes of a Theos/Logos tweak,
  including TrollStore / IPA injection.
---

# iOS 越狱插件：构建、打包、验证

## 一句话原则

**打包方案（rootless / roothide）决定三件事：安装前缀、Architecture、rpath。**
**三者必须自洽，否则轻则插件不生效，重则目标 App 启动即崩、甚至开机异常。**

## 0. 先确认目标环境

| 环境 | 方案 | 安装前缀 | Architecture | Theos |
|---|---|---|---|---|
| Dopamine / palera1n（传统无根） | `rootless` | `/var/jb` | `iphoneos-arm64` | `~/theos` |
| **RootHide / roothide** | `roothide` | **空** | **`iphoneos-arm64e`** | `~/theos-roothide` |

**roothide 必须原生构建。** 把 rootless 布局的包标成 arm64e 交付，RootHide 的 Patcher 会跳过转换，
结果是插件不生效甚至重启异常。**不要靠 Patcher 转换，自己用 `THEOS_PACKAGE_SCHEME=roothide` 编。**

## 1. 构建命令与 Makefile 骨架

```bash
export LC_ALL=C                                   # 见 §4.1，路径含中文时必须

# rootless
export THEOS=~/theos SYSROOT=~/theos/sdks/iPhoneOS16.5.sdk
make clean && make package

# roothide（原生）
export THEOS=~/theos-roothide SYSROOT=~/theos-roothide/sdks/iPhoneOS16.5.sdk
make clean && make package THEOS_PACKAGE_SCHEME=roothide
```

Makefile 关键片段：

```make
ARCHS = arm64 arm64e                 # 双架构 fat dylib
THEOS_PACKAGE_SCHEME ?= rootless     # 用 ?= 才能被命令行覆盖

BASE_PACKAGE = com.example.mytweak
ifeq ($(THEOS_PACKAGE_SCHEME),roothide)
JBROOT_PREFIX :=
else
JBROOT_PREFIX := /var/jb
endif

include $(THEOS)/makefiles/common.mk
# ... TWEAK_NAME / FILES / FRAMEWORKS ...
include $(THEOS_MAKE_PATH)/tweak.mk

# roothide 必须手动补 rpath（见 §2.1）
ifeq ($(THEOS_PACKAGE_SCHEME),roothide)
MyTweak_LDFLAGS += -Wl,-rpath,@loader_path/.jbroot/Library/Frameworks \
                   -Wl,-rpath,@loader_path/.jbroot/usr/lib
endif

before-package::
	@sed -i '' 's/^Version: .*/Version: $(VERSION)/' $(THEOS_STAGING_DIR)/DEBIAN/control
	@# 路径占位符按方案注入，别在源码/脚本里写死 /var/jb
	@for f in postinst prerm postrm; do \
		sed -i '' 's|@JBROOT@|$(JBROOT_PREFIX)|g' $(THEOS_STAGING_DIR)/DEBIAN/$$f 2>/dev/null; done; true
ifeq ($(THEOS_PACKAGE_SCHEME),roothide)
	@# 包名自洽（见 §2.4）
	@sed -i '' 's/^Package: .*/Package: $(BASE_PACKAGE).roothide/' $(THEOS_STAGING_DIR)/DEBIAN/control
endif
```

## 2. roothide 四个专属坑
### 2.1 缺 rpath → 目标 App 启动即崩（最坑）

theos **只给 rootless 方案自动加 rpath**。而 Logos / Orion 生成的 hook 代码会**弱链接**
`@rpath/CydiaSubstrate.framework/CydiaSubstrate`。缺 rpath → 弱符号解析成 NULL →
`MSHookFunction` / `MSHookMessageEx` 调用跳到地址 0 → 所有配了插件的 App 一启动就闪退。

崩溃特征：`pc=0`，而 `x0` 是某个被 hook 的 C 函数名字符串。

两处都要做：

1. 加两条 rpath：`@loader_path/.jbroot/Library/Frameworks` 与 `@loader_path/.jbroot/usr/lib`
2. 调 C hook 前判空兜底：`if (!MSHookFunction || !MSHookMessageEx) return;`

### 2.2 路径用 `jbroot()`，不要硬编码 `/var/jb`

```objc
#if __has_include(<roothide.h>)
  #import <roothide.h>
  #define JBP(p) [@(jbroot((p).UTF8String)) stringByAppendingPathComponent:(p)]
#else
  #define JBP(p) [@"/var/jb" stringByAppendingPathComponent:(p)]
#endif
```

DEBIAN 脚本与 LaunchDaemon plist 里的路径写占位符（`@JBROOT@`），打包时注入。**源码里不要出现 `/var/jb` 字面量。**

### 2.3 LaunchDaemon plist 必须是 XML / 二进制

roothide 的 Patcher 只认 XML/binary plist，**不认 NeXTSTEP 文本格式**。
若在 `before-package` 里把 plist 覆盖回文本格式，守护进程不会启动。

### 2.4 包名自洽：本地 deb 与仓库名要一致

roothide 变体在仓库里通常叫 `<base>.roothide`。本地 deb 内部仍是 `Package: <base>` 的话，
`dpkg -i` 覆盖安装会报：

```
trying to overwrite '/Library/LaunchDaemons/xxx.plist', which is also in package xxx.roothide
```

解法：在 `before-package` 里按方案改写 `Package:`（见 §1 代码），或先卸载旧包。

**反向也要注意**：改了包名之后，仓库生成器里靠 `internal.package != bundle_id` 判断的逻辑会失效
（例如「给 roothide 变体加 (Roothide) 后缀」那段），要改成按 section / bundle_id 判断。

## 2.5 巨魔 / TrollStore 注入（不走越狱注入）—— 环境完全不同

如果插件是通过「巨魔注入器」把 dylib 塞进 `.app` 包里的（`<App>.app/Frameworks/xxx.dylib`
+ 主程序加一条 `LC_LOAD_DYLIB`），运行环境和越狱注入**不一样**：

| | 越狱注入 | 巨魔 / IPA 注入 |
|---|---|---|
| dylib 位置 | jbroot 的 `DynamicLibraries/` | `<App>.app/Frameworks/` |
| `@loader_path/.jbroot/...` rpath | 能解析（越狱会在每个含 mach-o 的目录放 `.jbroot` 软链） | **解析不到**，那里没有 `.jbroot` |
| Substrate | 有 | 可能没有 |

所以：

1. **CydiaSubstrate 一律弱链接**：`-Wl,-weak_framework,CydiaSubstrate`。
   强链接时 dyld 解析不到就会直接失败。
2. **`%ctor` 里判空兜底**，为 NULL 就跳过 `%init`（插件不生效，但 App 不会崩）：

```objc
if (dlsym(RTLD_DEFAULT, "MSHookMessageEx") && dlsym(RTLD_DEFAULT, "MSHookFunction")) {
    %init(MyHooks);
} else {
    NSLog(@"[Tweak] 当前进程没有 CydiaSubstrate，跳过 hook 安装");
}
```

3. **注入的切片要和 App 主程序一致**。App Store 下的 App 主程序是 **arm64**（不是 arm64e），
   注入器如果挑了 arm64e 切片，属于架构错配。

> ⚠️ 副作用提醒：一旦弱链接 + 判空，**在巨魔注入环境里插件会静默不生效**。
> 如果必须让它在无 Substrate 环境也能干活，得换 fishhook/Orion 这类不依赖 Substrate 的方案。

## 3. Logos / hook 代码坑

### 3.1 `_logosLocalInit` 无条件构造器 → SpringBoard 无限安全模式

Logos 会给每个 `%hook` 生成独立构造器，在 dyld 初始化阶段就 `objc_getClass` 几十个类；
任何一个为 nil 且未判空就崩。**把所有 `%hook` 包进 `%group`，用一个 `%ctor` 门控 `%init`：**

```objc
%group MyHooks
%hook SomeClass
- (void)foo { %orig; }
%end
%end

static BOOL ShouldInstall(void) { /* 只对目标 App 返回 YES */ }

%ctor { if (ShouldInstall()) { %init(MyHooks); } }
```

验证：`nm dylib | grep _logosLocalInit` 应为 **0**。

### 3.2 `%hook` 的方法签名必须精确，类型错了会崩

Logos 靠 `@interface` 声明知道方法存在，**声明里的类型必须与真实签名一致**。
把 `double` / `long long` 写成 `id` 再用 `%@` 打日志，会把数值当指针解引用 → 闪退。

拿签名的办法：目标二进制导出的头文件，或 `otool -oV`（§4.2）。
**写 hook 前先确认这三个：返回值类型、每个参数类型、属性是对象还是标量。**

### 3.3 不要盲 hook Swift 类

Swift 类方法走 vtable 直接派发，`method_setImplementation` 可能完全不生效。
优先 hook **ObjC 类**；必须动 Swift 类时，先确认调用方是 ObjC（走 `objc_msgSend`）。

Swift 类的 ObjC 运行时名是 mangled 形式（如 `_TtC6FishAd18FishSplashAdConfig`），
可以用 `NSClassFromString` 拿到。

### 3.4 门控之后再做重活

C 函数 hook（fishhook / MSHookFunction）、`notify` 监听、文件 IO 都要放在
「确认这是目标进程」之后，避免在 SpringBoard 等关键进程里做无谓工作。

### 3.5 `%ctor` 里绝对不要碰目标 App 自己的类（血泪）

dyld 初始化阶段，目标 App 的类型系统、容器路径、第三方库**都还没就绪**。
在 `%ctor` 里调用它的类（**尤其 Swift 类**）可能触发一整条初始化链，比如 MMKV 这类
需要 App 容器就绪的库，直接解引用空指针闪退。

崩溃特征（一眼认出）：

- `procLaunch` 之后一两百毫秒就崩
- 栈顶是 `_dispatch_once_callout` → 某个库的 init → 崩
- `EXC_BAD_ACCESS`，`far` 是个很小的值（如 `0x8`）

**正确做法**：`%ctor` 只做 bundleId 判断 + 注册通知；
真正的初始化推迟到 `UIApplicationDidFinishLaunchingNotification` /
`UIApplicationDidBecomeActiveNotification` 之后（首次再 `dispatch_after` 缓 1 秒更稳）。

## 4. 工具链坑

### 4.1 中文路径 + GNU Make 3.81

macOS 自带 make 是 3.81，工程路径含中文时会报 `No rule to make target '/Volumes/代...'`。
**`export LC_ALL=C`** 可解，或把工程放到纯 ASCII 路径下构建。

### 4.2 class-dump 3.5 对现代二进制直接失效

```
class-dump: Unknown load command: 0x00000032      # LC_DYLD_CHAINED_FIXUPS
```

输出会是一句 "This file does not contain any Objective-C runtime information."。

替代方案：**`otool -oV`**（认识 chained fixups），流式过滤后自己解析「类 → 方法」：

```bash
otool -oV <binary> | grep -E "^[0-9a-f]{16} 0x[0-9a-f]+ _OBJC_CLASS_\\\$_|^ *name |^ *imp "
#          类标记行 ↑                ↑ indent 8 = 类名, indent 12 = 方法名   ↑ 方法实现地址
```

要看某个方法内部调了谁：从 `imp 0x...` 取地址，按 `__TEXT` 段的 `vmaddr` / `fileoff` 换算文件偏移，
再用 capstone 反汇编（记得 `md.detail = True` 才能读 operands）。

### 4.3 二进制里搜中文字符串会误判

含非 ASCII 的 ObjC 字符串字面量会被 clang 放进 **`__ustring` 段并以 UTF-16 存储**。
用 UTF-8 去 `strings` / `grep` 会**假报「字符串丢了」**。校验时两种编码都试：

```python
data = open(dylib, "rb").read()
assert s.encode() in data or s.encode("utf-16-le") in data
```

## 5. 交付前验证清单

**不要只看「编译成功」。** 从打出来的 deb 里实读，逐项确认：

- [ ] `Package:` / `Version:` / `Architecture:` 与方案相符
- [ ] payload 路径前缀正确（rootless 有 `var/jb`，roothide 没有）
- [ ] dylib 是 fat：`lipo -archs` 同时有 `arm64` 和 `arm64e`
- [ ] roothide：`otool -l dylib | grep -c LC_RPATH` ≥ 2
- [ ] `otool -L dylib` 里 CydiaSubstrate 是 weak，且 rpath 能解析到它
- [ ] Filter plist 的 `Bundles` 是目标 App 的 bundle id
- [ ] 关键 selector / 类名字符串确实在 dylib 里（**两种编码都试**）
- [ ] 用了 `%group` 门控的话，`nm | grep -c _logosLocalInit` 为 0
- [ ] 与上一版比体积，突增/突减通常意味着多打或漏打文件
- [ ] 把 deb 解出来核对，**不要用 src 目录里的 dylib 代替检查**

## 6. 常见症状 → 病因速查

| 症状 | 病因 |
|---|---|
| 所有配插件的 App 一启动就闪退，`pc=0` | roothide 缺 rpath，弱符号 CydiaSubstrate = NULL（§2.1） |
| 开机/注销后异常，或插件完全不生效 | 用 rootless 布局冒充 arm64e 交付（§0） |
| 覆盖安装报 `trying to overwrite` | 本地 deb 包名与已装包不一致（§2.4） |
| 守护进程不启动 | LaunchDaemon plist 是文本格式，或路径硬编码 `/var/jb`（§2.2 / §2.3） |
| SpringBoard 无限安全模式 | `_logosLocalInit` 无条件构造器（§3.1） |
| 日志或 `%orig` 一调就崩 | 接口声明类型与实际签名不符（§3.2） |
| hook 语法正确但完全不生效 | 目标是 Swift 类，走 vtable 派发（§3.3） |
| `make` 报路径乱码 | 中文路径 + GNU Make 3.81（§4.1） |
| class-dump 说没有 ObjC 运行时信息 | class-dump 太老，不认识 chained fixups（§4.2） |
| 搜二进制说字符串没编进去 | 非 ASCII 字面量在 `__ustring`，是 UTF-16（§4.3） |
