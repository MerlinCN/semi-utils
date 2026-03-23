# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

Semi-Utils 是一个照片批量水印处理工具,支持:
- 批量添加 EXIF 水印(相机型号、镜头、光圈、快门、ISO 等)
- 处理照片像素比、图像色彩和质量
- 支持多种水印模板(经典 EXIF、尼康风格、模糊背景等)
- Web UI 界面进行可视化操作
- HEIC 格式转换支持

## 开发环境

### 依赖管理
- 使用 `uv` 管理 Python 项目
- 添加依赖: `uv add <package>`(不要直接编辑 pyproject.toml)
- 运行命令: `uv run <command>`

### 运行项目
```bash
# 启动 Web 服务
uv run python app.py

# 服务默认地址: http://localhost:15050
```

### 代码质量
- 使用 ruff 进行代码检查: `uv run ruff check <file> --fix`
- ⚠️ 不要对整个项目运行 ruff,修改单个文件后再检查该文件

## 核心架构

### 模块结构

**app.py**: Flask Web 应用主入口
- 提供 REST API 和前端界面
- 处理文件上传、配置管理、模板管理
- SSE 流式处理进度反馈
- 多线程并发处理图片(ThreadPoolExecutor)

**processor/**: 图像处理管道系统
- `core.py`: 处理管道核心,基于责任链模式
  - `PipelineContext`: 管道上下文,存储处理器间共享数据
  - `ImageProcessor`: 处理器抽象基类,所有处理器继承此类
  - 自动注册机制: 处理器通过 `__init_subclass__` 钩子自动注册
  - `start_process()`: 执行处理管道的主函数
- `filters.py`: 图像滤镜处理器(模糊、锐化、色彩调整等)
- `generators.py`: 图像生成器(文本、形状、渐变等)
- `mergers.py`: 图像合并器(拼接、叠加等)
- `types.py`: 共享类型定义(Direction, Alignment 枚举)

**core/**: 核心工具模块
- `configs.py`: 配置加载(config.ini 和 pyproject.toml)
- `util.py`: 工具函数
  - EXIF 提取(`get_exif`)使用 exiftool
  - 文件树扫描(`list_files`)
  - HEIC 转 JPEG
  - 模板管理函数
- `jinja2renders.py`: Jinja2 自定义函数
  - `vw()`, `vh()`: 基于图片尺寸的百分比计算
  - `auto_logo()`: 根据相机品牌自动选择 logo
- `logger.py`: 基于 loguru 的日志系统

### 处理流程

1. 用户在 Web UI 选择图片和模板
2. 模板是 JSON 格式,支持 Jinja2 语法动态渲染
3. 后端解析模板生成处理器配置列表
4. 图像处理管道按顺序执行处理器:
   - Generator: 生成新图层(文本、Logo、形状)
   - Filter: 对图层应用效果(模糊、圆角、阴影)
   - Merger: 合并多个图层
5. 最终输出到 output_folder

### 配置文件

`config/config.ini`: 运行时配置
- 服务器设置(host, port, debug)
- 输入/输出文件夹路径
- 图片质量设置
- 当前模板名称

`config/templates/*.json`: 水印模板文件
- JSON 格式的处理器配置数组
- 支持 Jinja2 模板语法,可访问 EXIF 数据

### 关键设计模式

**责任链模式**: 图像处理管道,每个处理器处理后传递给下一个
**策略模式**: 不同的处理器实现不同的图像处理策略
**工厂模式**: 处理器注册表,根据名称动态创建处理器实例

### 注意事项

- import 语句永远放在文件最上面,除非有循环引用
- 避免使用 hasattr 和 getattr
- 减少 try-catch 滥用,尽量 catch 核心会出错的点
- 避免多层嵌套,多使用 early return
- 所有函数添加类型注释和 docstring
- 在 Windows 上使用 PowerShell 语法,文件路径用双引号包裹
