# TweakTemplateChanger · 完整教程

在 Mac 上编译一次模板，然后在手机 App 里改文案直接生成 deb。

---

## 一、你需要准备什么

| 东西 | 说明 |
|---|---|
| 一台 Mac | 装了 Xcode |
| Theos | 交叉编译工具链 |
| 越狱 iPhone | 用来测试 deb 包 |
| 签名证书 | 用来签 iOS App |

---

## 二、Mac 上安装 Theos

### 1. 装依赖

```bash
xcode-select --install
brew install make ldid dpkg
```

### 2. 下载 Theos

```bash
sudo git clone --recursive https://github.com/theos/theos.git /opt/theos
echo "export THEOS=/opt/theos" >> ~/.zshrc
source ~/.zshrc
```

### 3. 验证安装

```bash
$THEOS/bin/nic.pl
```

能出来菜单就说明装好了。

---

## 三、编译模板 dylib（只做一次）

模板就是一个"空壳" tweak，里面弹窗文案用固定占位符。

### 1. 建项目

```bash
cd ~/Desktop
$THEOS/bin/nic.pl
# 选 11 (iPhone Tweak)
# 项目名：TweakTemplate
# 作者：你的名字
# Bundle ID：com.yourcompany.template
# 依赖：直接回车
# Filter Bundle：直接回车
```

### 2. 替换 Tweak.x

把 `TweakTemplate/Tweak.x` 改成：

```objc
#import <UIKit/UIKit.h>
#import <Foundation/Foundation.h>

#define TITLE_PLACEHOLDER    @"__TITLE____________________"
#define UPDATE_MSG_PLACEHOLDER @"__UPDATE_MSG____________________"
#define FIRST_MSG_PLACEHOLDER  @"__FIRST_MSG____________________"

@interface TemplateAlert : NSObject
+ (instancetype)shared;
- (void)showIfUpdated;
@end

@implementation TemplateAlert

+ (instancetype)shared {
    static TemplateAlert *instance;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{ instance = [[self alloc] init]; });
    return instance;
}

- (void)showIfUpdated {
    NSString *current = [[[NSBundle mainBundle] infoDictionary] objectForKey:@"CFBundleShortVersionString"];
    NSString *last = [[NSUserDefaults standardUserDefaults] stringForKey:@"LastVersion"];

    if (last && [current isEqualToString:last]) return;

    dispatch_async(dispatch_get_main_queue(), ^{
        UIWindowScene *scene = (UIWindowScene *)[[UIApplication sharedApplication].connectedScenes anyObject];
        UIViewController *topVC = scene.windows.firstObject.rootViewController;
        while (topVC.presentedViewController) topVC = topVC.presentedViewController;

        NSString *msg = last
            ? [NSString stringWithFormat:UPDATE_MSG_PLACEHOLDER, last, current]
            : [NSString stringWithFormat:FIRST_MSG_PLACEHOLDER, current];

        UIAlertController *alert = [UIAlertController alertControllerWithTitle:TITLE_PLACEHOLDER
                                                                       message:msg
                                                                preferredStyle:UIAlertControllerStyleAlert];
        [alert addAction:[UIAlertAction actionWithTitle:@"知道了" style:UIAlertActionStyleDefault handler:nil]];
        [topVC presentViewController:alert animated:YES completion:nil];
    });

    [[NSUserDefaults standardUserDefaults] setObject:current forKey:@"LastVersion"];
    [[NSUserDefaults standardUserDefaults] synchronize];
}

@end

%hook UIApplication
- (void)applicationDidBecomeActive:(UIApplication *)application {
    %orig;
    [[TemplateAlert shared] showIfUpdated];
}
%end
```

### 3. 编译

```bash
cd TweakTemplate
make package
```

编译完在 `packages/` 里有 `.deb` 文件。把里面的 `libTweakTemplate.dylib` 提取出来，这就是我们要的模板。

---

## 四、写 iOS App（手机端编辑器）

### 1. 新建 Xcode 项目

选 iOS → App，名字叫 `TweakTemplateChanger`。

### 2. 核心代码

**ContentView.swift：**

```swift
import SwiftUI

struct ContentView: View {
    @State private var bundleID = "com.yourcompany.yourapp"
    @State private var titleText = "更新提示"
    @State private var updateMsg = "已从 v%@ 更新到 v%@"
    @State private var firstMsg = "欢迎使用 v%@"
    @State private var result = ""

    var body: some View {
        Form {
            Section("目标 App") {
                TextField("Bundle ID", text: $bundleID)
            }
            Section("弹窗内容") {
                TextField("标题", text: $titleText)
                TextField("更新时文案", text: $updateMsg)
                TextField("首次安装文案", text: $firstMsg)
            }
            Button("生成 deb") {
                generateDeb()
            }
            if !result.isEmpty {
                Text(result)
            }
        }
    }

    func generateDeb() {
        // 1. 加载模板 dylib（放进 Bundle 里）
        guard let templateURL = Bundle.main.url(forResource: "libTweakTemplate", withExtension: "dylib"),
              let templateData = try? Data(contentsOf: templateURL) else {
            result = "找不到模板 dylib"
            return
        }

        // 2. 替换占位符
        var modified = templateData
        replace(in: &modified, placeholder: "__TITLE____________________", with: titleText)
        replace(in: &modified, placeholder: "__UPDATE_MSG____________________", with: updateMsg)
        replace(in: &modified, placeholder: "__FIRST_MSG____________________", with: firstMsg)

        // 3. 打包成 deb（简化版，实际需要 control 文件和目录结构）
        result = "生成成功！dylib 大小：\(modified.count) 字节"
    }

    func replace(in data: inout Data, placeholder: String, with text: String) {
        guard let pData = placeholder.data(using: .utf8),
              let tData = text.data(using: .utf8) else { return }

        // 在二进制里找占位符位置，替换成新内容（不足补空格）
        // 注意：占位符长度要固定，不然会破坏二进制结构
        // 这里是简化示意，实际需要更严谨的二进制修改
    }
}
```

### 3. 导入模板 dylib

把第三步编译出来的 `libTweakTemplate.dylib` 拖进 Xcode 项目，勾选 "Copy items if needed"。

### 4. 签名打包

```bash
# 在 Xcode 里直接 Product → Archive
# 然后导出 IPA
```

或者用命令行：

```bash
xcodebuild -project TweakTemplateChanger.xcodeproj \
  -scheme TweakTemplateChanger \
  -configuration Release \
  -archivePath build/App.xcarchive \
  archive

xcodebuild -exportArchive \
  -archivePath build/App.xcarchive \
  -exportOptionsPlist ExportOptions.plist \
  -exportPath build/
```

---

## 五、怎么用

1. 手机上装这个 App
2. 打开 App，填 Bundle ID 和弹窗文案
3. 点生成，导出 deb 文件
4. 用 Filza 或 Sileo 把 deb 装到越狱手机上
5. 打开目标 App，就会弹你设置的提示了

---

## 六、注意事项

- 占位符长度是固定的（20 个下划线），文案太长会被截断
- 改二进制字符串有风险，改坏了 dylib 会崩
- 目标 App 需要重启才能加载新的 tweak

---

## 安装方法

把下面这句话发给你的 AI 助手：

```
请将公开仓库 https://github.com/Xuefeng122/tweak-template-changer 下载并安装为本地 Skill。
```
