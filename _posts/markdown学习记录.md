---
layout:     post
title:      markdown学习记录
subtitle:   
date:       2020-02-26
author:     sugar-foxs
catalog: 	true
tags:
    - markdown
---

本文是学习markdown过程中的一些记录。

<!-- more -->

# 表格
| 列1 |  列2 |
| -- | -- |
|aaa | bbb |
|ccc | dddd |


- 如果表格内字符串太长，需要换行，可使用</br>

| 列1 |  列2 |
| -- | -- |
|aaaaaaaaaaaaa</br>acaa | bbb |
|ccc | dddd |

- 设置表格列的宽度，在表格前加如下代码：
<style> table th:first-of-type { width: 100px; } </style>
| 列1 |  列2 |
| :-- | :-- |
|aaaffff | bbb |
|ccc | dddd |


