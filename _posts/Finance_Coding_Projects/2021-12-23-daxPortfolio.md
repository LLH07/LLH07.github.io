---
layout: "post"
title: 利用因子分析建構投資組合 – 以德國 DAX 指數為例
date: 2021-12-23 05:13:00 +0800
categories: [Coding, Finance project]
tags: [Factor Investing, Backtest]
author:
  name: Lung Hung Lin
  link: https://www.linkedin.com/in/lung-hung-blair-lin-645a85194/ 
---
- [相關連結](#相關連結)
- [結果](#結果)
- [程式碼](#程式碼)
  - [Get data from yahoo finance](#get-data-from-yahoo-finance)
  - [Slice price and market equity data](#slice-price-and-market-equity-data)
  - [Get momentum according to formula](#get-momentum-according-to-formula)
  - [Set rebalance rule and combine two cut-off portfolio](#set-rebalance-rule-and-combine-two-cut-off-portfolio)
  - [Stimulate real world trading and the visualization of our strategy (backtesting)](#stimulate-real-world-trading-and-the-visualization-of-our-strategy-backtesting)
  - [Identify the weight of each stock on 2020-12-30 and identify the **good** stock of each stock on 2020-12-30](#identify-the-weight-of-each-stock-on-2020-12-30-and-identify-the-good-stock-of-each-stock-on-2020-12-30)
  - [Identify the bad stock of each stock on 2020-12-30](#identify-the-bad-stock-of-each-stock-on-2020-12-30)
  - [Plot the out-of-sample performance](#plot-the-out-of-sample-performance)
- [回饋與報告影片連結](#回饋與報告影片連結)
  
## 相關連結
- [專案資料夾](https://drive.google.com/drive/folders/1nkuYKCLmsEUGmpHFYRLXhx9-43kgDoLM?usp=sharing){:target="_blank"}
- [Github Repo](https://github.com/LLH07/InvestnPortfolio/tree/main/project){:target="_blank"}
  
## 結果
優化傳統 CAPM 模型，找出可能帶來超額報酬的因子，投資組合在 2021 年超越 DAX 大盤指數 7%
## 程式碼
### Get data from yahoo finance
```python
import yfinance as yf
import numpy as np
import pandas as pd

start_date = '2020-01-01'
end_date = '2020-12-31'

Constituent = list(pd.read_html('https://en.wikipedia.org/wiki/DAX')[3]['Ticker symbol'])
shares = {}
df = pd.DataFrame()

for ticker in Constituent:
    try:
        df[ticker] = yf.download(ticker, start = start_date, end = end_date)['Adj Close']
    except Exception as e:
        print('Failed to Download or merge : '+ticker)

    try:
        shares[ticker] = yf.Ticker(ticker).info['floatShares']
    except:
        print(ticker+' shares unfound')
```

### Slice price and market equity data

```python
# get price
PRC = df.copy()
PRC.drop(['ENR.DE'], axis=1, inplace=True) # ENR.DE is not listed in DAX30 until 2020-12, so drop it 

# get market equity
for ticker in df.columns:
    df[ticker] *= shares[ticker]

# market equity cutoff
cutoff = df.quantile(0.5, axis=1)
port_df = pd.DataFrame(columns=df.columns,index=df.index)
port_df[df.gt(cutoff,axis=0)] = 'Big'
port_df[df.le(cutoff,axis=0)] = 'Small'


ME_port = port_df.copy()
ME_port.drop(columns=['ENR.DE'],inplace=True)
ME_port = ME_port.iloc[13:,:]
```

### Get momentum according to formula
```python
RET = PRC.pct_change()
tmp_RET = (RET + 1)
tmp_RET = tmp_RET.iloc[1:, :] # delete the first one since it has no return
MOM = tmp_RET.rolling(11).apply(np.prod).shift(2)
MOM = MOM.apply(pd.to_numeric)
L_cutoff = pd.to_numeric(MOM.quantile(.3,axis=1,numeric_only=False))
H_cutoff = pd.to_numeric(MOM.quantile(.7,axis=1,numeric_only=False))
WL_port = pd.DataFrame(index=RET.index, columns=RET.columns)
WL_port[MOM.gt(H_cutoff, axis=0)] = 'Winner'
WL_port[(MOM.le(H_cutoff, axis=0)) & (MOM.ge(L_cutoff, axis=0))] = 'Neutral'
WL_port[MOM.lt(L_cutoff, axis=0)] = 'Loser'
WL_port.tail()
```

### Set rebalance rule and combine two cut-off portfolio
Since the momentum will be updated everyday, and market equity will not change significantly,
we decide to only get momentum data on the last day of every month.

```python
WL_port.index = pd.to_datetime(WL_port.index, format='%Y%m%d', errors='ignore')+ pd.offsets.MonthEnd(0)

# store 2020 Feb and months other than Feb's date
other_list = []
feb_list = []

feb = pd.DataFrame()
other = pd.DataFrame()

for i in PRC.index:
    if i.month == 2:
        feb_list.append(i)
    else:
        other_list.append(i)

feb = WL_port[WL_port.index.month == 2]
other = WL_port[WL_port.index.month != 2]

feb.index = feb_list
other.index = other_list

# merge the two dataframes (Feb and Non-Feb)
WL_port = other.merge(feb, how='outer')
# then revise the index to the correct order (Jan, Feb, Mar,....)
all_date = feb_list + other_list
all_date.sort()
WL_port.index = all_date
WL_port.dropna(inplace=True)

ME_MOM_port = ME_port + WL_port
ME_MOM_port.head(5)
```
### Stimulate real world trading and the visualization of our strategy (backtesting)
```python
ME_lag = df.shift(1)
unique_port = ['SmallLoser', 'SmallNeutral', 'SmallWinner', 'BigLoser','BigNeutral', 'BigWinner']

RET_port = pd.DataFrame(index=RET.index, columns=unique_port)
N_firm = pd.DataFrame(index=RET.index, columns=unique_port)

for p in unique_port:
  TMP_RET = RET[ME_MOM_port==p].apply(pd.to_numeric)
  TMP_ME = ME_lag[ME_MOM_port==p].apply(pd.to_numeric)
  TMP_PROD = TMP_RET*TMP_ME
  RET_port[p] = TMP_PROD.sum(axis=1)/TMP_ME.sum(axis=1)
  N_firm[p] = TMP_RET.count(axis=1)
RET_port = RET_port.dropna()

import matplotlib.pyplot as plt
fig = plt.figure(figsize=(15, 10))
ax = fig.add_subplot(111)

tmp = (RET_port.dropna()+1).cumprod()

ax.plot(tmp, label=unique_port)
ax.legend(loc='best')
ax.set_xlabel('Time')
ax.set_ylabel('Return')
ax.set_title('Backtest: Momentum-Market Equity Strategy on DAX30')
```
![backtest result](https://lh3.googleusercontent.com/pw/AM-JKLUkj8fGDly1HKqUvDaTGSZ9qWtSXWOuXMUXNhYYy8GYqKqOFQevbTYa4zjLPsHu2Eqeu-MUJCewhzVooB4X2O5ih_R2uIQGOulobjPTDVbnVrz6zJdRSp8XYiesZXSZknRJnSpYMIAat3HNCQmcWzNm=w888-h604-no?authuser=0)

[Explanation]
After backtesting the data for one year, we define our strategy as:  
- **Long ```BigLoser```, ```SmallNeutral```, ```BigNeutral```** (top 3 best performers on 2020), **Short ```SamllLoser```, ```SmallWinner```, ```BigWinner```** (worst 3 performers on 2020)
- How to determine weight: the weight on 2020-12-30   
- How to rank the stocks: the ranking on 2020-12-30

### Identify the weight of each stock on 2020-12-30 and identify the **good** stock of each stock on 2020-12-30
```python
# identify stocks that we are going to long
filter_ = (last_rank.T['2020-12-30'] == 'BigLoser') | (last_rank.T['2020-12-30'] == 'SmallNeutral') | (last_rank.T['2020-12-30'] == 'SmallLoser') 
good_rank = list(last_rank.T[filter_].index)
last_date = df.iloc[-1:]

good_weights = {} # store the weight for each stock
sum_me = 0
last_date = df.iloc[-1:]

for c in df.columns:
  if c in good_rank:
    me = last_date[c]
    sum_me += me # calculate total market equity

for c in df.columns:
  if c in good_rank:
    good_weights[c] = last_date[c] / sum_me # calculate the value-weighted weight
```

###  Identify the bad stock of each stock on 2020-12-30
```python
filter_ = (last_rank.T['2020-12-30'] == 'BigNeutral') | (last_rank.T['2020-12-30'] == 'BigWinner') | (last_rank.T['2020-12-30'] == 'SmallWinner') 
bad_rank = list(last_rank.T[filter_].index)
bad_weights = {}
sum_me = 0
last_date = df.iloc[-1:]

for c in df.columns:
  if c in bad_rank:
    me = last_date[c]
    sum_me += me 

for c in df.columns:
  if c in bad_rank:
    bad_weights[c] = last_date[c] / sum_me
```

### Plot the out-of-sample performance
```python
# merge the two dicts
for ticker, value in (good_weights.items()):
    bad_weights[ticker] = value
weights = bad_weights

start_date = '2021-01-01'
end_date = '2021-12-22'
#data = yf.download('DAX',start='2017-01-01',end='2021-01-01')

Constituent = list(pd.read_html('https://en.wikipedia.org/wiki/DAX')[3]['Ticker symbol'])
shares = {}
df_2021 = pd.DataFrame()
for ticker in Constituent:
    try:
        df_2021[ticker] = yf.download(ticker, start = start_date, end = end_date)['Close']
    except Exception as e:
        print('Failed to Download or merge : '+ticker)

# get the daily return 
ret_2021 = df_2021.pct_change()
ret_2021 = ret_2021.iloc[1:,]
ret_2021.drop(columns=['ENR.DE'], inplace=True)
weight_df = pd.DataFrame(np.nan, index=ret_2021.index, columns=ret_2021.columns)
# weight_df will store the weight of the stocks 
for c in weight_df:
  for k, v in weights.items():
    if c == k: # e.g. if 
      weight_df[c] = v[0] 

# calculate return
res_df = weight_df * ret_2021
res_df.sum(axis=1)
plt.style.use('seaborn')
fig = plt.figure(figsize=(15, 10))
ax = fig.add_subplot(111)

tmp = 1 * ((res_df).sum(axis=1) + 1).cumprod()

ax.plot(tmp)
ax.set_xlabel('Time')
ax.set_ylabel('Return')
ax.set_title('Out of Sample: LONG top three SHORT last three strategy')
```
![out of sample result](https://lh3.googleusercontent.com/pw/AM-JKLUKiGKVCSdqcwv9K7rSaR1_7mr0i9fKw5rBRPC-RuAJGJGZ9aMXxNqo1dCzXSPJ9DWkVtI_C-ab8_sPVpXqfRoFxc7vYEb6p5lVMgvC1ptdwn7SCyJVNvMGk_eZKo6V5y3EAK1bV0NxeVu4NugjeV97=w896-h602-no?authuser=0)

```python
final_res = 1 * ((res_df).sum(axis=1) + 1).cumprod()
VAR = (res_df.sum(axis=1) * 100 ).var()
MEAN = (res_df.sum(axis=1) * 100 ).mean()

# Sharpe Raito
# risk-free: 0%
SR = (MEAN-0) / (VAR)**0.5
SR
```

## 回饋與報告影片連結
- 指導老師評語:
  ![指導老師評語](https://lh3.googleusercontent.com/pw/AM-JKLXx-Zmv1H1hqmH44gT-_YtPD3DJEhEfTd3b-KrMkWl8DYPR5w64Jx1wdPpT3DhvQ3PaH5cuGpdarXbHf-tOk2Mv8tr0zgsgvryhdLCOJJtJcFrKGa0oStZcOaKug5msFxulZfsKlJ9DQKRnEbguYk1M=w760-h341-no?authuser=0)

- [2022.03.18 [110-2 投資學助教課] Yahoo Finance and JupyterLab Introduction](https://www.youtube.com/watch?v=axOSG4ZaT8E){:target="_blank"}
