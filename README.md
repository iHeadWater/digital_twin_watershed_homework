# CAMELS 数据与 LSTM-CAMELS 流量预测作业

## 评分标准

本项目作为《数字孪生流域》课程作业，评分基于固定训练/验证期下的验证集表现与覆盖流域数：

- 训练期：1990-09-01 ~ 2000-08-31（与示例一致）
- 验证期：2000-09-01 ~ 2005-08-31（与示例一致）
- 及格线：验证集 NSE ≥ 0.40（如课堂另有通知，以课堂口径为准）
- 原则：在上述训练/验证期下，最终评估阶段达到及格线的流域越多，得分越高

| 等级 | 分数区间 | 完成标准 |
|------|----------|----------|
| **A** | 90-100分 | >200 个流域达到及格线。综合考虑流域数量与平均 NSE，流域数量越多、平均 NSE 越高，得分越高。 |
| **B** | 80-89分 | 41~200 个流域达到及格线。综合考虑流域数量与平均 NSE，流域数量越多、平均 NSE 越高，得分越高。 |
| **C** | 70-79分 | 21~40 个流域达到及格线。综合考虑流域数量与平均 NSE，流域数量越多、平均 NSE 越高，得分越高。 |
| **D** | 60-69分 | 5~20 个流域达到及格线即达及格。综合考虑流域数量与平均 NSE，流域数量越多、平均 NSE 越高，得分越高。 |
| **F** | <60分 | 无流域达到及格线或无有效结果。 |

*每个人随机挑选流域，不能完全重复随机种子和重复流域号，一经发现两者完全重复，会检查两人代码，完全重复，本次作业记为0分*


## 概述

本项目演示如何基于 CAMELS 数据集实现 LSTM-CAMELS 流量预测：
- 获取与读取 CAMELS 数据（NetCDF/feather），并采用 `xarray` 懒加载与 `pandas` 读取属性表；
- 使用 PyTorch 实现两层 LSTM 的预测模型与数据集封装；
- 构建训练/验证/测试流程，计算 NSE 等指标，并进行可视化。

支持在服务器环境运行。

## 快速开始

### 方式一：服务器运行（推荐）

- 登录平台服务器http://jupyterhub.waterism.com:666/ ，输入用户名和密码，直接运行本项目笔记本。
- 平台已预置运行环境与依赖，无需安装或配置。
- 平台已预设 `hydrodataset` 数据目录，可直接运行 `1_获取数据.ipynb` 开始。
- 建议先在 `1_获取数据.ipynb` 中执行 ROOT_DIR 检查单元，确认路径有效。

> 如需查看或确认数据目录，请参考 `1_获取数据.ipynb` 前置章节中的说明与检查代码。

### 方式二：本地 IDE 运行

- 环境要求
  - Windows 10/11、macOS 或 Linux
  - Python 3.9 ~ 3.11（推荐 3.10）
  - 可选：CUDA 11.8+（如需 GPU，与 PyTorch 版本匹配）

- 使用 conda 安装

```bash
conda create -n camels python=3.10 -y
conda activate camels
pip install -r requirements.txt
```

- 使用 venv 安装

Windows PowerShell：

```powershell
python -m venv .venv
.\\.venv\\Scripts\\Activate.ps1
pip install -r requirements.txt
```

macOS/Linux：

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

- 配置 hydrodataset 数据目录（本地必做）
  - 创建/编辑配置文件：
    - Windows: `%USERPROFILE%\\.hydrodataset\\settings.txt`
    - macOS/Linux: `~/.hydrodataset/settings.txt`
  - 文件内容：仅一行，填写 CAMELS 数据根目录的绝对路径（不加引号）。示例：
    - Windows: `D:\\data\\hydrodataset`
    - macOS/Linux: `/data/hydrodataset`
  - 验证：

```python
import hydrodataset
print("数据根目录:", hydrodataset.ROOT_DIR)
```

  - 目录结构应包含：
    - `ROOT_DIR/camels/camels_us/camels_streamflow.nc`
    - `ROOT_DIR/camels/camels_us/camels_daymet_forcing.nc`
    - `ROOT_DIR/camels/camels_us/camels_attributes_v2.0.feather`

- 数据与包下载
  - CAMELS 数据下载地址：<https://zenodo.org/records/15529996>
  - hydrodataset 包：<https://github.com/iHeadWater/hydrodataset>
  - 下载后将数据解压至上文配置的 `ROOT_DIR`，保持目录结构一致。

- 运行笔记本

```bash
jupyter notebook
```

打开 `1_获取数据.ipynb`，按顺序运行。首次运行请先完成“配置 hydrodataset 数据目录”，并先运行“本地数据文件校验”单元。

- 常见问题
  - `netCDF4` 安装失败：优先使用 `requirements.txt`；仍失败可尝试 `conda install -c conda-forge netcdf4`。
  - Windows 路径问题：避免中文与空格；`settings.txt` 仅填写绝对路径。
  - 找不到数据：检查 `settings.txt` 路径与 `camels/camels_us/*.nc` 及属性 `*.feather` 是否存在。

## 教程结构

### 学生任务

**重要**：`2_PyTorch实现LSTM-CAMELS与训练.ipynb` 中的 `train_epoch` 函数已置空，需要学生自行补充完整训练循环逻辑，这是本次作业的核心任务之一。

train_epoch函数需实现的功能：
1. 设置模型为训练模式
2. 遍历数据加载器中的每个批次
3. 对每个批次执行：清零梯度、前向传播、计算损失、反向传播、更新参数
4. （可选）使用 tqdm 显示训练进度和损失值

完成此函数后方可正常训练模型并获得验证集 NSE 指标。

### 第一部分：获取数据 (`1_获取数据.ipynb`)
- 本地/服务器两种运行方式说明；配置 `hydrodataset` 路径与 ROOT_DIR 校验；
- 本地数据文件校验单元：检查 `camels_streamflow.nc`、`camels_daymet_forcing.nc`、`camels_attributes_v2.0.feather` 是否存在；
- 读取流量、强迫与属性数据；
- 示例：按 `basin`/`time` 选择与可视化（`xarray` 懒加载）。

### 第二部分：PyTorch 实现 LSTM-CAMELS与训练 (`2_PyTorch实现LSTM-CAMELS与训练.ipynb`)
- `CamelsDataset`：计算/复用均值与标准差，局地归一化强迫/属性/流量，构造样本查找表；
- `local_denormalization`：用于将 `streamflow` 预测从标准化空间反变换回原尺度；
- `LSTM_CAMELS`：两层 LSTM，尾部全连接，仅用最后时刻隐状态进行回归预测；
- 流域选择：默认以单个索引 `i` 选择一个流域；备选支持“前 `basins_num` 个”与“索引区间 `[start_idx:end_idx)`”多流域选择；
- 训练与评估函数（`train_epoch`/`eval_model`）；
- 数据拆分：训练/验证/测试时间段与变量选择；
- 可调超参数：`sequence_length`、`batch_size`、`hidden_size`、`learning_rate`、`n_epochs`；
- 训练循环与验证集 NSE 输出；
- 测试集评估与结果曲线绘制；
- 随机种子：建议每位同学自定义不同的 `seed`（见 `2_PyTorch实现LSTM-CAMELS与训练.ipynb` 中的 `seed` 变量），用于数据拆分、初始化与打乱，避免作业结果雷同。

### 同学们在运行时注意把两个notebook合并后运行，不要分开运行，分开运行会导致变量未定义无法正常运行。
### 在服务器上运行的同学注意选择Python（tutorial）内核，不要选择其他内核。
---

***本项目笔记本以教学示例为主，你可自由更换变量组合、调参或扩展模型结构；评分仍以流域覆盖与时段长度以及结果指标为主。***
