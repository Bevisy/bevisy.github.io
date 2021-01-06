---
title: "The Skills of VIM"
date: 2020-07-24T15:02:12+08:00
draft: false
tags: 
    - Vim
categories:
    - Linux
---
# VIM 使用技巧

## 查找

### 大小写敏感控制

```sh
# VIM 默认大小写敏感查找；

# 大小写不敏感查找
：/foo\c
# 大小写敏感查找
：/foo\C
```

### 查找光标当前单词

```sh
# normal 模式下，"*" 键查找当前单词
```



## Vim 查找和替换

```sh
:{作用范围}s/{目标}/{替换}/{替换标志}

# 当前行替换
：s/vivian/sky/ 替换当前行第一个 vivian 为 sky 
：s/vivian/sky/g 替换当前行所有 vivian 为 sky 

# 范围替换； n 为数字，若 n 为 .，表示从当前行开始到最后一行 
：n,$s/vivian/sky/ 替换第 n 行开始到最后一行中每一行的第一个 vivian 为 sky 
：n,$s/vivian/sky/g 替换第 n 行开始到最后一行中每一行所有 vivian 为 sky 
：2,12s/hello/world/ 2-12行,第一次出现替换
：.,+2s/hello/world/ 当前行与接下来的2行，第一次出现替换

# 全行替换
：%s/vivian/sky/（等同于 ：g/vivian/s//sky/） 替换每一行的第一个 vivian 为 sky 
：%s/vivian/sky/g（等同于 ：g/vivian/s//sky/g） 替换每一行中所有 vivian 为 sky 
 
# 使用"#"作为分隔符，此时中间出现的 / 不会作为分隔符 
：s#vivian/#sky/# 替换当前行第一个 vivian/ 为 sky/ 
：%s+/oradata/apras/+/user01/apras1+ （使用+ 来 替换 / ）： /oradata/apras/替换成/user01/apras1/ 

# 替换确认，添加参数c，提示是否替换
:%s/hello/world/gc 针对每一个匹配项，提示是否替换
```


