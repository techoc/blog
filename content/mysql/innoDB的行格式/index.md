---
title: "InnoDB的行格式"
description:
date: 2024-03-25T16:21:45+08:00
image:
tags: [ "MySQL", "InnoDB" ]
categories: [ "MySQL" ]
slug: "innodb-row-format"
---

InnoDB引擎支持四种行格式：

- REDUNDANT
- COMPACT
- DYNAMIC
- COMPRESSED

他们的区别如下：

| Row Format | Compact Storage Characteristics | Enhanced Variable-Length Column Storage | Large Index Key Prefix Support | Compression Support | Supported Tablespace Types      | Required File Format  |
|------------|---------------------------------|-----------------------------------------|--------------------------------|---------------------|---------------------------------|-----------------------|
| REDUNDANT  | No                              | No                                      | No                             | No                  | system, file-per-table, general | Antelope or Barracuda |
| COMPACT    | Yes                             | No                                      | No                             | No                  | system, file-per-table, general | Antelope or Barracuda |
| DYNAMIC    | Yes                             | Yes                                     | Yes                            | No                  | system, file-per-table, general | Barracuda             |
| COMPRESSED | Yes                             | Yes                                     | Yes                            | Yes                 | file-per-table, general         | Barracuda             |

## COMPACT 行格式

COMPACT 中文翻译为紧凑， COMPACT 行格式是 MySQL 5.0 版本之后引入的新行格式。Compact 是一种紧凑的行格式，设计的初衷就是为了让一个数据页中可以存放更多的行记录，从
MySQL 5.1 版本之后，行格式默认设置成 Compact。

![compact.png](compact.png)

## REDUNDANT 行格式

REDUNDANT 中文翻译为冗余， REDUNDANT 行格式是 MySQL 5.0 版本之前的行格式，在 REDUNDANT 行格式中，每列的数据都存储在行尾，并且每列的数据长度都存储在行首。
