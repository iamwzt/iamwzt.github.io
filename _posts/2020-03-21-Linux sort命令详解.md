---
layout:     post
title:      Linux sort命令详解
subtitle:   
date:       2020-03-21
author:     iamwzt
header-img: img/post-web.jpg
catalog: true
tags:
    - Linux
    - sort
---

## Linux sort 命令详解
> 原文 ： [linux sort 命令详解](https://www.cnblogs.com/51linux/archive/2012/05/23/2515299.html)

### 何为 sort 
sort将文件的每一行作为一个单位，相互比较，比较原则是从首字符向后，依次按ASCII码值进行比较，最后将他们按升序输出。

### 基本使用

```sh
sort [-bcCdfghiRMmnrsuVz] [-k field1[,field2]] [-S memsize] [-T dir] [-t char] [-o output] [file ...]
```

比如，有一个文档是这样的：
```sh
➜  linux_cmd cat seq.txt
banana
apple
pear
orange
orange
```
排序后：
```sh
➜  linux_cmd sort seq.txt
apple
banana
orange
orange
pear
```

### 加点参数

#### 1. -u 去重
上面的文件中有两个 orange，加了 -u 后：
```sh
➜  linux_cmd sort -u seq.txt
apple
banana
orange
pear
```
u，就是 `unique` 了

#### 2. -r 反向
```sh
➜  linux_cmd sort -r seq.txt
pear
orange
orange
banana
apple
```
r，就是 `reverse` 了，默认升序，现为降序

#### 3. -o 输出到文件
输出到文件，用重定向符号也可以，但是想原文件修改，就必须要 `-o` 了：
```sh
➜  linux_cmd sort seq.txt -o seq.txt
➜  linux_cmd cat seq.txt
apple
banana
orange
orange
pear
```
o，就是 `output` 了，并且是可以输出到原文件的那种

#### 4. -n 按数字排序
如果要排序的列是数字，那么直接排会出现问题，因为 sort 默认是按照字符串处理的：
```sh
➜  linux_cmd sort seq1.txt
10
13
2
8
9
```
此时可以加 -n 选项来声明按数字处理：
```sh
➜  linux_cmd sort -n seq1.txt
2
8
9
10
13
```
n，就是 `number` 了

#### 5. -k 和 -t 指定列进行排序
若有一个文件是这样的：
```sh
➜  linux_cmd cat seq.txt
apple:10:6.5
banana:5:3.5
orange:7:5
pear:8:5.5
```
若想以第二列进行排序，则需：
```sh
➜  linux_cmd sort -k 2 -t : -n seq.txt
banana:5:3.5
orange:7:5
pear:8:5.5
apple:10:6.5
```
这里，`-k` 指明了要以第2列进行排序，然后 `-t` 指明了要以 “:” 作为分隔符将每行进行切分（默认是任意个空格，换行符`\t`也行）

#### 6. 更多选项
-f 会将小写字母都转换为大写字母来进行比较，亦即忽略大小写

-c 会检查文件是否已排好序，如果乱序，则输出第一个乱序的行的相关信息，最后返回1

-C 会检查文件是否已排好序，如果乱序，不输出内容，仅返回1

-M 会以月份来排序，比如JAN小于FEB等等

-b 会忽略每一行前面的所有空白部分，从第一个可见字符开始比较。

更多选项可参考 man 手册

### -k 的高级用法
有时候学习脚本，你会发现sort命令后面跟了一堆类似 `-k1,2`，或者 `-k1.2 -k3.4` 的东东，有些匪夷所思。今天，我们就来搞定它—-k选项！

首先准备一个文件：
```sh
banana 5 3.5
orange 8 5
pear 8 4.5
apple 10 6.5
```
其中第一列是水果名，第二列是个数，第三个是单价。现在想 **按个数排序** ：
```sh
➜  linux_cmd sort -k 2 -n seq.txt
banana 5 3.5
orange 8 5
pear 8 4.5
apple 10 6.5
```
发现 orange 和 pear 个数相同，但是 orange 在前。这是因为默认情况下，当排序的列相同时，再按顺序从第一列开始排。如果我想 **个数相同就按价格来排** 的话，可以这样：
```sh
➜  linux_cmd sort -k 2 -k 3 -n seq.txt
banana 5 3.5
pear 8 4.5
orange 8 5
apple 10 6.5
```
是的，k 选项可以有多个。

那如果想 **个数相同，就按价格倒序来排** 的话。。。
```sh
➜  linux_cmd sort -k 2 -k 3r -n seq.txt
banana 5 3.5
orange 8 5
pear 8 4.5
apple 10 6.5
```
这里有个细节：`-k 3r`，这里的 r 其实和 -r 是一样的作用，只不过有多列，就这么处理来单独配置某一列，同样 -n 也可以这么处理：
```sh
➜  linux_cmd sort -k 2n -k 3rn seq.txt
banana 5 3.5
orange 8 5
pear 8 4.5
apple 10 6.5
```

最后再来看看 -k 的高级用法，首选需要了解-k选项的语法格式，如下：

`[ FStart [ .CStart ] ] [ Modifier ] [ , [ FEnd [ .CEnd ] ][ Modifier ] ]`

- 这里的 Modifier 就是上面说的 `-k 3rn` 里的 `r/n` 了。
- CStart 可以理解为 offset 吧，不设的话默认为列首，比如 -k 3，就代表第三列的第一个字符开始；
- FEnd 代表排序的列范围，那不设就默认为到最后一列为止。（其实是不对的，可以看下面）
- CEnd 同 CStart

比如有这么个文件：
```sh
➜  linux_cmd cat seq.txt
apple 10 6.5
banana 5 3.5
paear 8 4.5
orange 8 5
```
想要 **按第1列，从第2个字母起进行排序** ，那么：
```sh
➜  linux_cmd sort -k 1.2 seq.txt
paear 8 4.5
banana 5 3.5
apple 10 6.5
orange 8 5
```
会发现 paear 因为第三个字母比 banana 靠前，所以整行都在前面了，如果 **只想按第2个字母排序**，那么：
```sh
➜  linux_cmd sort -k 1.2,1.2 seq.txt
banana 5 3.5
paear 8 4.5
apple 10 6.5
orange 8 5
```

再说一下 Modifier ，可以用到b、d、f、i、n 或 r。
- 其中n和r你肯定已经很熟悉了。
- b表示忽略本域的签到空白符号。
- d表示对本域按照字典顺序排序（即，只考虑空白和字母）。
- f表示对本域忽略大小写进行排序。
- i表示忽略“不可打印字符”，只针对可打印字符进行排序。（有些ASCII就是不可打印字符，比如\a是报警，\b是退格，\n是换行，\r是回车等等）

再说说”跨列“的事，既然k选项可以设定 FStart 和 FEnd，是否可以将两个列合起来比较呢？比如说有这么一个文件：
```sh
➜  linux_cmd cat seq3.txt
2 100 200
1 101 300
5 101 200
4 100 300
```
这样来排序：
```sh
➜  linux_cmd sort -k 2.2n,3.1n seq3.txt
2 100 200
4 100 300
1 101 300
5 101 200
```
按理说，”5 101 200“应该比”1 101 300“小才对啊，其实这里有一个坑，跨列是不存在的，当比较完第2列后，就自动去比较第1列了。。。值得注意。

