---
layout: "post"
title: 日幣的三月季節性 (以 EURJPY 回測)
date: 2022-03-13 23:00:00 +0800
categories: [Coding, Finance project]
tags: [Backtest]
author:
  name: Lung Hung Lin
  link: https://www.linkedin.com/in/lung-hung-blair-lin-645a85194/ 
---
- [三月的日幣季節性](#三月的日幣季節性)
- [實測結論](#實測結論)
- [EURJPY 回測 (1999-2021)](#eurjpy-回測-1999-2021)
  - [引入相關套件 (以 ```dbnomics``` 套件取得 EURJPY 資料)](#引入相關套件-以-dbnomics-套件取得-eurjpy-資料)
  - [自套件取得資料並計算報酬](#自套件取得資料並計算報酬)
  - [畫出持有三個交易日的報酬情況](#畫出持有三個交易日的報酬情況)
- [USDJPY 回測 (2017-2021)](#usdjpy-回測-2017-2021)
- [附錄](#附錄)
  
## 三月的日幣季節性
近期在前大型銀行交易員 Brent Donnelly 的文章讀到日幣在三月有季節性的跌幅。 一些網路文章亦提到各種貨幣對的季節性，例如 [這篇文章](https://www.investopedia.com/articles/forex/07/forex_seasonality.asp){:target="_blank"} 與 [這篇文章](https://vantagepointtrading.com/forex-seasonality-usdjpy-seasonal-patterns/){:target="_blank"}。

Brent 的論點如下:

每年三月中旬時 USDJPY 會先下跌，之後上漲至 4 月初，接著又下跌至四月中。作者並且回測這個理論 (從 2010 至 2021)，發現的確有這個現象。

其給出的理由是: 日本的會計年度在 03/31 結束，因此 3 月中開始，大量在外國設址的日本企業會將外幣轉為日幣 (以讓財務表現更好看)，推升日幣買壓，使 USDJPY 下跌，Brent 大大用的詞是 “repatriation”，中文是遣返，也有 “將外幣兌換成本國貨幣”的意思。 4 月一到來 (新的會計年度開始)，這些企業又將日幣轉為外幣以方便做生意，使得 USDJPY 上漲，至於 4 月初到 4 月中的持續上漲，Brent 表示這是企業發給員工的獎金，至於這個現象是否可靠? 作者 Brent 有在日本野村銀行工作過，他說保證有這樣的現象。

## 實測結論
我實際用 EURJPY 與 USDJPY 測，發現使用這個策略，也就是在 4 月初做多 EURJPY (持有期間三個交易日)，從 1999 年到 2021 年，這策略的勝率約 52% (樣本數為 46)。若改用 USDJPY，則從 2017 年至 2021 年，勝率約 55% (樣本數為 11)。

至於為何 EURJPY 我用 22 年的數據，USDJPY 卻只用近 6 年? 因為 Fred 提供的 USDJPY 資料從 2017 年開始，Dbnomics 又沒有提供 USDJPY 的資料，所以只能將就點囉。

## EURJPY 回測 (1999-2021)
### 引入相關套件 (以 ```dbnomics``` 套件取得 EURJPY 資料)
```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

from dbnomics import fetch_series, fetch_series_by_api_link
```

### 自套件取得資料並計算報酬
```python
eurjpy = fetch_series('BDF', 'EXR', 'EXR.D.JPY.EUR.SP00.A')
df = pd.DataFrame()
cols = ['period', 'value']

for i in cols:
    df[i] = eurjpy[i]
    
df.columns = ['Time', 'EURJPY']
df.dropna(inplace=True)
df.index = range(0, df.shape[0])
```

```python
period_ret_dict = dict()
for i in range(0, df.shape[0], 3):
    period_start = (df.loc[i, 'Time']).date()
    end_pos = i + 3
    tmp_df = pd.DataFrame(df.loc[i:end_pos])
    tmp_df['Ret'] = tmp_df['EURJPY'].pct_change()
    tmp_df.dropna(inplace=True)
    cum_ret_periods = (tmp_df['Ret']+1).cumprod() - 1 
    # this is a pandas Series
    period_ret_dict[period_start] = cum_ret_periods.tail(1)
```


### 畫出持有三個交易日的報酬情況
```python
tmp_l = list()
for i in period_ret_dict.values():
    tmp_l.append(float(i.values))
s = pd.Series(tmp_l, index=period_ret_dict.keys())

count = 0
pos_ret = 0
count_s = pd.Series()

for i in range(len(s)):
    if s.index[i].month == 4 and s.index[i].day < 10:
        count += 1
        if s[i] > 0:
            pos_ret += 1
        count_s[s.index[i]] = s[i]
count = 0
pos_ret = 0
count_s = pd.Series()

for i in range(len(s)):
    if s.index[i].month == 4 and s.index[i].day < 10:
        count += 1
        if s[i] > 0:
            pos_ret += 1
            
        count_s[s.index[i]] = s[i]
        #print(s.index[i], s[i])
win_rate = float("{:.3f}".format(pos_ret/count))

print(f'報酬 > 0 % 次數: {pos_ret} out of {count} 次')
print(f'四月初多 EURJPY 的勝率為: {win_rate*100} %')
```

![pic1](https://lh3.googleusercontent.com/pw/AM-JKLWoTKKTGzmdIrgOZoJp5OwMsqcUNEzqnlP3B8nphAMJ6UOM56_zXx3fl-CHp41lkQjXWW00RGmDkDqCC337SkzFFZSUSpsgDQBnXl2tKe46rdAhp7J5sqNLw3arIfzaVmdVc1uCvrOOiHp43X_apOAT=w327-h395-no?authuser=0)

![pic2](https://lh3.googleusercontent.com/pw/AM-JKLX3020EsJ587Awv-XKE5fmQ12yYhOaft1oIPGvV5EA-KMifibT8SchH8uPnMv9hrRlXX7MuOCE4w6Q_0tTQZkIMOW2zAl7nsi5HMxsV7U6XWs4Ch6s_W3nx0fLVjR477I4lhn14rBYfoYG9zKQeMWUx=w314-h58-no?authuser=0)


```python
import matplotlib.dates as mdates
plt.style.use('ggplot')
plt.xlabel('Time')
plt.ylabel('Return')
plt.title('EURJPY 3 trading day return (Early April)')
count_s.plot(kind='bar')
plt.gca().xaxis.set_major_locator(mdates.DayLocator(interval=5))
plt.gcf().autofmt_xdate()
```

![pic2](https://lh3.googleusercontent.com/pw/AM-JKLWNJQn6rudwoToYvc42BPGUn33SVbtdlC9sIfLkBK7b6Cnga35lAX6rQns0Zmf3QFvFabPjeMVFBi8yiI2dBEybzueCoIVPqQdJhcmaRTtutq4wOsX-xIswh8OoJFDP1vrKuG62-DRal9lLvfQ72Ni0=w403-h289-no?authuser=0)

在 4 月初做多 **EURJPY** 的報酬 (1999-2021)

## USDJPY 回測 (2017-2021)
```python
from pandas_datareader import data
df = data.DataReader('DEXJPUS', 'fred')
df['Time'] = df.index
df.index = range(0, df.shape[0])
df.rename(columns={'DEXJPUS':'USDJPY'}, inplace=True)

period_ret_dict = dict()

for i in range(0, df.shape[0], 3):
    period_start = (df.loc[i, 'Time']).date()
    end_pos = i + 3
    tmp_df = pd.DataFrame(df.loc[i:end_pos])
    tmp_df['Ret'] = tmp_df['USDJPY'].pct_change()
    tmp_df.dropna(inplace=True)
    cum_ret_periods = (tmp_df['Ret']+1).cumprod() - 1
    period_ret_dict[period_start] = cum_ret_periods.tail(1)

tmp_l = list()
for i in period_ret_dict.values():
    tmp_l.append(float(i.values))
s = pd.Series(tmp_l, index=period_ret_dict.keys())

count = 0
pos_ret = 0
count_s = pd.Series()

for i in range(len(s)):
    if s.index[i].month == 4 and s.index[i].day < 10:
        count += 1
        if s[i] > 0:
            pos_ret += 1
        if s[i] < 0:
            print(s.index[i])
            print(s[i])
            
        count_s[s.index[i]] = s[i]

win_rate = float("{:.3f}".format(pos_ret/count))

print(f'報酬 > 0 % 次數: {pos_ret} out of {count} 次')
print(f'四月初多 USDJPY 的勝率為: {win_rate*100} %')

import matplotlib.dates as mdates
count_s.plot(kind='bar')
plt.xlabel('Time')
plt.ylabel('Return')
plt.title('USDJPY 3 trading day return (Early April)')
plt.gcf().autofmt_xdate()
```

---
## 附錄
- [GritFX VOL 25](https://financeprotein.com/macroeconomics/fx/GritFX-VOL25/){:target="_blank"}
- [程式碼檔案](){:target="_blank"}