# RE4 Mod Tool

生化危机4重置版服装mod颜色自定义自动化工具。

## 功能

- 自动解析游戏贴图格式（.tex/.dds）
- 直观的调色盘界面（RGB/HSL滑块、颜色调整）
- 3D实时预览（通过REFramework游戏内集成）
- 路径配置系统（多mod多部件）
- 批量处理（格式转换、颜色调整、尺寸调整）
- 预设系统（保存/加载/分享颜色配置）
- 撤销/重做支持

## 安装

### 桌面工具
1. 下载最新发布版本
2. 运行安装程序
3. 按照向导完成安装

### DLL插件（游戏内预览）
1. 安装 [REFramework](https://github.com/praydog/REFramework)
2. 将 `re4modtool.dll` 复制到 `游戏目录/reframework/plugins/`
3. 启动游戏

## 使用

1. 启动 RE4 Mod Tool
2. 打开贴图文件（支持 .tex/.dds/.png/.tga）
3. 使用调色盘编辑颜色
4. 保存修改后的贴图

### 游戏内预览
1. 启动 RE4 游戏（确保 REFramework 和插件已安装）
2. 启动 RE4 Mod Tool
3. 在桌面工具中编辑颜色，游戏中实时预览效果

## 构建

### 前置条件
- Visual Studio 2022
- CMake 3.21+
- vcpkg
- Qt 6.8

### 步骤
```bash
# 克隆仓库
git clone https://github.com/yourname/re4modtool.git
cd re4modtool

# 安装依赖
vcpkg install

# 构建
cmake --preset vcpkg-debug
cmake --build build/debug --config Debug
```

### 构建DLL插件
```bash
cd re4modtool_dll
cmake -B build -G "Visual Studio 17 2022"
cmake --build build --config Release
```

## 支持的格式

| 格式 | 读取 | 写入 |
|------|------|------|
| .tex (RE Engine) | ✅ | ✅ |
| .dds (DirectDraw Surface) | ✅ | ✅ |
| .png (Portable Network Graphics) | ✅ | ✅ |
| .tga (Targa) | ✅ | ✅ |

## 许可证

MIT License

## 📖 使用手册

详细的使用说明请查看：[使用手册](docs/user_guide/README.md)

### 快速链接

- [安装指南](docs/user_guide/README.md#3-安装指南)
- [快速入门](docs/user_guide/README.md#4-快速入门)
- [功能详解](docs/user_guide/README.md#5-功能详解)
- [游戏内预览](docs/user_guide/README.md#6-游戏内预览)
- [批量处理](docs/user_guide/README.md#7-批量处理)
- [常见问题](docs/user_guide/README.md#11-常见问题)
