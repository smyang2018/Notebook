---
title: VCF文件操作工具 
date: 2017/10/12
tags: BCF
notebook: 工具笔记
categories: HTSToolkit
comments: true
---

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

# VCF/BCF文件格式及其处理工具

## VCF文件格式

## BCFtools

BCFtools是一套处理VCF和BCF格式的工具。它有提供许多子命令实现不同功能，我个人用的比较多有以下几个：

- mpileup + call:  根据参考基因组寻找变异位点
- view: 选取，过滤以及VCF/BCF之间的格式转换。 这个命令完成了`convert`,`filter`,
- query: VCF/BCF格式输出为更适合人类阅读的格式
- merge: 将多个VCF/BCF文件整合成一个
- isec: 求不同VCF/BCF文件的交集，合集和补集

### 通用参数

在单独介绍每个命令之前，需要了解一下所有子命令都可以用的参数：

文件输出： `-o, --ouput FILE`，默认输出到标准输出，通过该选项指定文件。 `-O, --output-type b|u|z|v`： 输出为压缩的BCF(b)， 未压缩的BCF(u), 压缩的VCF(z), 未压缩的VCF(v). 使用`-Ou`能够让bcftools命令间的操作更快。

- FILE：输入文件，可以是VCF或BCF，以及这些文件对应的BGZIP压缩形式。如果是`-`, 则认为是标准输入。有些工具需要tabix或CSI的索引文件。

**mpileup**和**call**是一套组合，最基本的用法为:

```shell
bcftools mpileup -Ou -f reference.fa alignments.bam | bcftools call -mv -Ob -o calls.bcf
```

根据不同情况可以添加`call`的参数，比如说 `bcftools call -P 1.1e-3`。

**view**一般要配合`-f LIST`参数共同使用，能有效对VCF/BCF文件内容进行筛选。比如说仅选择非indel, 且ref的reads数小于1， 深度在20和100之间。

```shell
bcftools view -i 'TYPE!="indel" && (DP4[0]+DP4[1])<1 && DP >20 && DP < 100'
```

有效表达式包括：

- 数值常量，字符常量和文件名: `1,1.0,1e-4`，`"string"`，`@filename`
- 算术运算符：`+, *, -, /`
- 比较操作符：`==, >, >=, <, <=, !=`
- 正则表达式：`INFO/HAYSTACK ~"needle/i"`
- 括号： `()`
- 逻辑运算符： `&&, ||`
- INFO标签和FORMAT标签以及列名

```shell
INFO/DP 或 DP
FORMAT/DV, FMT/DV, 或 DV
FILTER, QUAL, ID, POS, REF, ALT[0]
```

- 1或0用于判断flag是否存在
- "."则是判断是否有缺失值
- 样本基因型： 纯合("hom")，杂合("het")，单倍体("hap")，alt-alt纯合("AA")，ref-alt杂合("RA")，alt-alt杂合("Aa")，单倍体参考("R")，单倍体替换("A")
- REF/ALT列的变异类型(TYPE)： indel, snp, mnp, ref, bnd, other
- 数组下标： (DP[0]+DP[1])/(DP[2]+DP[3]) > 0.3。 其中`*`表示任意，`-`表示返回
- FORMAT和INFO标签的函数: MAX, MIN, AVG, SUM, STRLEN, ABS
- 运行过程中新增变量：`N_ALT, N_SAMPLES, AC, MAC, AF, MAF, AN, N_MISSING, F_MISSING`

**query**可以将VCF/BCF文件转换成更加人类可读的格式，依赖于`-f FORMAT`参数。

其中**FORMAT**可以是：

- 所有的列名：%CHROM, %POS, %ID, %REF, %ALT, %QUAL, %FILTER, 
- INFO列的其中一个：%INFO/标签（如INFO/DP4），此外标签是多值结果，能用`{}`进一步选取，例如`DP4{1}`
- 基因型: "%GT", "%TGT"
- 换行符和制表符:"\n","\t"

举个例子：

```shell
bcftools query -f '%CHROM\t%POS\t%REF\t%ALT\t%DP4{0}\t%DP{1}\t%DP{2}\t%DP{3}\n' calls.bcf -o call.delim
```

输出结果就能直接导入到R,Python进行分析。

## VCFtools

VCFtools: 用于描述性统计数据，计算数据，过滤数据以及数据格式转换。

基本用法：

```shell
vcftools [--vcf VCF文件 | --gzvcf gz压缩的VCF文件 --bcf BCF文件] [--out OUTPUT PREFFI]
```

他能做的事情：
1. 输出第一条染色体的所有位点等位基因频率
2. 从输入文件中仅保留SNP位点
3. 输出两个vcf文件的比较结果
4. 标准输出不含有filer tag的位点，并且以gzip压缩
5. 计算每个位点的hardy-weinberg p-value，这些位点不包括缺失的基因型
6. 计算一系列核酸多态性

常用参数如下：

- 和输入输出有关
```shell
--vcf, --gzvcf, --bcf：根据输入文件格式进行选择
--out, --stdout, -c --temp: 选择合适的输出方式
```

- 位点筛选(site filtering)有关参数
```
# 根据位置过滤
--chr/--not-chr Chr1: 选择染色体
--from-bp/--to-bp: 选择碱基范围
# 根据第三列ID进行过滤，不常用
--snp rsID
# 根据变异类型
--keep-only-indels: 仅保留INDEL
--remove-indels: 仅保留SNP
# 根据FILTER列进行过滤
--remove-filtered-all: 移除filter tag位点
--keep-filtered/--remove-filtered
# 根据INFO列进行过滤
--keep-INFO/--remove-INFO 目标类型
# 根据等位基因过滤
--maf/--max-maf # Minor Allele Frequency
--mac/--max-mac # Minor Allele Count
```

- 样本样本参数（我不常用）
- 基因型过滤参数（没怎么用）
- 输出VCF选项
```
--recode
--recode-INFO-all
--diff-site: 比较位点
--hardy: hardy-weinberg p值
--max-missing: 基因型缺失
--site-pi
```