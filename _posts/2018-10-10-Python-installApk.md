---
layout: post
title:  "使用Python实现apk自动安装"
date:   2018-10-10 14:42:54
categories: Python
tags: Python脚本 
---

* content
{:toc}

本文记录如何使用Python自动批量安装apk。





利用`Python`执行`ADB`命令 `adb install xxx`


```python
import os

folder = 'C:/users/17267/Desktop/debug/'
files = os.listdir(folder)
for file in files:
    if file[len(file) - 3:len(file)] == "apk":
        instructions = 'adb install ' + folder + file
        print("执行命令", instructions)
        os.system(instructions)
```
