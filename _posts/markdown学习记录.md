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
```
<style> table th:first-of-type { width: 100px; } </style>
```

# 段落
- 段落的换行是使用两个以上空格加上回车。示例：
- fggfg  
kkkk

## 字体
- 文字两边各1个\*或者\_,字体会变成斜体  
示例： `*`斜体`*` , \_斜体\_  
效果： *斜体* , _斜体_

- 文字两边各2个\*或者\_,字体会变成粗体  
示例：`**`粗体`**` , \_\_粗体\_\_  
效果：**粗体** , __粗体__

- 文字两边各3个\*或者\_,字体会变成粗斜体  
示例：`***`粗斜体`***` , \_\_\_粗斜体\_\_\_  
效果：***粗斜体*** , ___粗斜体___

## 分割线
- 一行中用三个以上的星号、减号、底线来建立一个分隔线，行内不能有其他东西。你也可以在星号或是减号中间插入空格。  
`***`  
`---`  
`* * *`  
`- - -`  
效果：  
***  
---
* * *
- - -

## 删除线
- 只需要在文字的两端加上两个波浪线\~\~。  
- 示例：\~\~删除线\~\~ 
- 效果：~~删除线~~

## 下划线
- 下划线可以通过HTML的\<u\>\</u>标签来实现。  
示例： \<u>下划线\</u>  
效果：<u>下划线</u>

## 脚注
- 示例:   
`[^1]`  
`[^1]:gch`


# 列表
## 无序列表
- 使用星号(*)、加号(+)或是减号(-)作为列表标记。我一般使用-。示例：  
`- 第1`  
`- 第2`  
`- 第3`   
效果：  
- 第1 
- 第2  
- 第3 

## 有序列表
- 使用数字并加上 . 号来表示。示例：  
`1. gg`  
`2. ff`  
`3. fff`  
效果：  
1. gg 
2. ff  
3. ff

# 链接
- `[链接名称](链接地址)`或者`<链接地址>`
- 示例：  
`[百度](http://www.baidu.com)`  
`<http://www.baidu.com>`
- 效果：  
[百度](http://www.baidu.com)   
<http://www.baidu.com>

# 时序图
- 在VScode中下载插件Markdown Preview Enhanced插件。
- 示例：
\```sequence
title:communication
participant main
participant FuncA as A
participant FuncB as B
A-->B:
B->main:sendmessage
Note over A:NOTE_A
Note right of B:NOTE_B
\```

效果：
![时序图.jpg](http://ww1.sinaimg.cn/large/dbf344a4ly1gcef6pnpqrj20zc0zc0vl.jpg)




