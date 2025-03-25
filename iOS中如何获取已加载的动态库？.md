嗯，用户问的是iOS中如何获取已加载的动态库。这个问题我之前有了解过一些，但需要结合最新的搜索结果来确认正确的方法。首先看看用户提供的搜索结果，特别是网页1、2、3提到了使用dlopen、dlsym和dlclose来动态加载和操作动态库，但用户的问题是关于获取已加载的动态库，所以可能需要其他方法。

网页6提到了动态库的加载机制，提到了动态链接器dyld的作用，可能和获取已加载库有关。网页7则详细说明了系统动态库在真机上的存储路径，比如dyld_shared_cache_arm64，还提到了使用dyld源码中的工具来提取动态库。不过用户的问题是关于获取已加载的库，可能包括应用自己加载的和系统已经加载的。

另外，网页4提到了动态库的懒加载，使用dlopen来加载，但可能也涉及到如何检查已经加载的库。网页5讨论了重签名动态库，可能不直接相关，但其中的工具如otool可能在查看依赖库时有用。

根据这些信息，可能的解决方法包括使用Objective-C运行时的方法，比如NSBundle或者Runtime函数。例如，NSBundle的allFrameworks方法可以获取所有已加载的框架。此外，dyld的API，如_dyld_image_count和_dyld_get_image_name，可以遍历当前加载的所有动态库。

另外，网页7提到使用dlsym和dladdr函数来获取符号信息，这可能用于检查特定函数是否来自某个动态库。还有，通过NSProcessInfo的arguments可能获取到DYLD_INSERT_LIBRARIES注入的库，但这种方法在非越狱环境下可能受限。

需要注意的是，某些方法可能需要私有API，比如使用_dyld_get_image_name，这在App Store应用中是禁止的，所以需要区分开发调试和上架应用的不同场景。另外，使用otool命令行工具分析Mach-O文件可以查看依赖的动态库，但这是静态分析，不是运行时获取。

总结下来，主要的获取方法可能有几种：使用NSBundle的API、dyld的私有函数、Runtime方法以及命令行工具。需要结合不同的使用场景，比如调试时可以使用私有API，而正式环境可能需要其他方法。




在 iOS 系统中获取已加载的动态库可以通过多种方式实现，以下是具体方法及注意事项：

---

### 一、通过 **NSBundle 接口**获取（公开 API，适合 App Store 应用）
**核心原理**：  
利用 `NSBundle` 类提供的运行时查询接口，获取所有已加载的 Framework 和动态库路径。

**实现代码**：
```objective-c
// 获取所有已加载的 Framework
NSArray<NSBundle *> *frameworks = [NSBundle allFrameworks];
for (NSBundle *bundle in frameworks) {
    NSLog(@"Framework路径: %@", bundle.bundlePath);
}

// 获取所有动态库（包含系统库和第三方库）
NSArray<NSString *> *loadedLibraries = [NSBundle allBundles];
for (NSString *path in loadedLibraries) {
    NSLog(@"动态库路径: %@", path);
}
```
**注意事项**：  
此方法仅能获取通过 `NSBundle` 加载的动态库，无法检测通过 `dlopen` 动态加载的私有库。

---

### 二、使用 **dyld 私有 API**（仅限调试，不适用于 App Store）
**核心原理**：  
通过访问动态链接器（dyld）的私有函数 `_dyld_get_image_name` 和 `_dyld_image_count`，遍历所有已加载的动态库。

**实现代码**：
```objective-c
#include <mach-o/dyld.h>

// 遍历所有已加载动态库
for (uint32_t i = 0; i < _dyld_image_count(); i++) {
    const char *imagePath = _dyld_get_image_name(i);
    NSLog(@"动态库路径: %s", imagePath);
}
```
**输出示例**：
```
/System/Library/Frameworks/UIKit.framework/UIKit
/usr/lib/libSystem.B.dylib
@rpath/AlipaySDK.framework/AlipaySDK
```
**适用场景**：  
适用于越狱设备或调试阶段，需注意此方法使用私有 API，提交 App Store 会被拒绝。

---

### 三、通过 **dladdr 函数**获取符号来源（适用于动态加载的库）
**核心原理**：  
通过 `dladdr` 函数解析某个函数或变量的内存地址，反向查找其所属动态库路径。

**示例代码**：
```objective-c
#include <dlfcn.h>

// 选择一个已知函数地址（如 NSLog）
Dl_info info;
if (dladdr((void *)&NSLog, &info)) {
    NSLog(@"NSLog 所属动态库: %s", info.dli_fname);  // 输出：/usr/lib/libSystem.B.dylib
}
```
**典型用途**：  
验证特定函数是否来自预期动态库，常用于调试注入库或私有 API 调用。

---

### 四、通过 **otool 命令行工具**（静态分析）
**核心原理**：  
使用 `otool -L` 命令分析 Mach-O 文件的动态库依赖列表，适用于查看编译时链接的库。

**操作步骤**：
```bash
# 查看可执行文件依赖的动态库
otool -L /path/to/YourApp.app/YourApp

# 输出示例：
@rpath/AlipaySDK.framework/AlipaySDk (compatibility version 1.0.0)
/usr/lib/libobjc.A.dylib
```
**限制**：  
仅能查看编译时声明的依赖，无法检测运行时通过 `dlopen` 动态加载的库。

---

### 五、特殊场景：获取 **DYLD_INSERT_LIBRARIES 注入的库**
**核心原理**：  
通过读取进程启动参数中的 `DYLD_INSERT_LIBRARIES` 环境变量，获取注入的动态库路径。

**实现代码**：
```objective-c
NSProcessInfo *processInfo = [NSProcessInfo processInfo];
NSArray<NSString *> *arguments = processInfo.arguments;
for (NSString *arg in arguments) {
    if ([arg containsString:@"DYLD_INSERT_LIBRARIES"]) {
        NSLog(@"注入库路径: %@", arg);
    }
}
```
**适用性**：  
仅在通过环境变量注入动态库时有效（如越狱插件），常规应用无法使用此方法。

---

### 六、注意事项
1. **审核风险**：  
   使用 `_dyld_get_image_name` 等私有 API 会导致 App Store 审核被拒，需在 Release 版本中移除。
2. **动态加载库检测**：  
   通过 `dlopen` 加载的动态库需结合 `dlsym` 遍历符号表或维护全局句柄列表才能完整获取。
3. **系统库路径**：  
   系统库通常位于 `/System/Library/` 或 `/usr/lib/`，第三方库路径可能包含 `@rpath` 或 `@executable_path`。

---

通过上述方法，开发者可以根据实际需求（调试、逆向或常规开发）选择合适的方式获取已加载动态库信息。
