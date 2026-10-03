# ECG 项目目录与功能说明

整理日期：2026-10-03  
浏览位置：`C:\STUDY\ecg5`  
项目根目录：`C:\STUDY\ecg5\ptbxl-ecg-analysis-main`

> 本文件依据当前目录、README、配置和代码入口整理。文件夹内主要是一个 ECG 项目，各子目录是同一项目的不同模块和实验成果。文中实验成绩、编译状态来自已有项目记录；本次整理没有重新训练、运行测试或验证硬件。

## 1. 这个项目总体是做什么的

这是一个基于 PTB-XL 数据集的十二导联心电图五类多标签识别项目，覆盖数据处理、神经网络训练、网页展示、预测接口、模型压缩和 STM32 部署。

输入为 10 秒、100 Hz 的十二导联心电信号，单条模型输入形状为 `[1,12,1000]`。输出五类概率，各类独立使用阈值判断，一条记录可以同时属于多个类别。

| 输出缩写 | 中文含义 |
|---|---|
| NORM | 正常心电 |
| MI | 心肌梗死 |
| STTC | ST/T 改变 |
| CD | 传导异常 |
| HYP | 心肌肥厚 |

项目有三种主要使用方式：

1. **算法实验**：使用数据库训练、比较和评估不同神经网络。
2. **电脑展示**：通过网页、命令行或 API 查看波形和预测结果。
3. **单片机演示**：电脑发送数据库波形，STM32 运行压缩后的模型并返回分类结果。

## 2. 总体工作流程

```text
PTB-XL 原始数据
  → 读取波形和诊断标签
  → 按患者划分训练、验证和测试集
  → 滤波、标准化与缓存
  → CNN / ResNet / 其他模型训练与评估
      ├─ 电脑端预测 → 网页、API、波形与解释图
      └─ 知识蒸馏 → 小型 CNN → ONNX → INT8 量化
          → STM32Cube.AI 转换 → Keil 工程 → HEX 固件
          → 串口发送波形 → 单片机返回五类结果

Chapman 外部数据 → 标签审计和信号适配 → 冻结模型外部评估
```

知识蒸馏是让较小的“学生模型”学习“教师模型”的输出；INT8 量化是用低精度数值表示模型，以适应单片机的存储和计算资源。

## 3. 一级目录分别负责什么

以下路径均相对于项目根目录 `ptbxl-ecg-analysis-main`。

| 目录 | 主要用途 | 适合什么时候查看 |
|---|---|---|
| `src/` | 核心算法与可复用功能 | 理解或修改数据处理、模型、训练和预测 |
| `scripts/` | 调用核心功能的执行脚本 | 下载数据、启动实验、导出模型或生成报告 |
| `app/` | Streamlit 网页应用 | 在电脑网页中展示波形和分析结果 |
| `api/` | FastAPI 预测服务 | 让其他程序通过接口调用模型 |
| `firmware/` | STM32 通信与嵌入式适配源码 | 理解串口协议和板端程序 |
| `configs/` | 默认配置 | 修改数据路径、滤波参数和训练参数 |
| `data/` | 数据位置和下载说明 | 配置完整数据库 |
| `tests/` | 自动化测试代码 | 检查核心功能修改后是否正常 |
| `outputs/` | 模型、指标、图表和部署成果 | 使用已训练模型、查看实验结果、获取烧录包 |
| `md/` | 中文专题说明 | 系统了解训练、指标、部署和外部验证 |
| `.git/` | Git 版本管理元数据 | 由 Git 工具管理，不是业务程序 |

## 4. 核心代码 src：算法具体放在哪里

| 模块/文件 | 具体功能 |
|---|---|
| `src/config.py` | 加载 YAML 配置，合并可选的本机 `local.yaml`，处理项目路径 |
| `src/data/ptbxl.py` | PTB-XL 数据读取、元数据和诊断标签处理 |
| `src/data/dataset.py` | 按需读取波形，组织模型需要的信号和多标签数据 |
| `src/signal_processing/filters.py` | 去除基线漂移、带通滤波、可选陷波与标准化 |
| `src/signal_processing/rpeak.py` | 检测心电信号中的 R 峰 |
| `src/signal_processing/hrv.py` | 根据心搏间隔计算心率变异性相关指标 |
| `src/models/cnn1d.py` | 一维卷积神经网络，包含轻量分类模型的结构 |
| `src/models/resnet1d.py` | 一维残差神经网络，用于分类模型对比和训练 |
| `src/models/alternatives.py` | 其他备选网络结构 |
| `src/training/data.py` | 训练数据划分、预处理缓存和训练集标准化统计 |
| `src/training/train.py` | 模型训练主逻辑 |
| `src/training/strategies.py` | 训练策略相关实现 |
| `src/training/distillation.py` | 教师输出与学生数据对齐、多标签蒸馏损失 |
| `src/training/evaluate.py` | 模型评估流程 |
| `src/training/metrics.py` | 分类指标计算 |
| `src/inference/predict.py` | 加载模型并进行预测 |
| `src/inference/ensemble.py` | 将多个模型的预测概率进行集成 |
| `src/inference/serial_link.py` | 电脑端串口连接和通信 |
| `src/inference/serial_simulator.py` | 串口模拟相关功能，便于软件流程调试 |
| `src/explainability/saliency.py` | 使用梯度与输入计算模型对时间位置的响应 |
| `src/visualization/ecg_plot.py` | 绘制单导联、十二导联、滤波对比、R 峰和 RR 间隔图 |
| `src/visualization/metrics_plot.py` | 绘制类别分布、训练曲线、ROC 和混淆矩阵 |
| `src/visualization/saliency_plot.py` | 绘制模型响应可视化图 |
| `src/external_validation/chapman.py` | Chapman 标签映射、单位转换和采样率转换 |
| `src/external_validation/chapman_audit.py` | 审计外部诊断代码，区分明确标签与未知标签 |

各目录中的 `__init__.py` 用于组织 Python 包，通常不是用户直接启动的入口。

## 5. scripts：每个脚本是做什么的

### 5.1 数据下载与检查

| 脚本 | 用途 |
|---|---|
| `download_ptbxl.py` | 下载 PTB-XL 1.0.3 的 100 Hz 数据并校验 SHA-256 |
| `prepare_data.py` | 检查 PTB-XL 文件，生成官方患者隔离划分清单 |
| `inspect_record.py` | 查看指定记录并保存波形和信号分析图 |
| `download_chapman.py` | 下载 Chapman-Shaoxing/Ningbo 官方数据压缩包 |
| `download_chapman_parallel.py` | 使用分段并行、可续传方式下载外部数据压缩包 |
| `download_chapman_s3.py` | 从官方公开 AWS 镜像下载并逐文件校验 |

三个 Chapman 下载脚本是不同下载方式的入口，不代表三套分类模型。

### 5.2 训练、优化与模型比较

| 脚本 | 用途 |
|---|---|
| `train_baseline.py` | 训练五类分类基线模型的入口 |
| `evaluate_model.py` | 在官方第 10 折测试集上评估训练好的检查点 |
| `run_optimization.py` | 运行可恢复的优化/消融实验，并更新实验报告 |
| `run_further_optimization.py` | 第二轮消融、模型集成及交叉验证相关实验 |
| `refine_ensemble_thresholds.py` | 在不改变集成权重的情况下改进分类阈值搜索 |
| `run_cross_validation.py` | 进行患者隔离的四折比较，固定第 9 折作校准集 |
| `run_diagnostic_only.py` | 剔除五类诊断标签全零的记录后重新训练 ResNet |
| `run_distillation.py` | 组织多随机种子的 CNN 蒸馏实验及 INT8、Cube.AI 分析 |

### 5.3 预测、解释与接口调用

| 脚本 | 用途 |
|---|---|
| `predict_record.py` | 输出指定 PTB-XL 记录的五类预测概率 |
| `explain_record.py` | 为单条记录生成模型时间响应图 |
| `call_api.py` | 将真实 PTB-XL 心电记录发送给已启动的本地预测 API |

### 5.4 STM32 导出、打包与演示

| 脚本 | 用途 |
|---|---|
| `prepare_stm32.py` | 将冻结的单模型导出并校验，为 STM32Cube.AI 准备输入 |
| `quantize_cubeai.py` | 使用训练集校准数据量化导出的模型 |
| `package_serial_demo.py` | 固定部署配置，准备验证波形和串口演示工程 |
| `ecg_stm32_viz.py` | 电脑端波形与串口分类界面，发送信号并展示返回结果 |
| `verify_student_firmware.py` | 构建学生模型固件，并验证电脑端 C 实现 |
| `report_cubeai.py` | 汇总 Cube.AI、量化和固件编译结果 |
| `report_student_validation.py` | 汇总学生固件资源、转换核对和 PTB-XL 指标 |

### 5.5 外部验证

| 脚本 | 用途 |
|---|---|
| `evaluate_external_chapman.py` | 使用冻结模型进行 Chapman 外部评估 |
| `evaluate_audited_chapman.py` | 结合诊断代码审计和标签掩码，评估外部完整队列 |
| `verify_chapman_results.py` | 根据已保存的预测概率独立重算外部指标 |

外部结果应结合 `md/Chapman_冻结模型外部验证总结.md` 理解。没有标出的诊断不自动等于阴性，不能直接将所有记录按完整五类标签计算成绩。

## 6. 网页、API 和启动文件

| 文件 | 作用 | 使用前提 |
|---|---|---|
| `app/streamlit_app.py` | ECG 网页分析与展示入口 | Python 依赖、数据路径和模型配置正确 |
| `api/main.py` | FastAPI 服务入口 | 启动服务并准备对应模型 |
| `start_ecg_student_viz.cmd` | 启动学生模型串口界面，明确指定蒸馏导出的 `serial_manifest.json` | 项目 `.venv`、相关依赖和演示文件可用 |
| `start_ecg_stm32_viz.cmd` | 使用脚本默认参数启动通用串口界面 | 默认部署配置可用 |
| `start_stm32cubemx_ascii.cmd`、`.ps1` | STM32CubeMX 的配套启动辅助文件 | 本机已安装并配置相关工具 |

当前学生固件演示应使用 `start_ecg_student_viz.cmd`，以匹配学生模型的预处理、类别和阈值。网页/API 默认模型与学生固件分别配置，并不会因为生成了学生固件而自动同步。

## 7. firmware：板端程序的分工

文件位于 `firmware/ecg_serial/`。

| 文件 | 作用 |
|---|---|
| `main_spl.c` | 基于标准外设库的板端主程序入口 |
| `ecg_protocol.c`、`ecg_protocol.h` | ECG 串口通信协议实现与声明 |
| `ecg_uart.c`、`ecg_uart.h` | UART 收发适配 |
| `ecg_tick.h` | 计时相关接口 |
| `ecg_deployment_config.h` | 部署配置头文件 |

这里是协议与适配源码；包含模型代码、库和 Keil 工程的完整学生工程位于 `outputs/distillation/stm32_student_project/`。

## 8. outputs：已有成果分别是什么

| 子目录 | 主要内容与用途 |
|---|---|
| `checkpoints/` | 正式 CNN、ResNet 模型权重，供电脑端加载 |
| `metrics/` | 训练过程和测试指标 JSON |
| `predictions/` | 测试记录的逐条预测 CSV |
| `figures/` | 训练曲线、ROC、混淆矩阵和类别分布图 |
| `optimization/` | 优化实验保留的模型成果 |
| `further_optimization/` | 后续优化的交叉验证汇总、最终选择和测试记录 |
| `cross_validation/` | 交叉验证协议和汇总 |
| `diagnostic_only/` | 仅保留有效五类诊断记录的实验配置、汇总及模型 |
| `distillation/` | 学生模型、蒸馏实验汇总、导出文件和 STM32 工程 |
| `external_validation/` | Chapman 评估、标签审计、概率文件和量化差异分析 |
| `stm32/` | 保留的 CNN FP32 ONNX、权重和导出一致性记录 |
| `board_release/` | 当前学生固件烧录包及交付说明 |

### 8.1 当前学生模型相关位置

| 路径 | 用途 |
|---|---|
| `outputs/distillation/distilled_seed44/checkpoint/cnn1d_best.pt` | 当前部署候选的 PyTorch 学生模型 |
| `outputs/distillation/distilled_export/` | ONNX、部署配置、串口清单、演示数据及许可说明 |
| `outputs/distillation/stm32_student_project/` | 包含模型代码和依赖的 STM32 学生源码工程 |
| `outputs/board_release/seed44_int8_stm32f407ve/` | 面向 STM32F407VE 的当前烧录交付目录 |

### 8.2 烧录交付目录内的文件

| 文件 | 用途 |
|---|---|
| `ECG_Student_INT8_STM32F407VE.hex` | 已编译的板端固件，用于烧录 |
| `ECG_Student_INT8_源码工程.zip` | 完整源码工程压缩包，用于查看和重新编译 |
| `使用说明.txt` | 烧录、晶振、接线、串口参数和启动方法 |
| `release_manifest.json` | 资源占用、文件哈希和检查记录 |

当前固件配置为 STM32F407VE、8 MHz 外部晶振、168 MHz 主频。接线和硬件条件以该目录的使用说明为准。

### 8.3 常见文件后缀如何理解

| 后缀 | 含义 |
|---|---|
| `.pt` | PyTorch 模型权重或检查点 |
| `.onnx` | 用于跨运行环境推理与转换的模型 |
| `.json` | 实验指标、配置、部署清单等结构化记录 |
| `.csv` | 表格形式的预测或统计 |
| `.npz` | NumPy 压缩数组数据，如预测概率 |
| `.png` | 图表或可视化结果 |
| `.c`、`.h` | C 源码与头文件 |
| `.uvprojx` | Keil 工程文件 |
| `.hex` | 可供烧录工具使用的固件文件 |

## 9. 配置、依赖与测试

### 9.1 配置和数据

- `configs/default.yaml`：默认使用 100 Hz，训练折 1–8、验证折 9、测试折 10，并设置滤波、模型、训练和输出参数。
- `configs/local.yaml`：可选的本机覆盖配置；当前顶层清单中未见该文件，不能假设已经配置。
- `data/README.md`：记录数据库位置与下载方式。历史文档提到 `D:\datasets\ptb-xl`，本次未检查该外部路径是否存在。
- 项目默认数据路径为 `data/ptb-xl`；完整原始数据库并非由当前目录说明文件自动提供。随包的 128 条串口演示样本与完整训练数据不同。

### 9.2 依赖文件

| 文件 | 安装内容 |
|---|---|
| `requirements.txt` | NumPy、Pandas、SciPy、WFDB、绘图和配置等基础依赖 |
| `requirements-train.txt` | 基础依赖，加 PyTorch、scikit-learn |
| `requirements-app.txt` | 训练依赖，加 Streamlit |
| `requirements-api.txt` | 应用依赖，加 FastAPI、HTTPX |
| `requirements-stm32.txt` | ONNX、ONNX Runtime、串口库 |
| `requirements-dev.txt` | 基础依赖和 pytest |

当前项目一级目录中未见 `.venv`。启动器引用 `.venv\Scripts\python.exe`，因此存在启动文件并不代表当前机器已具备完整运行环境。

### 9.3 测试目录

`tests/` 包括阶段 A–E 的功能测试，以及训练策略、诊断数据筛选、知识蒸馏、交叉验证、外部标签审计、STM32 导出和串口协议测试。它们用于回归检查，不是模型训练入口。

## 10. 中文文档应该按什么顺序看

| 文档 | 阅读目的 |
|---|---|
| 根目录 `README.md` | 快速了解总体目标、输入输出、当前状态和启动方式 |
| `md/README.md` | 理解环境、运行、目录结构和协作约定 |
| `md/模型训练与优化.md` | 理解数据筛选、训练流程、优化和蒸馏 |
| `md/各模型整体与逐类指标总表.md` | 比较不同模型和各类别表现 |
| `md/STM32部署与串口验证.md` | 查看导出、固件、通信协议和实板验证要求 |
| `md/Chapman_冻结模型外部验证总结.md` | 理解外部数据评估与标签口径 |

## 11. 想做某件事，应该从哪里开始

| 目标 | 建议入口 |
|---|---|
| 了解整个项目 | 本文件 → 根目录 README |
| 查看模型成绩 | 指标总表 → 对应 outputs 中的 JSON |
| 研究模型结构 | `src/models/` |
| 修改信号预处理 | `src/signal_processing/` 和配置文件 |
| 重新训练模型 | 训练文档 → 下载/准备数据 → 训练脚本 |
| 查看单条 ECG | `scripts/inspect_record.py` |
| 预测单条 ECG | `scripts/predict_record.py` |
| 在网页展示 | `app/streamlit_app.py` |
| 给其他软件提供预测 | `api/main.py` |
| 演示当前学生固件 | 烧录包使用说明 → `start_ecg_student_viz.cmd` |
| 修改板端程序 | 完整学生 Keil 工程和 `firmware/ecg_serial/` |
| 研究外部验证 | Chapman 总结 → `evaluate_audited_chapman.py` |

## 12. 当前完成程度与结果口径

根据已有 README 和部署记录：

- 已保存训练模型、模型比较、蒸馏与量化结果。
- 当前部署候选是 seed44 蒸馏 CNN INT8；记录的 PTB-XL 测试 Macro AUROC 为 0.924276，Macro F1 为 0.749441。
- 已提供 Cube.AI 转换结果、完整源码工程和 HEX；文档记录编译为 0 错误、0 警告。
- 电脑端 C 实现与 INT8 ONNX 的一致性验证，与真实芯片运行验证是不同环节。
- 实板烧录、串口收发、板上推理耗时和运行时栈峰值仍待验证。
- PTB-XL 测试折已用于多轮实验，不是新的盲测；Chapman 的附加指标依赖标签假设，应连同原文说明使用。

本项目定位为课程实验与展示系统。阅读和汇报时应区分“文件已生成”“电脑端验证完成”和“实板验证完成”三种状态。
