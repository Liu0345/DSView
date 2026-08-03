# 苹果电脑编译与打包说明

本文说明如何在苹果芯片电脑上编译 PXLogic-DSView，并生成可分发的 `PXLogic-DSView-1.3.2-macOS-arm64.dmg`。

## 适用范围

- 处理器架构：苹果芯片，也就是 `arm64`。
- 构建系统：CMake 3.16 或更高版本。
- 图形界面：Qt 6。
- 应用版本：1.3.2。
- 应用显示名称：`PXLogic-DSView`。
- 镜像内应用名称：`PXLogic-DSView.app`。

当前流程已在苹果芯片电脑和 macOS 26.5.1 上验证。虽然配置命令中设置了 `CMAKE_OSX_DEPLOYMENT_TARGET=13.0`，但 Homebrew 预编译库本身可能要求更高版本的 macOS。因此，在其他系统版本上发布前，仍需使用目标系统实际测试，不能只根据该配置项判断兼容性。

## 所需工具和依赖

### 系统工具

安装苹果命令行开发工具：

```bash
xcode-select --install
```

该工具包提供编译器、链接器以及打包时需要的 `codesign`、`install_name_tool`、`otool` 和其他系统命令。`hdiutil` 由 macOS 自带，用于创建和校验 DMG 镜像。

### Homebrew 依赖

先安装 Homebrew，然后安装以下依赖：

```bash
brew install cmake pkg-config qt glib python@3.11 fftw libusb boost
```

主要依赖的用途如下：

| 依赖 | 用途 |
| --- | --- |
| CMake | 配置、编译和安装工程 |
| pkg-config | 帮助 CMake 查找 GLib 等库 |
| Qt 6 | 图形界面、窗口、SVG 图标、图像格式和 DBus 支持 |
| GLib | 基础数据结构、事件及工具函数 |
| Python 3.11 | 协议解码器运行环境和开发库 |
| FFTW | 频谱和快速傅里叶变换 |
| libusb 1.0 | 与逻辑分析仪等 USB 设备通信 |
| Boost | C++ 通用组件和头文件 |
| Zlib | 数据压缩，由 macOS 系统提供 |

Qt 在苹果电脑上还必须提供以下模块或运行插件：

- `QtCore`
- `QtGui`
- `QtWidgets`
- `QtSvg`
- `QtDBus`
- `libqcocoa.dylib`
- `libqmacstyle.dylib`
- `libqsvgicon.dylib`
- `libqsvg.dylib`
- `libqgif.dylib`

最后五个插件必须进入应用的 `Contents/PlugIns` 目录。缺少 `libqsvgicon.dylib` 或 `libqsvg.dylib` 时，程序虽然能够启动，但工具栏中的模式、开始、触发、解码、测量、搜索、显示和文件等 SVG 图标会变成空白。

## 检查依赖环境

```bash
cmake --version
pkg-config --version
clang --version
brew --prefix qt
brew --prefix glib
brew --prefix python@3.11
brew --prefix fftw
brew --prefix libusb
brew --prefix boost
```

查看 Qt 插件目录：

```bash
"$(brew --prefix qt)/bin/qtpaths" --plugin-dir
```

## 配置编译目录

在仓库根目录，也就是本文件和 `README.md` 所在目录执行以下命令：

```bash
SOURCE_DIR="$(pwd)"
WORK_DIR="${TMPDIR%/}/pxlogic-dsview-arm64"
BUILD_DIR="$WORK_DIR/build"
STAGE_DIR="$WORK_DIR/stage"

QT_PREFIX="$(brew --prefix qt)"
GLIB_PREFIX="$(brew --prefix glib)"
PYTHON_PREFIX="$(brew --prefix python@3.11)"
FFTW_PREFIX="$(brew --prefix fftw)"
LIBUSB_PREFIX="$(brew --prefix libusb)"
BOOST_PREFIX="$(brew --prefix boost)"

mkdir -p "$BUILD_DIR" "$STAGE_DIR"
```

这里把中间文件放到系统临时目录中，避免构建文件污染 Git 仓库。确认最终 DMG 正常后，可以把整个 `$WORK_DIR` 移入废纸篓。

## 配置 CMake

```bash
cmake -S "$SOURCE_DIR" -B "$BUILD_DIR" \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_OSX_ARCHITECTURES=arm64 \
  -DCMAKE_OSX_DEPLOYMENT_TARGET=13.0 \
  -DCMAKE_INSTALL_PREFIX="$STAGE_DIR" \
  -DCMAKE_PREFIX_PATH="$QT_PREFIX;$GLIB_PREFIX;$LIBUSB_PREFIX;$FFTW_PREFIX;$BOOST_PREFIX;$PYTHON_PREFIX" \
  -DPython3_ROOT_DIR="$PYTHON_PREFIX" \
  -DPython3_EXECUTABLE="$PYTHON_PREFIX/bin/python3.11" \
  -DFFTW_INCLUDE_DIR="$FFTW_PREFIX/include" \
  -DFFTW_LIBRARY="$FFTW_PREFIX/lib/libfftw3.dylib" \
  -DLIBUSB_1_INCLUDE_DIRS="$LIBUSB_PREFIX/include/libusb-1.0" \
  -DLIBUSB_1_LIBRARIES="$LIBUSB_PREFIX/lib/libusb-1.0.dylib"
```

配置成功时，应在输出中看到 GLib、Python 3、FFTW、libusb、Zlib、Qt 6 和 Boost 均已找到。若 CMake 选中了 Qt 5，说明当前源码修改没有生效或配置缓存来自旧构建，应删除旧编译目录后重新配置。

## 编译和安装应用

```bash
cmake --build "$BUILD_DIR" --parallel "$(sysctl -n hw.ncpu)"
cmake --install "$BUILD_DIR" --prefix "$STAGE_DIR"
```

编译产物位于：

```text
$BUILD_DIR/bin/DSView.app
```

安装后的应用位于：

```text
$STAGE_DIR/DSView.app
```

这里的目录名仍然是 `DSView.app`，因为 CMake 工程目标名保持为 `DSView`。应用的显示名称、窗口标题和最终 DMG 中的目录名均使用 `PXLogic-DSView`。制作 DMG 时再把应用目录复制为 `PXLogic-DSView.app`。

## 本机运行测试

未封装依赖前，应用可以在安装了相同 Homebrew 依赖的编译电脑上测试：

```bash
open "$STAGE_DIR/DSView.app"
```

测试时至少确认以下内容：

- 主窗口能够正常打开。
- 顶部标题显示为 `PXLogic-DSView v1.3.2`。
- 工具栏中的所有 SVG 图标正常显示。
- 演示设备能够初始化。
- 连接真实设备时，设备能够被 libusb 识别。

## 封装应用依赖

仅完成编译还不能直接把应用复制给其他电脑，因为可执行文件可能仍然引用 `/opt/homebrew` 中的动态库。制作 DMG 前必须把依赖复制到应用内部，并改写动态库路径。

### 使用 Qt 部署工具

```bash
APP="$STAGE_DIR/DSView.app"

"$QT_PREFIX/bin/macdeployqt" "$APP" \
  -verbose=1 \
  -always-overwrite \
  -no-codesign \
  -no-plugins \
  -libpath="$QT_PREFIX/lib"
```

`macdeployqt` 会复制 Qt 框架和大部分关联动态库。由于 Python 框架的位置不符合该工具的默认查找规则，输出中可能出现找不到 Python 框架的提示，Python 需要在后续步骤中单独处理。

### 复制必要的 Qt 插件

```bash
QT_PLUGIN_DIR="$("$QT_PREFIX/bin/qtpaths" --plugin-dir)"
APP_PLUGIN_DIR="$APP/Contents/PlugIns"

mkdir -p \
  "$APP_PLUGIN_DIR/platforms" \
  "$APP_PLUGIN_DIR/styles" \
  "$APP_PLUGIN_DIR/iconengines" \
  "$APP_PLUGIN_DIR/imageformats"

ditto "$QT_PLUGIN_DIR/platforms/libqcocoa.dylib" \
  "$APP_PLUGIN_DIR/platforms/libqcocoa.dylib"
ditto "$QT_PLUGIN_DIR/styles/libqmacstyle.dylib" \
  "$APP_PLUGIN_DIR/styles/libqmacstyle.dylib"
ditto "$QT_PLUGIN_DIR/iconengines/libqsvgicon.dylib" \
  "$APP_PLUGIN_DIR/iconengines/libqsvgicon.dylib"
ditto "$QT_PLUGIN_DIR/imageformats/libqsvg.dylib" \
  "$APP_PLUGIN_DIR/imageformats/libqsvg.dylib"
ditto "$QT_PLUGIN_DIR/imageformats/libqgif.dylib" \
  "$APP_PLUGIN_DIR/imageformats/libqgif.dylib"
```

这些插件在 Homebrew 中通常带有指向安装目录的运行路径。复制后需要使用 `install_name_tool` 把运行路径改为：

```text
@loader_path/../../Frameworks
```

其中 `libqsvgicon.dylib` 和 `libqsvg.dylib` 对 `QtGui`、`QtCore`、`QtSvg` 的引用也必须改为应用内部的框架路径。修改后用 `otool -L` 和 `otool -l` 检查，不能继续引用 `/opt/homebrew`。

### 内置 Python 3.11

协议解码功能依赖 Python 3.11。需要把完整框架复制到应用内：

```bash
FRAMEWORKS="$APP/Contents/Frameworks"

ditto \
  "$PYTHON_PREFIX/Frameworks/Python.framework" \
  "$FRAMEWORKS/Python.framework"
```

随后需要完成以下处理：

1. 把主程序对 Python 的引用改为 `@executable_path/../Frameworks/Python.framework/Versions/3.11/Python`。
2. 把 Python 框架自身的安装名称改为 `@rpath/Python.framework/Versions/3.11/Python`。
3. 修正框架内 `bin/python3.11` 和 `Python.app` 对 Homebrew Cellar 路径的引用。
4. 建立 `Versions/Current`、顶层 `Python`、`Headers` 和 `Resources` 符号链接。
5. 删除指向 Homebrew 外部目录的 `site-packages` 符号链接，并在应用内建立空目录。
6. 检查 Python 原生扩展模块。`_sqlite3`、`_lzma`、`_decimal`、`_hashlib` 和 `_ssl` 还分别依赖 SQLite、XZ、mpdecimal 和 OpenSSL，需要把对应动态库复制到 `Contents/Frameworks` 并改写路径。
7. 删除应用中的 `__pycache__` 目录和 `.pyc` 文件，避免运行时缓存破坏应用签名。

这一部分与 Homebrew 中 Python 和 OpenSSL 的具体版本有关。不要把包含 Cellar 版本号的绝对路径写死；应通过 `brew --prefix` 和 `otool -L` 读取当前路径后再调用 `install_name_tool -change`。

## 检查外部动态库引用

下面的检查结果必须为 0：

```bash
EXTERNAL_DEPS=0
EXTERNAL_RPATHS=0

while IFS= read -r -d '' FILE; do
  if file "$FILE" | grep -q 'Mach-O'; then
    COUNT="$(otool -L "$FILE" | grep -Ec '/opt/homebrew|/usr/local' || true)"
    EXTERNAL_DEPS=$((EXTERNAL_DEPS + COUNT))

    COUNT="$(otool -l "$FILE" \
      | awk '/LC_RPATH/{getline; getline; print $2}' \
      | grep -Ec '^/opt/homebrew|^/usr/local' || true)"
    EXTERNAL_RPATHS=$((EXTERNAL_RPATHS + COUNT))
  fi
done < <(find "$APP" -type f -print0)

echo "外部依赖：$EXTERNAL_DEPS"
echo "外部运行路径：$EXTERNAL_RPATHS"
```

同时检查所有 Mach-O 文件都是 `arm64`：

```bash
while IFS= read -r -d '' FILE; do
  if file "$FILE" | grep -q 'Mach-O'; then
    file "$FILE" | grep -q 'arm64' || echo "架构错误：$FILE"
  fi
done < <(find "$APP" -type f -print0)
```

## 应用签名

修改动态库后，原有签名会失效。测试包可以使用临时签名。应先签名所有内部 Mach-O 文件和嵌套框架，最后签名整个应用：

```bash
while IFS= read -r -d '' FILE; do
  if file "$FILE" | grep -q 'Mach-O'; then
    codesign --force --sign - --timestamp=none "$FILE"
  fi
done < <(find "$APP" -type f -print0)

PYTHON_APP="$APP/Contents/Frameworks/Python.framework/Versions/3.11/Resources/Python.app"
if [ -d "$PYTHON_APP" ]; then
  codesign --force --sign - --timestamp=none "$PYTHON_APP"
fi

while IFS= read -r -d '' FRAMEWORK; do
  codesign --force --sign - --timestamp=none "$FRAMEWORK"
done < <(find "$APP/Contents/Frameworks" -depth -type d -name '*.framework' -print0)

codesign --force --sign - --timestamp=none "$APP"
codesign --verify --deep --strict --verbose=2 "$APP"
```

临时签名只适合本机测试和内部传递。面向普通用户公开发布时，应使用苹果开发者证书签名，并通过苹果公证服务完成公证和装订，否则 Gatekeeper 仍可能提示应用来自身份不明的开发者。

## 创建 DMG

```bash
DMG_ROOT="$WORK_DIR/dmg-root"
OUTPUT_DMG="$WORK_DIR/PXLogic-DSView-1.3.2-macOS-arm64.dmg"

mkdir -p "$DMG_ROOT"
ditto "$APP" "$DMG_ROOT/PXLogic-DSView.app"
ln -s /Applications "$DMG_ROOT/Applications"

hdiutil create \
  -volname 'PXLogic DSView 1.3.2' \
  -srcfolder "$DMG_ROOT" \
  -format UDZO \
  -ov \
  "$OUTPUT_DMG"
```

## 校验 DMG

```bash
hdiutil verify "$OUTPUT_DMG"
shasum -a 256 "$OUTPUT_DMG"
```

挂载后还要确认：

- 根目录包含 `PXLogic-DSView.app`。
- 根目录不包含旧名称 `DSView.app`。
- `Applications` 是指向系统应用程序目录的符号链接。
- `codesign --verify --deep --strict` 检查通过。
- 命令行版本显示为 `PXLogic-DSView 1.3.2`。
- 递归检查没有 `/opt/homebrew` 或 `/usr/local` 动态库引用。
- 所有 Mach-O 文件都是 `arm64`。
- 实际启动后标题和工具栏图标正常。

## 清理临时文件

最终 DMG 通过挂载和运行验证后，把 `$WORK_DIR` 整体移入废纸篓，只保留最终镜像和源码：

```bash
trash "$WORK_DIR"
```

如果系统没有 `trash` 命令，可以在访达中手动把该目录移入废纸篓。清理前必须先卸载仍然挂载在该目录中的验证镜像。

## 常见问题

### 工具栏图标全部或部分消失

重点检查以下文件是否存在：

```text
Contents/PlugIns/iconengines/libqsvgicon.dylib
Contents/PlugIns/imageformats/libqsvg.dylib
Contents/PlugIns/imageformats/libqgif.dylib
```

还要检查这些插件能否从 `@loader_path/../../Frameworks` 找到应用内的 Qt 框架。

### 程序在编译电脑上能启动，在其他电脑上不能启动

通常是动态库没有封装完整。使用 `otool -L` 和 `otool -l` 递归检查应用中的所有 Mach-O 文件，确认不存在 Homebrew 绝对路径。

### 启动时被系统直接终止

通常是内部 Python 原生扩展或动态库在修改后没有重新签名。必须先逐个签名内部文件，再签名嵌套框架，最后签名整个 `.app`。

### CMake 找不到 Python、FFTW 或 libusb

通过 `brew --prefix` 确认安装位置，并使用本文配置命令中的 `Python3_ROOT_DIR`、`FFTW_INCLUDE_DIR`、`FFTW_LIBRARY`、`LIBUSB_1_INCLUDE_DIRS` 和 `LIBUSB_1_LIBRARIES` 显式指定路径。

### 设置了较低的部署版本，旧版 macOS 仍然无法运行

`CMAKE_OSX_DEPLOYMENT_TARGET` 只控制当前工程的编译目标，不能降低已安装 Homebrew 预编译库的最低系统版本。如需支持更旧的 macOS，需要在对应系统或匹配的构建环境中重新编译全部依赖，并在真实目标系统上验证。
