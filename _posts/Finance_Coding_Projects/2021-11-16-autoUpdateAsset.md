---
layout: "post"
title: 證券每日報酬與大盤走勢視覺化
date: 2021-11-16 11:08:00 +0800
categories: [Coding, Finance project]
tags: [WebScrape, Visualization]
author:
  name: Lung Hung Lin
  link: https://www.linkedin.com/in/lung-hung-blair-lin-645a85194/ 
---
- [相關連結](#相關連結)
- [結果](#結果)
- [程式碼](#程式碼)
  
## 相關連結
[專案資料夾](https://drive.google.com/drive/folders/1_p7Y8dRdIvdE7ZAmwwaiFDr0PyrNyq7u?usp=sharing){:target="_blank"}

## 結果
![result](https://lh3.googleusercontent.com/pw/AM-JKLXqLYP9WXn1EIwZbMrCEuV2ewR6hVbIYXCnxa_ZfzWERM2D9zLet84j8jfVxaSsoO4FtuOpSPURD31yBnTneRBKEllTZ1EN1dk3dWzvTuuUtQ8BTqkBHFxy3q8OEnc9vyQinzPpNNkJRTdkzuKC5uTl=s864-no?authuser=0)

## 程式碼
```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import datetime as dt
from datetime import date
import time
import re
hold_df = pd.read_excel(r"C:\Users\User\理財\資產持有明細.xlsx")
stat = pd.read_excel(r"C:\Users\User\理財\林氏家族基金.xlsx")
df = stat.copy()
df.drop('Unnamed: 0', axis=1, inplace=True)
df.drop('index_pctchange', axis=1, inplace=True)
df.drop('asset_pctchange', axis=1, inplace=True)
new_dict = {
 "日期":np.nan,
 "股票成本":247379,
 "股票市值":0,
 "現金":df.loc[len(df)-1, '現金'], 
 "總資產":0,
 "台灣加權指數":np.nan, 
 "備註":""
}
costDaily = new_dict['股票成本'] - df.loc[len(df)-2, '股票成本']

# 更新當日時間
datetime_dt = dt.datetime.today()
dt_fmt = datetime_dt.strftime("%Y/%m/%d")
now_format = dt_fmt.replace('/', '-')
new_dict["日期"] = now_format
txt = new_dict['備註']
allTrade = txt.split(', ')
stock_b = []
amount_b = []
unit_b = []
chineseName_b = []
cost_b = []
stock_s = []
amount_s = []
unit_s = []
chineseName_s = []
gain_s = []
```
```python
for i in allTrade:
    keys = ['type', 'ticker', 'ChineseName', 'price', '_', 'amt', 'unit']
    vals = i.split(' ')
    global _
    _ = dict(zip(keys, vals))
    if _['type'] == "買入":
        cost_b.append(float(_['price']))
        stock_b.append(_['ticker'])
        amount_b.append(float(_['amt']))
        unit_b.append(_['unit'])
        chineseName_b.append(_['ChineseName'])
    elif _['type'] == "賣出":
        gain_s.append(float(_['price']))
        stock_s.append(_['ticker'])
        amount_s.append(float(_['amt']))
        unit_s.append(_['unit']) 
        chineseName_s.append(_['ChineseName'])
buydict = {'stocks':stock_b, 'chineseName':chineseName_b, 'amount':amount_b, 'unit_s':unit_b}
selldict = {'stocks':stock_s, 'chineseName':chineseName_s, 'amount':amount_s, 'unit_s':unit_s}
```

 ```python 
for i in range(len(stock_b)):
    if unit_b[i] == '張': # 買入整股
        new_dict['現金'] -= 1.001425 * amount_b[i] * cost_b[i] * 1000
    else: # 買入零股
        new_dict['現金'] -= 1.001425 * amount_b[i] * cost_b[i]
 
for i in range(len(stock_s)):
    if unit_s[i] == '張': # 賣出整股
        new_dict['現金'] += 0.995575 * amount_s[i] * gain_s[i] * 1000
    else: # 賣出零股
        new_dict['現金'] += 0.995575 * amount_s[i] * gain_s[i]

j = 0
for i in stock_b:
    i = int(i)
    s = hold_df['Stocks'] 
    if i not in s.unique(): # 資產中沒有此檔股票
        if unit_b[j] == '張':
            hold_df.loc[len(hold_df)] = [i, amount_b[j] * 1000]   
        elif unit_b[j] == '股':
            hold_df.loc[len(hold_df)] = [i, amount_b[j]]
        with open("C:\\Users\\User\\理財\\證券買賣紀錄\\"+ str(i) + " " + buydict['chineseName'][j] + ".txt",
                  'w', encoding='utf-8') as f:
            f.write(f'{now_format} 買入 {cost_b[j]} 元 {amount_b[j]} {unit_b[j]}\n')
            
    elif i in s.unique(): # 資產中已有此檔股票
        egg = (hold_df['Stocks'] == i)
        if unit_b[j] == '張':
            hold_df.loc[egg, 'amount'] =  hold_df.loc[egg, 'amount'] + amount_b[j]*1000     
        elif unit_b[j] == '股':
            hold_df.loc[egg, 'amount'] = hold_df.loc[egg, 'amount'] + amount_b[j] 
        with open("C:\\Users\\User\\理財\\證券買賣紀錄\\"+ str(i) + " " + buydict['chineseName'][j] + ".txt", 
                  'a+', encoding='utf-8') as f:
            f.write(f'{now_format} 買入 {cost_b[j]} 元 {amount_b[j]} {unit_b[j]}\n')
    else:
        print("Something Wrong! Check your account!")
    j += 1

j = 0
for i in stock_s:
    i = int(i)
    s = hold_df['Stocks']
    try:
        if i in s.unique():
            pos = (hold_df['Stocks']==i)
            if unit_s[j] == '張':
                hold_df.loc[pos] = [i, hold_df.loc[pos, 'amount'] - amount_s[j]*1000]
            elif unit_s[j] == '股':
                hold_df.loc[pos] = [i, hold_df.loc[pos, 'amount'] - amount_s[j] ]
            with open("C:\\Users\\User\\理財\\證券買賣紀錄\\"+ str(i) + " " + selldict['chineseName'][j] + ".txt", 
                      'a+', encoding='utf-8') as f:                   
                f.write(f'{now_format} 賣出 {gain_s[j]} 元 {amount_s[j]} {unit_s[j]}\n')
        else:
            print(f"Something Wrong! Maybe you don't have {i} in your account.")
        j += 1
    except:
        print(f"Other issues occur.")
        pass
 ```

 ```python
 stock_url_上櫃 = "https://invest.cnyes.com/twstock/tws/"
stock_url_上市 = "https://invest.cnyes.com/twstock/TWS/"
assetPrice_daily = {}
asset_daily = 0

for i in hold_df['Stocks'].astype(int):
    stock_url = stock_url_上櫃 + str(i)
    page = requests.get(stock_url)
    content = page.content
    soup = BeautifulSoup(content)
    egg = soup.find("div", {"class":"jsx-2941083017 info-price"})
    if egg == None: # 若股股票不是上市公司，則改用上櫃公司的網址爬蟲
        print('here')
        stock_url = stock_url_上市 + str(i)
        page = requests.get(stock_url)
        content = page.content
        egg = soup.find("div", {"class":"jsx-2941083017 info-price"})
        
    price = egg.find("div", {"class":"jsx-2941083017 info-lp"}).get_text()
    l = str(price)
    l = l.replace(',', '')
    price = float(l)
    print(f'{now_format} Price for {i} is {price}')
    egg = (hold_df['Stocks'] == i)
    asset_daily += price*int(hold_df.loc[egg, 'amount'])   
    assetPrice_daily[i] = price
```

```python 
stock_url_上櫃 = "https://invest.cnyes.com/twstock/tws/"
stock_url_上市 = "https://invest.cnyes.com/twstock/TWS/"
assetPrice_daily = {}
asset_daily = 0

for i in hold_df['Stocks'].astype(int):
    stock_url = stock_url_上櫃 + str(i)
    page = requests.get(stock_url)
    content = page.content
    soup = BeautifulSoup(content)
    egg = soup.find("div", {"class":"jsx-2941083017 info-price"})
    if egg == None: # 若股股票不是上市公司，則改用上櫃公司的網址爬蟲
        print('here')
        stock_url = stock_url_上市 + str(i)
        page = requests.get(stock_url)
        content = page.content
        egg = soup.find("div", {"class":"jsx-2941083017 info-price"})
        
    price = egg.find("div", {"class":"jsx-2941083017 info-lp"}).get_text()
    l = str(price)
    l = l.replace(',', '')
    price = float(l)
    print(f'{now_format} Price for {i} is {price}')
    egg = (hold_df['Stocks'] == i)
    asset_daily += price*int(hold_df.loc[egg, 'amount'])   
    assetPrice_daily[i] = price

url = 'https://invest.cnyes.com/index/TWS/TSE01'
page = requests.get(url)
content = page.content
soup = BeautifulSoup(content)
# 取得每日台灣加權指數
egg = soup.find("div", {"class":"jsx-2941083017 info-price"})
index = egg.find("div", {"class":"jsx-2941083017 info-lp"}).get_text()
l = str(index)
l = l.replace(',', '')

index = float(l)
print(f"{now_format} Taiex: {index}")
```

```python
new_dict['股票市值'] = asset_daily
new_dict['台灣加權指數'] = index
df = df.append(new_dict, ignore_index=True)

df.index = range(df.shape[0])
df['index_pctchange'] = df['台灣加權指數'].pct_change()
df['asset_pctchange'] = df['總資產'].pct_change()

from pandas.plotting import register_matplotlib_converters
register_matplotlib_converters()
fig = plt.figure(figsize=(12, 12))

ax0 = fig.add_subplot(211)
ax0.plot(df.iloc[20:, 0], df.loc[20:, "總資產"], color='green', marker='o')
ax0.set_title("Total Asset")
# 自動更新資產總報酬率
ax0.annotate(f"Total Asset return: {'{:,.2%}'.format(ret)}", 
            xy=(mdates.date2num(timeForPlot), df['總資產'][len(df)-1]),
            xytext=(mdates.date2num(timeForPlot)-3, df['總資產'][len(df)-1]+1000),
            arrowprops=dict(facecolor='green', shrink=0.05))
ax0.grid(color='black', linestyle='dotted', linewidth=1)

ax1 = fig.add_subplot(212)
ax1.plot(df.iloc[20:, 0], df.loc[20:, 'asset_pctchange'], label='Total Asset', color='red', marker='o')
ax1.plot(df.iloc[20:, 0], df.loc[20:, 'index_pctchange'], label='TAIEX', color='blue', marker='o')

# 自動更新當日大盤與資產的波動
ax1.annotate(f"TAIEX %: {'{:,.2%}'.format(df['index_pctchange'][len(df)-1])}", 
             xy=(mdates.date2num(timeForPlot), df['index_pctchange'][len(df)-1]), 
             xytext=(mdates.date2num(timeForPlot)-2, df['index_pctchange'][len(df)-1]+0.007), 
             arrowprops=dict(facecolor='blue', shrink=0.05))
ax1.annotate(f"Asset %: {'{:,.2%}'.format(df['asset_pctchange'][len(df)-1])}", 
             xy=(mdates.date2num(timeForPlot), df['asset_pctchange'][len(df)-1]), 
             xytext=(mdates.date2num(timeForPlot)-2, df['asset_pctchange'][len(df)-1]-0.007), 
             arrowprops=dict(facecolor='red', shrink=0.05))
ax1.grid(color='black', linestyle='dotted', linewidth=1)

ax1.legend(loc='upper center')
ax1.set_title("The daily percentage change of Total Asset and TAIEX")


vals = ax1.get_yticks()
ax1.set_yticklabels(['{:,.2%}'.format(x) for x in vals])

fig.savefig(r'C:\Users\User\理財\走勢圖.jpg')
df.to_excel(r"C:\Users\User\理財\林氏家族基金.xlsx")
hold_df_fmtted = hold_df.set_index('Stocks')
hold_df_fmtted.to_excel(r"C:\Users\User\理財\資產持有明細.xlsx")
```



