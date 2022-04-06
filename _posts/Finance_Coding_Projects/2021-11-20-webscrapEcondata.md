---
layout: "post"
title: 取得主要經濟體總經數據 
date: 2021-11-20 14:07:00 +0800
categories: [Coding, Finance project]
tags: [WebScrape]
author:
  name: Lung Hung Lin
  link: https://www.linkedin.com/in/lung-hung-blair-lin-645a85194/ 
---
- [相關連結](#相關連結)
- [結果](#結果)
  - [Raw Data](#raw-data)
  - [整理後資料](#整理後資料)
- [程式碼](#程式碼)
  
## 相關連結
[專案資料夾](https://drive.google.com/drive/folders/1-HbV4wOQU1Cqv8vSDy7BExcpiolkAa-1?usp=sharing){:target="_blank"}

## 結果

### Raw Data
![Raw Data](https://lh3.googleusercontent.com/pw/AM-JKLXE2T90bSMKAzCtjex_xLjVelglGf9baZA--6HshSsdFA8S002oWYky9XoScSeM0tSOS6whZ9nXfpzQio7STtCMABe3D-l4nZgWInV6HJcksV56Iy60UmMbFLk7_rPq8t382oT0KQ1Iawhhm1wGf_S5=w1160-h893-no?authuser=0)

### 整理後資料

![to excel Data](https://lh3.googleusercontent.com/pw/AM-JKLV2mX8nEyq4ICwLAzw08bKhBe5L-B5aMXVX7INgLhOXfLUI_kLLB3ehf_Roj2sBOM07n76tIQAzOmPlvOvzXtfItcKbfuYOjSTM3UEVAFlm-Ty4Yu5YBeLKmU5ckBZi_NRb8nZ0lby9-B-_KPYFxtw9=w927-h893-no?authuser=0)
## 程式碼
```python
import pandas as pd
import requests
from bs4 import BeautifulSoup
# scrap the data from yFinance
headers = { 
 'User-Agent' : 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 \ 
 (KHTML, like Gecko) Chrome/71.0.3578.98 Safari/537.36', 
 'Accept' : 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8', 
 'Accept-Language' : 'en-US,en;q=0.5',
 'DNT' : '1', # Do Not Track Request Header 
 'Connection' : 'close'
}
```

```python
from datetime import date
today = str(date.today())
url = 'https://finance.yahoo.com/calendar/economic?day=' + today
page = requests.get(url, headers=headers, timeout=5)
page_content = page.content
soup = BeautifulSoup(page_content, 'html.parser')

# two kinds of table from yFinance
code1 = "simpTblRow Bgc($hoverBgColor):h BdB Bdbc($seperatorColor) Bdbc($tableBorderBlue):h
H(32px) Bgc($lv1BgColor)"
code2 = "simpTblRow Bgc($hoverBgColor):h BdB Bdbc($seperatorColor) Bdbc($tableBorderBlue):h
H(32px) Bgc($lv2BgColor)"
rs1 = soup.find_all("tr", {"class":code1})
rs2 =soup.find_all("tr", {"class":code2})
rs = rs1 + rs2
```

```python
# only get data we need
import ast
import re
need_nat = ['US', 'EU', 'CH', 'DE', 'GB', 'JP', 'CA', 'AU', 'NZ']
res_list = []
for e in rs:
    cty = e.find("td", {"aria-label":"Country"}).text
    if cty in need_nat:
        temp_dict = dict()
        temp_dict['Nation'] = cty
        temp_dict['Event'] = e.find("td", {"aria-label":"Event"}).text
        time = e.find("td", {'aria-label':'Event Time'}).text
        actual_data = e.find("td", {"aria-label":"Actual"}).text
        cens_data = e.find("td", {"aria-label":"Market Expectation"}).text
        temp_dict['Time'] = time

        try:
            temp_dict['Actual'] = float(actual_data)
            temp_dict['Cens'] = float(cens_data)
        except ValueError as ve:
            if actual_data == '-' and cens_data == '-':
                temp_dict['Actual'] = actual_data
                temp_dict['Cens'] = cens_data
            if actual_data != '-' and cens_data == '-':
                temp_dict['Actual'] = float(actual_data)
            if actual_data == '-' and cens_data != '-': # actually impossible to hapend
                temp_dict['Cens'] = float(cens_data)
 
        res_list.append(temp_dict)
```

```python
# create a df and calculate additional info
df = pd.DataFrame(res_list)
tmp_list = []
for i in range(df.shape[0]):
    tmp_list.append(None)
tmp_df = pd.DataFrame(tmp_list)

for (idx, row) in df.iterrows():
    try:
        tmp_df.loc[idx, 'Dev'] = float(row.Actual) - float(row.Cens)
        tmp_df.loc[idx, 'Dev_pct'] = round(((float(row['Actual']) / float(row['Cens']) ) - 1) * 100, 1)
    except Exception as e:
        tmp_df.loc[idx, 'Dev'] = '-'
        tmp_df.loc[idx, 'Dev_pct'] = '-'
df['Dev'] = tmp_df['Dev']
df['Dev_pct'] = tmp_df['Dev_pct']


# transfer list of dicts to a dataframe and sort values according to the need_nat order)
from pandas.api.types import CategoricalDtype
cat_nation_order = CategoricalDtype(
 ['US', 'EU', 'DE', 'CH', 'GB', 'JP', 'CA', 'AU', 'NZ'],
 ordered = True
)
df['Nation'] = df['Nation'].astype(cat_nation_order)
df.sort_values('Nation', inplace=True)

# get the numbers of econ data for each country that day 
df_groupby = df.copy()
sector = df_groupby.groupby('Nation')
sector_dict = dict(sector.size())

# make the filename
from datetime import timedelta
week_end = str(date.today() + timedelta(days=6))
if date.today().weekday() == 0:
    xlBookNameStart = today
    xlBookNameEnd = week_end
    xlBookName = f'{xlBookNameStart}_{xlBookNameEnd}'
```

```python
# use xlsxwriter to convert dataframe to a .xlsx file
import xlsxwriter
from xlsxwriter import Workbook
ordered_list = df.columns.to_list()
if date.today().weekday() == 0: # 如果今天是禮拜一，就新增一個新的檔案
    wb = xlsxwriter.Workbook(f'D:\Web_scraper\Get_Econ_Data\his_data\{xlBookName}.xlsx')
else:
    this_week = 'example'
    wb = Workbook(f'D:\Web_scraper\Get_Econ_Data\his_data\{this_week}.xlsx')

ws = wb.add_worksheet(today)
first_row = 0 # set the column header
for header in ordered_list:
    col=ordered_list.index(header)
    ws.write(first_row, col, header)
row = 1
for data in df.iterrows():
    tmp = data[1].to_dict()
    for _key, _value in tmp.items():
        col = ordered_list.index(_key)
        try:
            ws.write(row, col, _value)
        except:
            pass
    row += 1

# merge the cells with the same nation
merge_format = wb.add_format({
 'bold': 1,
 'border': 1,
 'align': 'center',
 'valign': 'vcenter',
 'fg_color': 'yellow'})

# see every val in sector_dict (this will return key country's econ data number)
start_row = 2
for key, val in sector_dict.items():
    if val == 0:
        continue
    if val == 1:
        ws.write_string(start_row-1, 0, key, merge_format)
    if val != 1:
        ws.merge_range(f'A{start_row}:A{start_row+(val-1)}', key, merge_format)
    print(f'A{start_row}:A{start_row+(val-1)}')
    start_row += val

ws.set_column('B:B', 30) # expand B column (store the name of the econ data) to 30
ws.set_column('C:C', 20) 
wb.close()
```
