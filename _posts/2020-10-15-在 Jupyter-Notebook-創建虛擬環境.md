---
layout: "post"
title: 在 Jupyter Notebook 創建虛擬環境
date: 2020-10-15 17:00:00 +0800
categories: [Coding, Environment]
tags: [Jupyter Notebook]
author:
  name: Lung Hung Lin
  link: https://www.linkedin.com/in/lung-hung-blair-lin-645a85194/ 
---

####                                                                                                      																																	林隆鴻 @政大數理資訊社 2020/10/15
  
- [打開 ```anaconda prompt```](#打開-anaconda-prompt)
- [創建一個虛擬環境(```virtual environment```)](#創建一個虛擬環境virtual-environment)
  - [虛擬環境的好處](#虛擬環境的好處)
  - [在 ```anaconda prompt``` 創建虛擬環境](#在-anaconda-prompt-創建虛擬環境)
  - [Install ```ipykernel```](#install-ipykernel)

------
## 打開 ```anaconda prompt```

首先，在搜尋欄中打入```anaconda prompt```。

你會看到 ```anaconda``` 為我們預設的路徑，以我的電腦來說會出現: ```(base) C:\Users\User>```。

Useful tips:

- 上鍵回到上一個指令。下鍵回到下一個指令。
- ```cd``` 更改目前路徑。
  - ```cd ..``` 回到上一層路徑
- ```cls```: 清除視窗。[```macOS```: ```clear```]

## 創建一個虛擬環境(```virtual environment```)

### 虛擬環境的好處
  - 不同專案可以使用不同版本的相同套件。
  - 為你的專案創造一個獨立的工作平台。
  - 不會汙染電腦的環境。

  ![so many modules](https://lh3.googleusercontent.com/pw/AM-JKLXOJgEtRbEf5q8brG-4DBP39YLBZdX9bHrthr37JBZ1gv8DM5AYqxOV5OqLvHFEx1IG4mvkxs0uwjcwNF5pC1e4UJ11AELdCw4fwihEZ9tplwItjuaGT1Ib7ZQE6_8RehkmS8kWAxxVV_uklig-a_ra=w655-h582-no?authuser=0)
  
### 在 ```anaconda prompt``` 創建虛擬環境

  1. ```conda env list``` :列出現有虛擬環境。

  2. ```conda create  --name testENV```: 創建名稱為 ```testENV``` 的虛擬環境。

  3. ```conda activate testENV```: 開啟```testENV```虛擬環境。

     [如果是 ```macOS```]，則輸入: ```source activate testENV```。

     此時命令列會出現:   ```(testENV) C:\Users\User>```。

  4. ```conda list```: 查看目前的虛擬環境有哪些套件。

  5. ```conda install python```: 安裝 ```python```。

     - 會出現 ```Proceed ([y]/n)?```。就在後面打 ```y```。

     - 在 command line 裡，要看到 ```(base) C:\Users\User>``` 才代表你上一個指令已執行完畢。

  6. ```conda deactivate```: 離開虛擬環境。  

      重新回到: ```(base) C:\Users\User>```。

  7. ```conda env remove --name testENV```: 刪除```testENV``` 這個虛擬環境。

### Install ```ipykernel```

  1. 在 ```anaconda prompt``` 輸入:```pip install --user ipykernel```

  2. 繼續輸入: ```python -m ipykernel install --user --name=testENV```。

     此時會出現:   

     ```Installed kernelspec testENV in C:\Users\User\AppData\Roaming\jupyter\kernels\testenv```。  

     或是你當初 ```jupyter```的存放位置。

  3. 在我們的 ```anaconda virtual environment``` 中加入 ```ipykernel```:

     ```conda install -n testENV ipykernel```。

  4. 最後一步!

     輸入: ```jupyter notebook```。此時 jupyter notebook 會連接到你的瀏覽器。

     切換 ```kernel``` 到我們剛創立的 ```testENV``` 虛擬環境。

     ![](data:image/png;base64,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)

     [補充]:

     - ```jupyter kernelspec list```: 查看目前 ```jupyter``` 有哪些 ```kernel```。

     - ```jupyter kernelspec remove testENV```: 刪除 ```testENV``` 這個 ```kernel```。



