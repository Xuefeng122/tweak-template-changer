# TweakTemplateChanger

在手机上修改预编译好的 tweak 模板，一键生成新的 deb 包。

## 原理

不用在手机上编译，改字符串就行：

1. 提前在电脑上用 Theos 编译好一个"空模板" dylib
2. 模板里弹窗标题、内容都是固定占位符（比如 `__TITLE__`、`__MSG__`）
3. 手机 App 打开模板 dylib，替换占位符成你要的文案
4. 重新打包成 deb 导出

## 限制

- 只能改弹窗标题和内容（固定长度内）
- 改不了逻辑、版本号判断方式这些
- 模板 dylib 需要提前在电脑上编译好

## 项目结构

```
TweakTemplateChanger/
├── Template/              # 预编译好的模板 dylib（需要自己编译）
│   └── libTweakTemplate.dylib
├── TweakTemplateChanger/  # iOS App 源码（SwiftUI）
│   ├── ContentView.swift
│   ├── DylibPatcher.swift
│   └── TweakTemplateChangerApp.swift
├── Makefile              # 电脑上编译模板用
├── TweakTemplate.plist
└── TweakTemplate.x       # 模板源码
```

## 使用

### 第一步：电脑上编译模板（只做一次）

```bash
cd Template
make package
```

把编译出来的 `libTweakTemplate.dylib` 放进项目的 `Template/` 目录。

### 第二步：手机上用 App

1. 打开 App
2. 输入弹窗标题和内容
3. 点生成
4. 导出 deb 文件

## 模板里的占位符

| 占位符 | 位置 |
|---|---|
| `__TITLE__` | 弹窗标题 |
| `__MSG_UPDATE__` | 更新时的文案 |
| `__MSG_FIRST__` | 首次安装的文案 |

每个占位符预留 128 字节，超长会截断。

## 安装方法

把下面这句话发给你的 AI 助手：

```
请将公开仓库 https://github.com/Xuefeng122/tweak-template-changer 下载并安装为本地 Skill。
```
