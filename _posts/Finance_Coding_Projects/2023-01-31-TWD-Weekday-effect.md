---
layout: "post"
title: USDTWD 的星期因素探討
date: 2022-01-31 14:28:00 +0800
categories: [Coding, Finance project]
tags: [Backtest, Seasonality]
author:
  name: Lung Hung Lin
  link: https://www.linkedin.com/in/lung-hung-blair-lin-645a85194/ 
---
[本文 pdf 連結](https://drive.google.com/file/d/17jDSw-LyaEA8VIltt5PvMbnhGMPL2skK/view?usp=share_link){:target="_blank"}
- [程式碼](#程式碼)
- [摘要](#摘要)
- [一、前言](#一前言)
- [二、資料處理](#二資料處理)
- [三、實證結果](#三實證結果)
- [參考文獻](#參考文獻)

## 程式碼
```python
def dataCollect(twd):
    twd.Taiwan_Date = twd.Taiwan_Date.apply(lambda _: dt.datetime.strptime(_, "%Y/%m/%d"))
    twd.index = twd.Taiwan_Date
    del twd['Taiwan_Date']
    # convert datetime string to the week of the day
    twd['Weekday'] = twd.index.strftime('%a')
    twd = twd[ (twd.index > dt.datetime(2009, 2, 28)) & (twd.index < dt.datetime(2022, 12, 31)) ] 
    twd = twd.rename(columns={"Taiwan_CCY": "TWD"})
    twd = twd[ ['Weekday', 'TWD']] # only need weeday and spot data
    return twd

def calReturn(twd):
    # use previous day's data and today's data to calculate daily return of USDTWD
    twd['TWD_prev'] = twd['TWD'].shift(1)
    twd.dropna(inplace=True)
    twd['Daily_ret'] = np.log(twd['TWD']/twd['TWD_prev']) * 100
    twd = twd[ twd['Daily_ret'] != 0]
    return twd

# basic statistics
def descriptiveStat(twd):
    statDict = {}
    statDict['Max'] = twd.TWD.max()
    statDict['MaxPriceDate'] = twd.loc[twd.TWD == twd.TWD.max()].index[0] # get the date when UDSTWD is the highest
    statDict['Min'] = twd.TWD.min() 
    statDict['MinPriceDate'] = twd.loc[twd.TWD == twd.TWD.min()].index[0] # get the date when UDSTWD is the lowest
    statDict['Median'] = twd.TWD.median()
    statDict['Mean'] = twd.TWD.mean()
    statDict['Std'] = twd.TWD.std()
    print('The descriptive Statistics of the sample: ')
    return statDict

# calculate higher moments of each workday
def calHighMoments(twd):
    weekdays = ['Mon', 'Tue', 'Wed', 'Thu', 'Fri']
    stat_df = pd.DataFrame(columns=weekdays)
    for i in weekdays:
        tmpDict = {} # tmp dictionary to store data
        tmpDf = twd[twd.Weekday == i] # filter each workday
    
        n = tmpDf.shape[0] # this is the sample size
        tmpMean = tmpDf.Daily_ret.mean() # mean of each workday
        tmpStd = tmpDf.Daily_ret.std() # standard deviation of each workday

        for j in tmpDf.Daily_ret: # there is a summation in the formula of Skewness and Kurtosis 
            power3 = np.float64(0.) # summation for skewness
            power4 = np.float64(0.) # summation for kurtosis
            power3 = power3 + ( ((j - tmpMean)**3)/(tmpStd**3) )
            power4 = power4 + ((j - tmpMean)**4)

        # formulas for skewness and kurtosis
        tmpSk = (n/(n-1)/(n-2)) * power3
        tmpKur = power4 / (n-1) / (tmpStd)**4

        tmpDict['Mean'] = tmpMean
        tmpDict['Std'] = tmpStd
        tmpDict['Sk'] = tmpSk
        tmpDict['Kur'] = tmpKur
        stat_df[i] = pd.Series(tmpDict, index=['Mean', 'Std', 'Sk', 'Kur'])
    print('Higher moment statistics of the sample: ')
    return stat_df

# plot the time series data of USDTWD data
def plotTS(twd):
    plt.plot(twd.TWD)
    plt.title("2009.03 to 2022.12 USDTWD Mid Price")
    plt.xlabel("Year")
    plt.ylabel("USDTWD")
    plt.show()
    plt.close() # close the figure

# plot the histogram of each workday's daily return
def plotHist(twd, style):
    if style == 'together': # plot the histogram together
        # 分 30 個箱子 (bins=30)
        twd.groupby('Weekday')['Daily_ret'].hist(bins=30, legend=True)

        # change the legend order to: Mon, Tue, Wed...
        handles, labels = plt.gca().get_legend_handles_labels()
        order = [1, 3, 4, 2, 0]
        plt.legend([handles[i] for i in order], [labels[i] for i in order])

        plt.xlabel("%")
        plt.ylabel("Count")
        plt.title("Daily Return Histogram")
        plt.show()
        plt.close()
        

    if style == 'seperate': # plot each workday's daily return histogram in different plot
        g = twd.groupby('Weekday')['Daily_ret']
        fig, axes = plt.subplots(g.ngroups, sharex=True, figsize=(15, 15))

        for i, (wd, d) in enumerate(g):
            ax = d.plot.hist(x='%', y='Count', ax=axes[i], title=wd, bins=100)
            ax.axvline(d.mean(), c='r', linestyle='--', linewidth=2) # denote the mean in RED line
            ax.axvline(d.median(), c='y', linestyle='--', linewidth=1) # denote the median in YELLOW line
            ax.set_xlabel('%')
            ax.legend().remove()
        fig.tight_layout()
        plt.show()
        plt.close()

def checkDist(twd):
    g = twd.groupby('Weekday')['Daily_ret'].apply(stats.kstest, cdf='norm')
    print('The t-statistics (first value) and the p-value (second value) of the sample: ')
    return g

def testWeekdaySign(twd, wd):
    testRes = {}
    weekdays = ['Mon', 'Tue', 'Wed', 'Thu', 'Fri']
    firstSample = twd[twd.Weekday == wd]['Daily_ret'] # get first sample and we will compare it with four other workdays
    for i in weekdays:
        if i == wd: # if we get the first sample, we should ignore it
            continue
        secondSample = twd[twd.Weekday == i]['Daily_ret']
        testRes[i] = stats.ks_2samp(firstSample, secondSample)[1] # Kolmogorov-Smirnov test
    return testRes
    

def __init__():
    # read the data and rearrange by date
    twd = pd.read_csv('twdSpot.csv').dropna().iloc[::-1]
    twd = dataCollect(twd)
    twd = calReturn(twd)

    print('Complete dataframe: ')
    print(twd)
    print('')

    print(descriptiveStat(twd))
    print('')


    # plot the USDTWD time series
    plotTS(twd)
    print('')

    # calculate skewness and kurtosis and plot them
    print(calHighMoments(twd))
    plotHist(twd, 'together')
    plotHist(twd, 'seperate')
    print('')

    # check if the whole sample comes from normal distribution
    print(checkDist(twd))
    print('')

    # check if each workday comes from the same distribution of other workday
    print('The p-value of whether the two samples come from the same distribution: ')
    print(testWeekdaySign(twd, 'Mon'))
    print(testWeekdaySign(twd, 'Tue'))
    print(testWeekdaySign(twd, 'Wed'))
    print(testWeekdaySign(twd, 'Thu'))
    print(testWeekdaySign(twd, 'Fri'))

import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import datetime as dt
import scipy
from scipy import stats
__init__()
```

## 摘要
本文主要探討台幣外匯市場之每日報酬是否可被星期因素 (day-of-the-week effect)，實證結果各工作日 (星期一至星期五)與其他工作日差異不明顯，未能有足夠信心拒絕「兩樣本來自相同分配」的虛無假設。這與本文參考的論文結果不一致，一個可能是本文使用的是美國開啟量化寬鬆 (Quantitative easing)後之匯率資料，故市場對貨幣價值的評價不同，進而導致在現代匯率市場中星期因素不明顯。

## 一、前言
在證券市場中，星期因素 (day-of-the-week effect)被許多學者研究，該理論指出，投資者在星期五買入證券所獲得的報酬高於在星期一買入，原因是證券市場的交割日 (Settlement Date)制度。以台灣為例，買入證券當稱為交易日 (Trade Day)，但交割買入證券的日期是在交易日的兩天後 (Settlement Date)，由於周末兩天為休假日，故在星期五買入證券時，需要等到下周二才能交割，投資者有多餘兩天將交割款項用於其他市場，例如: 隔夜拆款市場，賺取較高的報酬。

星期因素不僅出現在證券市場，在交易量更為龐大的外匯市場也曾出現，McFarland, Pettit, and Sung (1982)發現對美國投資者來說，在星期一與星期三投資外幣的報酬較高、星期四與星期五的報酬則偏低，Gordon Tang (1998)提出假設，認為即使是同一貨幣對，在不同個工作天 (weekday)的每日報酬並非來自同樣的分配，本文即使用 USDTWD 數據檢驗。

## 二、資料處理
本篇報告使用路透 (Reuters)提供的 USDTWD 每日即期收盤價，並使用中價，資料區間為 1990 年三月至 2022 年十二月。報酬率的定義為 $ln⁡(P_t-P_{t-1} )-1$，其中 $P_t$ 為當日收盤中價、 $P_{t-1}$ 為前一交易日收盤中價。圖一為資料區間內 USDTWD 歷史走勢，可以看到在此期間 USDTWD 在 27.5 至 35.5 之間波動，最高價位 35.168 出現在 2009 年 3 月 3 日，為台幣區間低點，最低價位 27.554 則出現在 2022 年 1 月 18 日，當時在 Fed 持續放出鴿派訊息表明不升息下讓台幣大漲。整個區間 USDTWD 平均匯價為 30.426，標準差為 1.393，敘述型統計資料整理如下表一。

表一

|USDTWD	|值	|日期|
|---|---|---|
|最大值 (台幣相對美元最弱)	|35.168|	2009-03-03
|最小值 (台幣相對美元最強)	|27.554|	2022-01-18
|中位數	|30.241	
|平均數	|30.426	
|標準差	|1.393	

圖一
![pic1](https://lh3.googleusercontent.com/pw/AMWts8BHSVVqwBIy6I5Us7Ww6YuBmrGUDVt-Y_fXGOyaVuPJsHzufXESCOarG3LSvPUfAOdh4AN_i63oThwgXaEgm0ZCgwqMIW5B_CNEqo9MGsAWA1wsdfPedZKCPcZtl_3dUpilSVZj6cuS7cUg12EKvF6a=w382-h278-no?authuser=0)

## 三、實證結果
圖二則為星期一至星期五每日報酬的直方圖，可以看到多數的日報酬介於 -1% 至 1% 區間，極端值也在 -2% 至 1.5% 之內。

圖二
![pic2](https://lh3.googleusercontent.com/pw/AMWts8BKQwC49kjqxKVMwUHDkE9gDe3z6fWUgPvMRBSKNRBdnqHeQuiSrQDCkXGmHhg0HkAhz0n0GdnUQOox1cNfaI_08Gs-nd0HYPEcNGcQ4sOzrNylTL0s4VKaOa7f5YJwcskNVxvFQoWtYMj_ecJ4yhiu=w389-h278-no?authuser=0)

表二為依照星期區分的統計值，除了一般的均值及標準差外，另外加入三階動差-偏態 (skewness)與四階動差-峰態 (Kurtosis)。偏態讓我們得以檢測資料分布是否偏向某一方，其公式為:

$$Skewness= \frac{n}{(n-1)(n-2)}×∑_{i=1}^n\frac{(x_i-\bar x)^3}{s^3}$$

其中 $n$ 為資料個數、$x$ 為 USDTWD 每日報酬、$\bar x$ 為資料區間內 USDTWD 每日報酬的平均值、$s$ 為資料區間內 USDTWD 每日報酬的標準差。通常我們認為只要偏態值不等於 0，即視為有偏態傾向，若資料的平均數大於中位數，則資料為右偏 (right-skewed)，反之，若資料的平均數小於中位數，則資料為左偏 (right-skewed)。

峰態用來衡量手中資料與常態分布相比是較為高峻或是平坦，通常認為若峰態不等於 0 即視為有偏態，大於 0 代表手中資料較常態分配來的高瘦、小於 0 代表較常態分配來的低寬。峰態的公式如下:
$$Kurtosis=\frac{1}{(n-1)×s^4}×∑_{i=1}^n(x_i-\bar x)^4 $$

表二
|星期一	|星期二	|星期三	|星期四	|星期五|
|---	|---|---	|---|---|
|平均數	|-0.013742	|0.017944	|0.001231	|-0.008164	|-0.015501
|標準差	|0.257144	|0.291845	|0.225287	|0.235711	|0.280908
|偏態	|0.000961	|-0.000045	|0.000003	|0.000423	|-0.000005
|峰態	|0.000840	|0.000014	|0.000000	|0.000282	|0.000000

由表二可以看出，星期一的平均報酬率最大，平均報酬接近 2%，報酬最低者為星期五，平均報酬為 -1.5%。報酬標準差則大致相同，不管在哪一天交易，投資者均面對約 22 至 29% 的波動。偏態與峰態則不明顯，但大致來說，USDTWD 各工作的報酬較標準常態分佈高瘦 (峰態不小於 0)。圖三顯示各工作日的歷史報酬分布，紅色虛線為報酬平均數、黃色虛線為報酬中位數，兩者在各工作日中幾乎相同，顯示資料區間內的偏度不明顯。

圖三
![pic3](https://lh3.googleusercontent.com/pw/AMWts8D9ZGeifEjK10MxBeXIBslKOOXp_JyQyGrUn7Uj8dfnGf1V796lGWXDFdN7QhlXaI0Bfc9q3CKRjZJts10LCNaNrVByRQwVC3tWyhzwIQ2B4opgtfgHw143KVvdfWnXwDKgSPrUj7hNBQhJfO-0OnVa=s893-no?authuser=0)

然而，有必要對資料分布做更精確的檢定，在此使用 Kolmogorov-Smirnov 檢定來檢測各工作日報酬的分布是否符合常態分布，Kolmogorov-Smirnov 檢定是一種無母數檢定法 (Non-parametric test)，用於不確定樣本是何種分配的情況。將每個工作日的每日報酬作為一個樣本，並依序檢驗其與其他工作日樣本是否來自同樣的母體分配，若 p-value 小於 0.05，則拒絕雙尾檢定的虛無假設: 「兩樣本來自相同母體分配」，表四為各工作日對其他工作日之 Kolmogorov-Smirnov 檢定的 p 值，粗體表示在 5 % 的信心水準下無法被拒絕的情況。

由表四可以看到，除了同工作日與同工作日無須檢定外，均無足夠信心拒絕「兩樣本來自相同分配」的虛無假設，顯示星期一與星期五對其他工作日基本上來自相同分配，這與 Gordon Tang 的檢驗結果不一致。

表四
![pic4](https://lh3.googleusercontent.com/pw/AMWts8ATST56brc1cyxWSOUdRKEHIasHe0Svy17h_7Ngfy5VXCIM6USuPI1OngwVAB4Wl7jLGBYKc0tx2epp89lZDRQtThFLDjetX1iv1IH4j3dgSScziGSCQ0KuAw5w43r5OBMheV1qw48-9tniQl7xDDrU=w727-h227-no?authuser=0)



## 參考文獻
Gordon Tang, 1998. "Weekly Pattern of Exchange Rate Risks: Evidence from Ten Asian-Pacific Currencies," Asia-Pacific Financial Markets, Springer;Japanese Association of Financial Economics and Engineering, vol. 5(3), pages 261-274, November.

McFarland, J., Pettit, R., and Sung, S. (1982), The distribution of foreign exchange price changes: Trading day effects and risk measurement, J. Finance 37, 693–715.




