# RE4 Mod Tool - Phase 6: System Integration + Installer

> **日期：** 2026-06-18
> **状态：** 实施中
> **依赖：** Phase 1-5

## 目标

1. 修复遗留的3个测试失败
2. 完善CI/CD配置
3. 创建Windows安装程序（Inno Setup）
4. 编写README和用户文档
5. 最终代码审查和清理

---

## Task 1: 修复遗留测试失败

3个预存失败：
- FileUtilsTest.ChangeExtension
- MathUtilsTest.AdjustContrast
- ColorProcessorTest.AdjustContrastIncrease

## Task 2: 更新README

创建完整的README.md，包含：
- 项目介绍
- 功能列表
- 安装说明
- 使用说明
- 构建说明
- 许可证

## Task 3: 创建Inno Setup安装脚本

创建installer/setup.iss，支持：
- 安装桌面工具
- 安装DLL插件到REFramework目录
- 创建开始菜单快捷方式
- 卸载程序

## Task 4: 最终清理

- 清理TODO注释
- 统一代码风格
- 更新.gitignore
