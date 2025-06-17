# 基于逻辑回归的价格趋势预测模型

本项目旨在利用OKX提供的5年期、多时间频率（1D, 1H, 15m）的K线数据，构建一个机器学习模型，以预测ETH（以太坊）和BTC（比特币）在未来特定时间窗口（15/30/60分钟）内的价格方向（上涨/下跌）。

当前版本使用**逻辑回归 (Logistic Regression)** 作为基线模型。

> **备注**: 您在提问时可能使用了"线性回归"，但由于我们的目标是预测"涨/跌"方向（一个分类问题），因此在实际执行中我们采用了更适合此任务的"逻辑回归"模型。

## 项目结构

```
price_trend_attention_model/
├── configs/          # 配置文件 (暂未使用)
├── data/             # 数据目录
│   ├── raw/          # 原始下载数据
│   ├── processed/    # 预处理后数据 (暂未使用)
│   └── featured/     # 特征工程后数据
├── logs/             # 日志、模型评估报告、混淆矩阵图
├── models/           # 保存的已训练模型
├── notebooks/        # Jupyter Notebooks (探索性分析)
├── scripts/          # 主要执行脚本
│   ├── download_history_mark_price.py # 数据下载
│   ├── advanced_feature_engineering.py# 特征工程
│   ├── train_logistic_regression.py   # 模型训练
│   └── predict.py                     # 模型预测
├── pyproject.toml    # 项目依赖
├── uv.lock           # 依赖锁定文件
└── README.md         # 本文档
```

## 工作流：如何复现 (以ETH为例)

请按照以下步骤来完成从数据下载到模型预测的全过程。

### 步骤 0: 环境设置

本项目使用 `uv` 作为包管理器。首先，请同步依赖环境：

```bash
uv sync
```

### 步骤 1: 下载数据

运行以下命令来下载 `ETH-USD-SWAP` 的5年历史K线数据。数据会保存在 `data/raw/` 目录下。

```bash
uv run python3 scripts/download_history_mark_price.py --instId ETH-USD-SWAP
```

*脚本会自动下载 1D, 1H, 15m 三种时间频率的数据。*

### 步骤 2: 特征工程

运行高级特征工程脚本。它会加载原始数据，计算各种技术指标和统计特征，使用VIF进行特征选择，并将最终的特征数据集保存在 `data/featured/` 目录下。

```bash
uv run python3 scripts/advanced_feature_engineering.py
```
*该脚本会同时处理 `BTC` 和 `ETH` 的数据。*

### 步骤 3: 训练模型

现在，我们可以训练逻辑回归模型。你需要指定 `--instId` (资产) 和 `--horizon` (预测窗口)。

*   `--horizon 1`: 预测未来1个15分钟K线
*   `--horizon 2`: 预测未来2个15分钟K线 (30分钟)
*   `--horizon 4`: 预测未来4个15分钟K线 (60分钟)

例如，训练一个用于预测ETH未来15分钟方向的模型：

```bash
uv run python3 scripts/train_logistic_regression.py --instId ETH-USD-SWAP --horizon 1
```

训练完成后，模型将保存在 `models/` 目录，而详细的评估报告和混淆矩阵图将保存在 `logs/` 目录。

### 步骤 4: 使用模型进行预测

使用 `predict.py` 脚本和你刚刚训练好的模型来进行预测。该脚本会自动加载与你指定的资产和视界相匹配的**最新**模型，并对**最新**的一条K线数据进行预测。

例如，使用刚训练的ETH模型进行预测：

```bash
uv run python3 scripts/predict.py --instId ETH-USD_SWAP --horizon 1
```

你将会看到类似以下的输出：

```
INFO: ======================================================================
INFO: 🚀 开始为资产 ETH-USD-SWAP 在视界 h1 上进行预测...
INFO: Step 1: 正在查找并加载最新模型...
INFO: ✅ 成功加载模型: ETH-USD-SWAP_h1_logistic_regression_YYYYMMDD_HHMMSS.joblib
INFO: Step 2: 正在加载特征数据...
INFO: Step 3: 准备最新的数据点进行预测...
INFO:   预测所基于的最新K线时间点: 2024-XX-XX XX:XX:XX
INFO: Step 4: 执行预测...
INFO: 
============================ 预测结果 ============================
INFO:   资产: ETH-USD-SWAP
INFO:   预测时间: 2024-XX-XX XX:XX:XX
INFO:   预测基准K线: 2024-XX-XX XX:XX:XX
INFO:   预测未来窗口: 15 分钟后
INFO:   预测方向: 看涨 (UP)
INFO:   看涨置信度: 65.73%
INFO:   看跌置信度: 34.27%
INFO: ======================================================================
```

## 关于测试集的说明

为了确保模型评估的公正性和有效性，本项目采用严格的**时间序列分割**方法来划分训练集和测试集。

*   **数据集来源**: `data/featured/` 中的特征数据，已按时间从旧到新排序。
*   **训练集 (Training Set)**: 使用数据集中**前80%** 的连续数据。
*   **测试集 (Test Set)**: 使用数据集中**后20%** 的连续数据。

这种方法保证了模型只在"过去"的数据上训练，并在"未来"的、完全未见过的数据上进行测试，这对于评估金融预测模型至关重要。

## 特征工程详解 (Feature Formulas)

以下是在 `advanced_feature_engineering.py` 脚本中生成的核心特征及其计算公式。

### 1. 目标变量 (Target Variable)

- **Mid-price-2**: `(high + low) / 2`
    - 这是我们用来判断未来价格方向的基础。
- **Label_h (预测标签)**: `(Mid-price-2.shift(-h) > Mid-price-2).astype(int)`
    - 如果未来第 `h` 个时间点的 `Mid-price-2` 高于当前，标签为1 (看涨)，否则为0 (看跌)。

### 2. OHLC 衍生特征

- **Range**: `high - low`
- **Body**: `abs(close - open)`
- **Upper_Wick**: `high - max(open, close)`
- **Lower_Wick**: `min(open, close) - low`

### 3. 技术指标 (Technical Indicators)

- **SMA_N (简单移动平均)**: `close.rolling(window=N).mean()`
- **EMA_N (指数移动平均)**: `close.ewm(span=N, adjust=False).mean()`
- **RSI_14 (相对强弱指数)**:
    - `delta = close.diff()`
    - `gain = (delta.where(delta > 0, 0)).ewm(alpha=1/14).mean()`
    - `loss = (-delta.where(delta < 0, 0)).ewm(alpha=1/14).mean()`
    - `rs = gain / loss`
    - `RSI_14 = 100 - (100 / (1 + rs))`
- **MACD (移动平均收敛散度)**:
    - `MACD_line = EMA(close, 12) - EMA(close, 26)`
    - `Signal_line = EMA(MACD_line, 9)`
    - `MACD_hist = MACD_line - Signal_line`
- **Bollinger Bands (布林带)**:
    - `Middle_Band = SMA(close, 20)`
    - `Upper_Band = Middle_Band + 2 * close.rolling(20).std()`
    - `Lower_Band = Middle_Band - 2 * close.rolling(20).std()`
- **ATRr_14 (平均真实波幅)**:
    - `tr = max(high - low, abs(high - close.shift()), abs(low - close.shift()))`
    - `ATRr_14 = EMA(tr, 14)`
- **MOM_10 (动量)**: `close.diff(10)`

### 4. 高级统计特征

- **Log_Return (对数收益率)**: `log(close / close.shift(1))`
- **RV (已实现波动率)**: `Log_Return.rolling(window=20).std() * sqrt(20)`
- **Parkinson_Vol (帕金森波动率)**: `sqrt(0.5 * log(high / low)^2).rolling(window=20).mean()`
- **Skewness (偏度)**: `Log_Return.rolling(window=60).skew()`
- **Kurtosis (峰度)**: `Log_Return.rolling(window=60).kurt()`

### 5. 多频率融合特征 (以15分钟数据为核心)

- **MidPrice2_vs_1h (相对1小时中间价)**: `(MidPrice2_15m - MidPrice2_1h) / MidPrice2_1h`
- **MidPrice2_vs_1d (相对1天中间价)**: `(MidPrice2_15m - MidPrice2_1d) / MidPrice2_1d`
- **RSI_15min_vs_1h (相对1小时RSI)**: `RSI_15m - RSI_1h`
- **RSI_15min_vs_1d (相对1天RSI)**: `RSI_15m - RSI_1d`
