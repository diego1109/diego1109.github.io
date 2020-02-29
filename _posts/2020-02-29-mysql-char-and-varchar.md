---

layout: post
title: char 和 varchar 类型
date: 2020-02-29
categories: 学习笔记
tags: mysql
comments: true 

---

[原文在此](https://dev.mysql.com/doc/refman/5.7/en/char.html)

`char` 和 `varchar` 类型很相似，但在存储和读取方式、支持的最大长度、是否保留尾随空格这些方面有很大不同。

char 和 varchar类型在创建时声明打算存储的最大字符个数，例如：char(30)表示最大可存储30个字符。

char类型列的长度在建表时就已经通过声明固定下来了，并且该长度在0-255之间。当char类型的值在存储时，字符串末尾如果有空格也依旧会存储起来，比如：char值为'diego  '。当SQL开启 `PAD_CHAR_TO_FULL_LENGTH`模式时，字符串的尾随空格会被读取，否则尾随空格会被去掉。

varchar类型列存储可变长度的字符串，且字符串的长度在0-65535之间。varchar的有效最大长度取决于行的最大尺寸（每行最大尺寸65535bytes，这些字节被所有的列共享）和使用的字符集种类（latin1、utf8等等）。

与char不同，varchar需要1byte或者2byte的空间存储字段所占的字节数。当字段所占的字节数小于255时，需要1byte空间存储字节数，当字段所占的字节数大于255是，需要2byte空间存储。

如果 `strict SQL mode` 没有开启，当分配给char或者varchar列的值超过了最大长度限制，这个值会被mysql自动截断来适应列的长度并且发出 warning 。对于非空格字符串截断，可以通过开启严格sql模式，使mysql报异常（而不是警告）并且静止该值的插入。推荐开启该模式。

对于varchar列，无论使用哪种SQL模式，插入前都会截断超出列长度的尾随空格，并生成警告。 对于char列，无论SQL模式如何，都将以静默方式执行从插入值中截断多余尾随空格的操作。

下面这张表通过存储不同的字符串到char(4)列和varchar(4)列的结果展示这两种类型的不同。假设列采用单字节字符集：latin1。`（latin1: 一个字符占一个字节，utf8：一个字符占三个字节）`

| value      | char(4) | Storage required | varchar(4) | Storage required |
| ---------- | ------- | ---------------- | ---------- | ---------------- |
| ' '        | '    '  | 4 bytes          | ' '        | 1 byte           |
| 'ab'       | 'ab  '  | 4 bytes          | 'ab'       | 2 byte           |
| 'abcd'     | 'abcd'  | 4 bytes          | 'abcd'     | 5 byte           |
| 'abcdefgh' | 'abcd'  | 4 bytes          | 'abcd'     | 5 byte           |