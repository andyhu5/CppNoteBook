# 使用 windeployqt.exe 生成可独立运行的 Qt 应用程序

## 什么是 windeployqt.exe？

`windeployqt.exe` 是 Qt 官方提供的部署工具，用于**自动分析你的可执行文件（.exe）依赖了哪些 Qt 库、插件、QML 模块**，并将它们自动复制到你的程序目录下，使程序可以在没有安装 Qt 环境的电脑上独立运行。

它通常位于：

```
<Qt安装目录>\<版本>\<编译器>\bin\windeployqt.exe
```

例如：

```
C:\Qt\6.5.0\mingw_64\bin\windeployqt.exe
C:\Qt\6.5.0\msvc2019_64\bin\windeployqt.exe
```

------

## 基本使用步骤

### 第一步：先用 Release 模式编译你的项目

⚠️ **重要**：一定要用 **Release** 模式编译，Debug 模式生成的依赖库体积巨大且运行慢。

在 Qt Creator 中：左下角选择 **Release** 构建模式 → 点击构建（锤子图标）

编译完成后，会生成一个 `.exe` 文件，例如：

```
build-MyApp-Desktop_Qt_6_5_0-Release\release\MyApp.exe
```

------

### 第二步：新建一个干净的部署目录

```cmd
mkdir D:\MyApp_Deploy
copy build-MyApp-Desktop_Qt_6_5_0-Release\release\MyApp.exe D:\MyApp_Deploy\
```

⚠️ 建议单独拷贝到一个新文件夹，不要直接在 build 目录里操作（避免混入编译产生的中间文件）。

------

### 第三步：打开命令行工具（推荐用 Qt 自带的命令行）

打开开始菜单，找到：

```
Qt 6.5.0 (MinGW 11.2.0 64-bit) → "Qt 6.5.0 (MinGW 11.2.0 64-bit)" 命令行
```

（这个命令行已经自动配置好了 PATH 环境变量）

或者手动设置环境变量后使用普通 cmd：

```cmd
set PATH=C:\Qt\6.5.0\mingw_64\bin;%PATH%
```

------

### 第四步：运行 windeployqt

```cmd
cd D:\MyApp_Deploy
windeployqt.exe MyApp.exe
```

执行后，windeployqt 会自动：

- 分析 `MyApp.exe` 依赖的 Qt DLL（如 `Qt6Core.dll`、`Qt6Gui.dll`、`Qt6Widgets.dll` 等）
- 复制必需的 **platforms 插件**（如 `qwindows.dll`，没有这个程序无法启动！）
- 复制 QML 相关模块（如果使用了 QML/Quick）
- 复制字体、图像格式插件等

------

## 常用参数

```cmd
windeployqt.exe --qmldir <QML源码目录> MyApp.exe
```

| 参数                 | 作用                                                         |
| -------------------- | ------------------------------------------------------------ |
| `--qmldir <路径>`    | 如果项目用了 **QML**，必须加这个参数，指向你的 .qml 文件所在目录，否则不会正确扫描 QML 依赖 |
| `--release`          | 只拷贝 Release 版本的库（默认已根据 exe 类型判断）           |
| `--no-translations`  | 不拷贝多语言翻译文件（可减小体积）                           |
| `--no-opengl-sw`     | 不拷贝软件渲染 OpenGL 库（如果确定不需要软件渲染）           |
| `--compiler-runtime` | 拷贝 MSVC/MinGW 的运行时库（如 vcruntime、msvcp 等），**强烈建议加上**，否则用户电脑没装 VC++ 运行库会启动失败 |
| `--verbose=2`        | 显示详细日志，方便调试缺失了哪些依赖                         |

### 针对 QML 项目的推荐命令：

```cmd
windeployqt.exe --qmldir D:\MyApp\qml --compiler-runtime MyApp.exe
```

------

## 第五步：检查生成结果

执行完成后，`D:\MyApp_Deploy` 目录大致会变成：

```
MyApp_Deploy/
├── MyApp.exe
├── Qt6Core.dll
├── Qt6Gui.dll
├── Qt6Widgets.dll
├── Qt6Qml.dll          (如果是QML项目)
├── Qt6Quick.dll
├── platforms/
│   └── qwindows.dll     ← 必需！没有这个程序无法启动
├── styles/
├── imageformats/
├── qml/                 (QML模块)
│   └── QtQuick/
│       └── Controls/
└── ...
```

------

## 第六步：测试

**关键测试方法**：把整个 `MyApp_Deploy` 文件夹拷贝到一台**没有安装 Qt 和 Visual Studio** 的电脑上，双击 `MyApp.exe`，能正常启动就说明部署成功。

------

## 常见问题排查

| 问题现象                                                   | 可能原因                                 | 解决方法                                                     |
| ---------------------------------------------------------- | ---------------------------------------- | ------------------------------------------------------------ |
| 双击无反应/闪退                                            | 缺少 `platforms\qwindows.dll`            | 确认 windeployqt 是否执行成功，该文件是否存在                |
| 提示缺少 `VCRUNTIME140.dll`                                | 缺少 VC++ 运行库                         | 加 `--compiler-runtime` 参数，或让用户安装 [VC++ Redistributable](https://learn.microsoft.com/cpp/windows/latest-supported-vc-redist) |
| QML 界面显示空白/报模块找不到                              | 未指定 `--qmldir` 或 QML import 路径错误 | 加上 `--qmldir <qml源码路径>` 重新运行                       |
| 缺少 `libgcc_s_seh-1.dll`、`libstdc++-6.dll`（MinGW 编译） | MinGW 运行时库未拷贝                     | 手动从 `Qt\Tools\mingw_64\bin` 拷贝，或用 `--compiler-runtime` |
| 打包体积太大                                               | 拷贝了不需要的翻译/调试符号              | 加 `--no-translations`，删除多余的 `imageformats`/`platforms` 中用不到的插件 |

------

## 打包为安装包（可选，进阶）

如果希望生成一个真正的 **安装程序（.exe/.msi）** 而不只是文件夹，可以配合以下工具：

- **Inno Setup**（免费，最常用）—— 把 `MyApp_Deploy` 文件夹打包成 `setup.exe`
- **NSIS**（免费）
- **Qt Installer Framework**（Qt 官方工具，适合复杂场景）

需要的话我可以帮你写一个 Inno Setup 的打包脚本示例。

------

## 完整流程总结

```cmd
:: 1. Release 模式编译（在 Qt Creator 里完成）

:: 2. 新建部署目录并拷贝exe
mkdir D:\MyApp_Deploy
copy release\MyApp.exe D:\MyApp_Deploy\

:: 3. 打开 Qt 命令行，进入部署目录
cd D:\MyApp_Deploy

:: 4. 执行部署（QML项目示例）
windeployqt.exe --qmldir D:\MyApp\qml --compiler-runtime --verbose=2 MyApp.exe

:: 5. 拷贝到无Qt环境的电脑测试运行
```