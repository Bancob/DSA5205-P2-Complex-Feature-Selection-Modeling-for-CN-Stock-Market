# DSA5205 量化金融数据准备工具

## 项目概述

本项目是一个基于Tushare Pro API的量化金融数据获取和整合系统，专为DSA5205课程设计。该系统能够自动抓取、清洗、整合多维度的中国金融市场数据，并生成完整的数据字典，为后续的量化分析和机器学习建模提供高质量的数据基础。

### 核心特点

- **多维度数据整合**：覆盖个股、ETF、市场指数、宏观经济等多个维度
- **时间序列对齐**：使用先进的`merge_asof`算法确保不同频率数据的准确对齐
- **自动化数据字典**：生成包含中英文说明的完整字段文档
- **财务数据去重**：智能处理财务报告更新，保留最新且完整的数据
- **模块化设计**：清晰的数据流程，易于扩展和维护

---

## 系统架构

### 数据流程图

```
原始数据抓取 → 数据清洗去重 → 时间序列对齐 → 多源数据整合 → 最终输出
     ↓              ↓                ↓              ↓            ↓
  raw/       processed/         宏观数据        full/      列名字典
```

### 目录结构

```
A2/
├── dsa5205_p2_data_prep.ipynb    # 主程序Notebook
├── README.md                      # 本文档
├── data/
│   ├── trade_cal.csv             # 交易日历
│   ├── column_description.csv    # 数据字典
│   ├── benchmark/                # 基准指数数据
│   │   ├── 000300.SH_daily.csv  # 沪深300指数
│   │   └── 000906.SH_daily.csv  # 中证2000指数
│   ├── stock/                    # 个股数据
│   │   ├── raw/                 # 原始抓取数据
│   │   ├── processed/           # 整合后数据（无宏观）
│   │   └── full/                # 完整数据（含宏观）
│   ├── etf/                     # ETF数据
│   │   ├── raw/
│   │   ├── processed/
│   │   └── full/
│   └── macro/                   # 宏观和市场数据
│       ├── raw/
│       └── processed/
```

---

## 数据维度详解

### 一、个股数据（3只股票）

**标的股票**：
- 招商银行 (600036.SH) - 银行业龙头
- 比亚迪 (002594.SZ) - 新能源汽车龙头
- 贵州茅台 (600519.SH) - 白酒行业龙头

**数据类型**（7个维度）：

| 前缀 | 维度名称 | 数据频率 | 来源接口 | 字段数 | 说明 |
|------|---------|---------|---------|-------|------|
| `indiv_d_` | 复权行情 | 日度 | `ts.pro_bar` | 12 | 后复权价格、成交量额等 |
| `indiv_din_` | 每日指标 | 日度 | `pro.daily_basic` | 22 | 市盈率、换手率、市值等 |
| `indiv_fin_` | 财务指标 | 季度 | `pro.fina_indicator` | 154 | 财务报表核心指标 |
| `indiv_sh_` | 股东人数 | 季度 | `pro.stk_holdernumber` | 4 | 股东户数变化 |
| `indiv_chip_` | 筹码分布 | 日度 | `pro.cyq_perf` | 9 | 成本分布、胜率分析 |
| `indiv_techin_` | 技术因子 | 日度 | `pro.stk_factor` | 120+ | MACD、KDJ、RSI等 |
| `indiv_margin_` | 融资融券 | 日度 | `pro.margin_detail` | 8 | 融资融券余额及变动 |

**关键特性**：
- ✅ 财务数据自动去重（保留最新公告的完整报告）
- ✅ 股东人数自动去重
- ✅ 财务数据向前扩展180天以确保覆盖
- ✅ 使用`merge_asof`向后填充方法对齐公告日期

### 二、ETF数据（2只基金）

**标的ETF**：
- 华泰柏瑞沪深300ETF (510300.SH)
- 南方中证500ETF (510500.SH)

**数据类型**（5个维度）：

| 前缀 | 维度名称 | 数据频率 | 来源接口 | 说明 |
|------|---------|---------|---------|------|
| `indiv_d_` | ETF日线 | 日度 | `pro.fund_daily` | 收盘价、净值等 |
| `indiv_adj_d_` | 复权因子 | 日度 | `pro.fund_adj` | 用于计算复权价格 |
| `indiv_ind_` | 标的指数行情 | 日度 | `pro.index_daily` | 沪深300/中证500指数 |
| `indiv_din_` | 指数每日指标 | 日度 | `pro.index_dailybasic` | 指数PE、PB等 |
| `indiv_techin_` | 指数技术因子 | 日度 | `pro.idx_factor_pro` | 指数技术指标 |

### 三、市场数据（Market）

**大盘指数**：
- 上证指数 (000001.SH)
- 深证成指 (399001.SZ)

**数据类型**（3个维度）：

| 前缀 | 维度名称 | 数据频率 | 来源接口 | 说明 |
|------|---------|---------|---------|------|
| `market_din_` | 大盘指标 | 日度 | `pro.index_dailybasic` | 上证、深证的PE、PB等 |
| `market_flow_` | 资金流向 | 日度 | `pro.moneyflow_hsgt` | 沪深港通资金流向 |
| `market_margin_` | 融资融券汇总 | 日度 | `pro.margin` | 两市融资融券汇总 |

### 四、宏观数据（Macro）

**数据类型**（8个维度）：

| 前缀 | 维度名称 | 数据频率 | 来源接口 | 关键指标 |
|------|---------|---------|---------|---------|
| `macro_shibor_` | SHIBOR利率 | 日度 | `pro.shibor` | 隔夜、1周、1月至1年利率 |
| `macro_gdp_` | GDP | 季度 | `pro.cn_gdp` | GDP及三产业增速 |
| `macro_cpi_` | CPI | 月度 | `pro.cn_cpi` | 全国/城市/农村CPI |
| `macro_ppi_` | PPI | 月度 | `pro.cn_ppi` | 工业生产者价格指数 |
| `macro_m_` | 货币供应 | 月度 | `pro.cn_m` | M0、M1、M2及增速 |
| `macro_sf_` | 社融 | 月度 | `pro.sf_month` | 社会融资规模 |
| `macro_ust_` | 美债收益率 | 日度 | `pro.us_trycr` | 1/2/3/5/7/10/20/30年期 |
| `macro_im_` | 国际指数 | 日度 | `pro.index_global` | 标普500(SPX)、恒生(HSI) |

---

## 技术实现细节

### 1. 交易日历生成

```python
# 获取上交所和深交所交易日历
cal_sh = pro.trade_cal(exchange='SSE', start_date=start_date, end_date=end_date)
cal_sz = pro.trade_cal(exchange='SZSE', start_date=start_date, end_date=end_date)

# 验证一致性并使用上交所日历
assert cal_sh['cal_date'].equals(cal_sz['cal_date'])
cal = cal_sh[cal_sh['is_open'] == 1]
```

### 2. 财务数据去重算法

```python
def drop_expired_fin_report(df):
    """
    财务报告去重逻辑：
    1. 相同end_date: 保留最新ann_date的报告
    2. 相同end_date且ann_date: 保留NaN最少的报告
    """
    df['nan_count'] = df.isna().sum(axis=1)
    df = df.sort_values(
        by=['indiv_fin_end_date', 'ann_date', 'nan_count'], 
        ascending=[True, False, True]
    )
    df = df.drop_duplicates(subset=['indiv_fin_end_date'], keep='first')
    return df.sort_values('ann_date')
```

**为什么需要去重？**
- 上市公司会发布快报、年报、修正报告等多个版本
- 同一季度可能有多次公告，数据完整性不同
- 保留最新且最完整的数据确保分析准确性

### 3. 时间序列对齐策略

#### 3.1 日度数据对齐（直接合并）

```python
# 相同频率的日度数据使用inner join
df = df.merge(
    df_din.drop(columns=['indiv_din_ts_code', 'indiv_din_trade_date']), 
    left_index=True, 
    right_index=True, 
    how='left'
)
```

#### 3.2 财务数据对齐（向后填充）

```python
# 财务数据使用merge_asof，向后查找最近的已公告数据
df = pd.merge_asof(
    df.sort_index(),                    # 日度数据（trade_date为索引）
    df_fin.sort_values('ann_date'),     # 财务数据（按公告日期排序）
    left_index=True,                     # 使用trade_date作为左侧键
    right_on='ann_date',                 # 使用ann_date作为右侧键
    direction='backward'                 # 向后填充：找≤trade_date的最新公告
)
```

**示例**：
```
交易日     财务报告(end_date)  公告日(ann_date)  结果
2024-03-15    2023Q4         2024-03-10    ✓ 匹配到2023Q4
2024-04-20    2024Q1         2024-04-25    ✗ 尚未公告，仍用2023Q4
2024-04-26    2024Q1         2024-04-25    ✓ 匹配到2024Q1
```

#### 3.3 宏观数据对齐

**季度数据转换**：
```python
# GDP等季度数据：将季度转换为季度末日期
df_gdp['trade_date'] = pd.PeriodIndex(
    df_gdp['quarter'].astype(str), 
    freq='Q'
).to_timestamp(how='end')  # 20231 → 2023-03-31
```

**月度数据转换**：
```python
# CPI等月度数据：将月份转换为月末日期
df_cpi['trade_date'] = pd.to_datetime(
    df_cpi['month'], 
    format='%Y%m'
) + pd.offsets.MonthEnd(0)  # 202401 → 2024-01-31
```

**合并策略**：
```python
# 所有宏观数据使用merge_asof向后填充
df_macro = pd.merge_asof(
    df_macro.sort_values('trade_date'),
    df_gdp.sort_values('trade_date'),
    on='trade_date',
    direction='backward'  # 低频数据→高频数据
)
```

### 4. 数据字典自动生成

#### 4.1 字段分类逻辑

```python
def classify_column(col_name):
    """
    根据前缀自动分类：
    - indiv_d_*    → 个股日线数据 → pro_bar接口
    - indiv_fin_*  → 个股财务数据 → fina_indicator接口
    - macro_cpi_*  → 宏观CPI数据 → cn_cpi接口
    """
    prefix_to_api = {
        'indiv_d_': 'pro_bar',
        'indiv_fin_': 'fina_indicator',
        'macro_cpi_': 'cn_cpi',
        # ... 更多映射
    }
    
    for prefix, api in prefix_to_api.items():
        if col_name.startswith(prefix):
            return prefix, api
    return None, None
```

#### 4.2 字段描述字典

```python
field_descriptions = {
    'close': {'cn': '收盘价', 'en': 'Close Price', 'unit': '元'},
    'pe_ttm': {'cn': '市盈率(TTM)', 'en': 'P/E Ratio (TTM)', 'unit': '倍'},
    'roe': {'cn': '净资产收益率', 'en': 'Return on Equity', 'unit': '%'},
    # 包含300+字段的完整定义
}
```

#### 4.3 生成流程

```mermaid
graph LR
    A[读取最终CSV列名] --> B[提取原始字段名]
    B --> C[查询字段描述字典]
    C --> D[确定API来源]
    D --> E[生成中英文说明]
    E --> F[输出column_description.csv]
```

---

## 快速开始

### 环境要求

```bash
Python 3.8+
pandas >= 1.3.0
numpy >= 1.20.0
tushare >= 1.2.89
```

### 安装步骤

```bash
# 1. 克隆或下载项目
cd "DSA5205  Data Science in Quantitative Finance/A2"

# 2. 安装依赖
pip install tushare pandas numpy

# 3. 注册Tushare账号获取token
# 访问 https://tushare.pro/register
```

### 配置Token

在Notebook第一个代码单元中修改：

```python
ts.set_token("YOUR_TOKEN_HERE")  # 替换为你的token
pro = ts.pro_api()
```

### 运行程序

1. **打开Jupyter Notebook**
   ```bash
   jupyter notebook dsa5205_p2_data_prep.ipynb
   ```

2. **按顺序执行所有单元格**
   - 单元1: 环境设置
   - 单元2-4: 交易日历和基准指数
   - 单元5-13: 个股数据抓取（7个维度）
   - 单元14-18: ETF数据抓取（5个维度）
   - 单元19-26: 市场和宏观数据抓取
   - 单元27-30: 数据整合
   - 单元31: 生成数据字典

3. **预计运行时间**
   - 数据抓取: 15-20分钟（受API限速影响）
   - 数据整合: 2-3分钟
   - 总计: ~25分钟

### 时间参数配置

```python
# 研究时间范围设置（位于第2个单元格）
end_date = "20251023"  # 结束日期（可修改为当前日期）
start_date = (datetime.datetime.strptime(end_date, "%Y%m%d") 
              - datetime.timedelta(days=760)).strftime("%Y%m%d")  # 向前约2年

# 财务数据额外缓冲
fin_start_date = (datetime.datetime.strptime(start_date, "%Y%m%d") 
                  - datetime.timedelta(days=180)).strftime("%Y%m%d")  # 再向前6个月
```

---

## 输出数据说明

### 1. 交易日历
**文件**: `data/trade_cal.csv`

| 字段 | 说明 |
|------|------|
| cal_date | 交易日期（索引） |
| is_open | 是否开市（全为1） |
| pretrade_date | 前一个交易日 |

### 2. 基准指数
**文件**: `data/benchmark/*.csv`

包含沪深300和中证2000指数的日线行情，用于收益率对比。

### 3. 个股完整数据
**文件**: `data/stock/full/{ts_code}_indiv_full_macro.csv`

每个文件包含一只股票的完整数据：
- **行数**: ~500条（对应交易日数）
- **列数**: ~330列（7个个股维度 + 市场数据 + 宏观数据）
- **索引**: trade_date（交易日期）
- **特点**: 所有数据已对齐，可直接用于建模

**示例列名**：
```
trade_date              # 交易日期（索引）
indiv_d_ts_code        # 股票代码
indiv_d_close          # 收盘价（后复权）
indiv_d_vol            # 成交量
indiv_din_pe_ttm       # 市盈率TTM
indiv_din_turnover_rate # 换手率
indiv_fin_roe          # 净资产收益率
indiv_fin_netprofit_margin # 销售净利率
indiv_chip_winner_rate # 筹码胜率
indiv_techin_macd_dif  # MACD-DIF
market_din_sh_pe       # 上证PE
macro_shibor_1m        # 1个月SHIBOR
macro_gdp_gdp_yoy      # GDP同比增速
macro_cpi_nt_yoy       # CPI同比
macro_im_spx_close     # 标普500收盘点位
```

### 4. ETF完整数据
**文件**: `data/etf/full/{ts_code}_indiv_full_macro.csv`

结构类似个股数据，列数约260列。

### 5. 数据字典
**文件**: `data/column_description.csv`

| 列名 | 说明 |
|------|------|
| 当前列名 | 数据文件中的完整列名 |
| 一级来源 | indiv/market/macro |
| 二级分类 | d/din/fin/chip等 |
| tushare接口链接 | 官方文档链接 |
| 字段名 | 去掉前缀的原始字段名 |
| 中文含义 | 字段的中文说明 |
| 英文含义 | 字段的英文说明 |
| 单位 | 元/点/%等 |

**示例行**：
```csv
当前列名,一级来源,二级分类,tushare接口链接,字段名,中文含义,英文含义,单位
indiv_d_close,indiv,d,https://tushare.pro/document/2?doc_id=109,close,收盘价,Close Price,元
indiv_fin_roe,indiv,fin,https://tushare.pro/document/2?doc_id=79,roe,净资产收益率,Return on Equity,%
macro_cpi_nt_yoy,macro,cpi,https://tushare.pro/document/2?doc_id=228,nt_yoy,全国居民消费价格同比,CPI (YoY),%
```

---

## 常见问题（FAQ）

### Q1: Tushare API调用失败怎么办？

**A**: 常见原因和解决方案：

1. **Token无效**
   ```python
   # 检查token是否正确设置
   import tushare as ts
   ts.set_token("YOUR_TOKEN_HERE")
   pro = ts.pro_api()
   
   # 测试连接
   df = pro.trade_cal(exchange='SSE', start_date='20240101', end_date='20240110')
   print(df)
   ```

2. **达到积分上限**
   - Tushare有每分钟调用次数限制
   - 在抓取代码中已加入`time.sleep(0.5)`延时
   - 如果仍报错，增加延时到`time.sleep(1.0)`

3. **网络问题**
   - 检查网络连接
   - 尝试使用代理

### Q2: 为什么有些列有很多NaN？

**A**: 这是正常现象，原因包括：

1. **数据频率不同**
   - 财务数据（季度）在非财报日为空
   - 解决：使用`fillna(method='ffill')`前向填充

2. **部分字段不适用**
   - 某些技术指标需要足够历史数据才能计算
   - 解决：删除前N行或使用插值

3. **数据源缺失**
   - Tushare部分字段本身就有缺失
   - 解决：使用替代字段或其他数据源

**示例处理**：
```python
# 前向填充财务数据
fin_cols = [col for col in df.columns if 'indiv_fin_' in col]
df[fin_cols] = df[fin_cols].fillna(method='ffill')

# 删除缺失率超过50%的列
threshold = 0.5
missing_ratio = df.isna().mean()
cols_to_keep = missing_ratio[missing_ratio < threshold].index
df_clean = df[cols_to_keep]
```

### Q3: 如何更新数据到最新日期？

**A**: 修改Notebook第2个单元格：

```python
# 方案1: 使用当前日期
end_date = datetime.datetime.today().strftime("%Y%m%d")

# 方案2: 指定日期
end_date = "20241231"

# 时间范围保持2年
start_date = (datetime.datetime.strptime(end_date, "%Y%m%d") 
              - datetime.timedelta(days=760)).strftime("%Y%m%d")
```

然后重新运行所有单元格。

### Q4: 如何添加更多股票？

**A**: 修改股票列表（单元格5-13中）：

```python
# 在第5个单元格修改
stocks = ['600036.SH', '002594.SZ', '600519.SH', '600000.SH']  # 添加浦发银行
stock_names = ['招商银行', '比亚迪', '贵州茅台', '浦发银行']
```

然后依次执行后续单元格即可。

### Q5: 宏观数据的频率如何处理？

**A**: 系统已自动处理不同频率：

- **日度数据**（SHIBOR、美债）: 直接对齐
- **月度数据**（CPI、PPI）: 转换为月末日期后向后填充
- **季度数据**（GDP）: 转换为季末日期后向后填充

使用时无需特殊处理，系统保证每个交易日都有对应的宏观数据值（前向填充）。

### Q6: column_description.csv 显示"待补充"怎么办？

**A**: 这表示某些字段尚未添加中英文说明。解决方案：

1. 访问对应的Tushare文档链接
2. 在`field_descriptions`字典中添加定义
3. 重新运行生成数据字典的单元格

**示例**：
```python
field_descriptions = {
    # 添加新字段定义
    'new_field': {'cn': '新字段中文名', 'en': 'New Field English Name', 'unit': '单位'},
    # ... 其他字段
}
```

### Q7: 如何处理复权数据？

**A**: 系统提供三种价格：

```python
# 后复权价格（推荐用于价格走势和收益率计算）
df['indiv_d_close']  # 已后复权

# 不复权价格（技术指标已包含）
df['indiv_techin_close_bfq']

# 前复权价格（如需要可从技术指标中获取）
df['indiv_techin_close_qfq']
```

**建议**：
- 价格走势分析：使用后复权（`indiv_d_close`）
- 技术指标：使用不复权（`*_bfq`系列）
- 交易模拟：使用当日实际价格（不复权）

### Q8: 内存不足怎么办？

**A**: 数据量较大时可能出现内存问题：

```python
# 方案1: 只加载需要的列
cols_needed = ['indiv_d_close', 'indiv_fin_roe', 'macro_gdp_gdp_yoy']
df = pd.read_csv('data/stock/full/600036.SH_indiv_full_macro.csv', 
                 usecols=cols_needed, index_col='trade_date')

# 方案2: 使用数据类型优化
df = pd.read_csv('data/stock/full/600036.SH_indiv_full_macro.csv',
                 dtype={'indiv_d_vol': 'int32'},  # 指定更小的数据类型
                 index_col='trade_date')

# 方案3: 只加载部分日期
df = pd.read_csv('data/stock/full/600036.SH_indiv_full_macro.csv', 
                 index_col='trade_date', parse_dates=True)
df = df.loc['2024-01-01':]  # 只保留2024年数据
```

---

## 数据质量检查

### 自动化检查清单

运行以下代码检查数据质量：

```python
def check_data_quality(file_path):
    """检查数据质量"""
    df = pd.read_csv(file_path, index_col='trade_date', parse_dates=True)
    
    print("=" * 80)
    print(f"数据文件: {file_path}")
    print("=" * 80)
    
    # 1. 基本信息
    print(f"\n1. 基本信息:")
    print(f"   时间范围: {df.index.min()} 至 {df.index.max()}")
    print(f"   交易日数: {len(df)}")
    print(f"   特征数量: {len(df.columns)}")
    
    # 2. 缺失值分析
    print(f"\n2. 缺失值分析:")
    missing = df.isna().mean() * 100
    print(f"   整体缺失率: {missing.mean():.2f}%")
    print(f"   缺失>50%列数: {(missing > 50).sum()}")
    print(f"   完全无缺失列数: {(missing == 0).sum()}")
    
    # 3. 关键字段检查
    print(f"\n3. 关键字段检查:")
    key_fields = ['indiv_d_close', 'indiv_din_pe_ttm', 'indiv_fin_roe']
    for field in key_fields:
        if field in df.columns:
            miss_rate = df[field].isna().mean() * 100
            print(f"   {field}: 缺失率 {miss_rate:.2f}%")
    
    # 4. 异常值检查
    print(f"\n4. 异常值检查:")
    if 'indiv_d_close' in df.columns:
        price = df['indiv_d_close']
        print(f"   价格范围: {price.min():.2f} - {price.max():.2f}")
        print(f"   价格异常(0或负): {(price <= 0).sum()}")
    
    # 5. 时间序列连续性
    print(f"\n5. 时间序列检查:")
    date_diff = df.index.to_series().diff().dt.days
    print(f"   最大间隔天数: {date_diff.max()}")
    print(f"   >10天间隔次数: {(date_diff > 10).sum()}")
    
    return df

# 检查所有股票数据
for stock in ['600036.SH', '002594.SZ', '600519.SH']:
    check_data_quality(f'data/stock/full/{stock}_indiv_full_macro.csv')
```

### 预期结果

**优质数据标准**：
- ✅ 整体缺失率 < 30%
- ✅ 核心字段（价格、PE、ROE）缺失率 < 10%
- ✅ 无价格异常（0或负值）
- ✅ 时间序列连续（最大间隔<10天为节假日）

---

## 性能优化建议

### 1. 数据抓取优化

```python
# 批量抓取减少API调用次数
stocks_batch = ['600036.SH', '002594.SZ', '600519.SH']
df_list = []
for stock in stocks_batch:
    df = pro.daily_basic(ts_code=stock, start_date=start_date, end_date=end_date)
    df_list.append(df)
    time.sleep(0.6)  # 稍微加长延时避免触发限制
df_all = pd.concat(df_list)
```

### 2. 内存优化

```python
# 使用更小的数据类型
dtype_dict = {
    'indiv_d_vol': 'int32',
    'indiv_d_amount': 'float32',
    'indiv_din_turnover_rate': 'float16',
}
df = pd.read_csv(file_path, dtype=dtype_dict)
```

### 3. 并行处理

```python
from multiprocessing import Pool

def process_stock(stock_info):
    stock, name = stock_info
    # ... 处理单只股票
    return result

# 并行处理多只股票
with Pool(processes=3) as pool:
    results = pool.map(process_stock, zip(stocks, stock_names))
```

---

## 扩展功能建议

### 1. 增加更多数据源

```python
# 北向资金持股
df_hold = pro.hk_hold(ts_code='600036.SH', start_date=start_date, end_date=end_date)

# 研报数据
df_report = pro.report_rc(ts_code='600036.SH', start_date=start_date, end_date=end_date)

# 行业数据
df_industry = pro.index_classify(level='L1', src='SW2021')
```

### 2. 自动化数据更新

```python
import schedule

def daily_update():
    """每日自动更新数据"""
    today = datetime.datetime.today().strftime("%Y%m%d")
    # 只更新最近5个交易日的数据
    start = (datetime.datetime.today() - datetime.timedelta(days=10)).strftime("%Y%m%d")
    
    for stock in stocks:
        df_new = ts.pro_bar(ts_code=stock, adj='hfq', start_date=start, end_date=today)
        # 追加到现有数据
        df_old = pd.read_csv(f'data/stock/raw/{stock}_indiv_d.csv')
        df_combined = pd.concat([df_old, df_new]).drop_duplicates(subset=['trade_date'])
        df_combined.to_csv(f'data/stock/raw/{stock}_indiv_d.csv', index=False)

# 每天17:00执行
schedule.every().day.at("17:00").do(daily_update)
```

### 3. 数据可视化仪表盘

```python
import plotly.graph_objects as go
from plotly.subplots import make_subplots

def create_dashboard(stock_code):
    df = pd.read_csv(f'data/stock/full/{stock_code}_indiv_full_macro.csv',
                     index_col='trade_date', parse_dates=True)
    
    # 创建多子图仪表盘
    fig = make_subplots(
        rows=3, cols=2,
        subplot_titles=('股价走势', '市盈率', '净资产收益率', '换手率', 'MACD', 'GDP增速')
    )
    
    # ... 添加各个图表
    
    fig.update_layout(height=1000, title_text=f"{stock_code} 数据仪表盘")
    fig.show()
```

---

## 参考

### 数据来源
- [Tushare Pro](https://tushare.pro/)

### 参考资料
- Tushare Pro官方文档: https://tushare.pro/document/1

### 版本历史
- v1.0 (2024-10-24): 初始版本，包含核心功能
  - 3只个股 + 2只ETF数据
  - 7个个股维度 + 5个ETF维度
  - 市场和宏观数据整合
  - 自动生成数据字典

---

## 许可证

本项目仅供学术研究使用，请遵守Tushare Pro的使用条款。

---

---

**最后更新**: 2025年10月24日
