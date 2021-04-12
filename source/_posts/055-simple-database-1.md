---
title: Let's Build a Simple Database - Introduction and Setting up the REPL
date: 2021-03-27 21:12:13
tags:
- 翻译
- Database
categories:
- 翻译
---

> 原文地址：https://cstack.github.io/db_tutorial/parts/part1.html
> 原文作者： [cstack](https://github.com/cstack)
> 译者：[Win-Man](https://github.com/Win-Man)

## 介绍并设置 REPL

作为一个 Web 开发人员，我每天的工作中都会使用到关系型数据库，但是这些数据库对于我来说就是一个黑盒。因此我有一些疑问：

* 在内存中、在磁盘上数据存储格式是怎么样的？
* 什么时候数据库从内存转移到磁盘？
* 为什么每个表只能有一个主键？
* 事务回滚是如何工作的？
* 索引的格式是怎么样的？
* 什么时候会发生全表扫以及全表扫是如何工作的？
* 预处理语句是以什么形式存储的？

简而言之，问题就是数据库是怎么工作的？

为了搞清楚这些问题，我从头开始写了一个数据库。它是模拟 sqlite 数据库实现的，因为 sqlite 设计的就是一个比 MySQL 或者 PostgreSQL 特性更少的数据库。所以我更有希望理解它。这整个数据库都存储在一个文件中。

## Sqlite

sqlite 官网上有很多关于 [sqlite 内核](https://www.sqlite.org/arch.html)的文档，除此之外我还有一份资料 [SQLite Database System: Design and Implementation](https://play.google.com/store/books/details?id=9Z6IQQnX1JEC)

![](https://cstack.github.io/db_tutorial/assets/images/arch1.gif)
<center>sqlite architecture(https://www.sqlite.org/zipvfs/doc/trunk/www/howitworks.wiki)<center>

一个查询如果要获得数据或者修改数据的话，需要经过一系列的组件。前端包括：

* tokenizer
* parser
* code generator

前端的输入是一个 SQL 查询. 输出的是 sqlite 虚拟机字节码(本质上是一个可以在数据库上操作的编译程序)

后端包括：
* virtual machine
* B-tree
* pager
* os interface

virtual machine 虚拟机接收前端生成的字节码作为指令。这些指令可以操作一个或多个表、索引，表和索引都是以 B 树的数据结构存储的。虚拟机本质上是一个大的语句与字节码指令转换器。

每棵 B-tree 包含很多节点。每个节点的长度是一页。B 树可以从磁盘上读取一页的数据或者通过命令将数据写回到数据库。

pager 组件接收读取或者写入数据的指令。它需要在正确的数据库数据文件位置写入或者读取数据，也需要将最近访问到的数据页缓存到内存中，并且决定什么时候将这些数据缓存页写到磁盘上。

os interface 操作系统接口层取决于 sqlite 是在哪个操作系统层编译的。在这个课程中，我不会支持多个操作系统平台。

千里之行，始于足下，让我们从一些比较简单的内容开始：REPL（交互式顶层构件）。

## 实现一个简单的 REPL

当你从命令行启动 sqlite client 的时候，会启动一个循环读取命令并执行命令：

```c
~ sqlite3
SQLite version 3.16.0 2016-11-04 19:09:39
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> create table users (id int, username varchar(255), email varchar(255));
sqlite> .tables
users
sqlite> .exit
~
```

为了实现这个，我们的主函数会有一个无限输出提示符的循环，它会从读取一行输入，然后处理一行输入：

```c
int main(int argc, char* argv[]) {
  InputBuffer* input_buffer = new_input_buffer();
  while (true) {
    print_prompt();
    read_input(input_buffer);

    if (strcmp(input_buffer->buffer, ".exit") == 0) {
      close_input_buffer(input_buffer);
      exit(EXIT_SUCCESS);
    } else {
      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
    }
  }
}
```

我们需要定义一个 `InputBuffer` 结构体来存储 `getline()` 函数得到的内容及信息。

```c
typedef struct {
  char* buffer;
  size_t buffer_length;
  ssize_t input_length;
} InputBuffer;

InputBuffer* new_input_buffer() {
  InputBuffer* input_buffer = (InputBuffer*)malloc(sizeof(InputBuffer));
  input_buffer->buffer = NULL;
  input_buffer->buffer_length = 0;
  input_buffer->input_length = 0;

  return input_buffer;
}
```

`print_prompt()` 函数用于输出提示符。我们需要在读取输入之前需要打印提示符。

```c
void print_prompt() { printf("db > "); }
```

通过 `getline()` 函数读取一行输入

```c
ssize_t getline(char **lineptr, size_t *n, FILE *stream);
```

lineptr: 存储输入内容的缓冲区地址指针。如果被 getline() 函数 mallocated 了 NULL 值，那么用户需要手动释放该空间，即使命令执行失败了。

n：存储分配的缓冲区大小值的变量指针

stream: 读取输入流，我们会从标准输入中读取。

返回值：读取的字节数，有可能比缓冲区大小小。

我们通过 getline() 函数，将读取的输入存储在 `input_buffer->buffer` 中，分配的缓冲区大小存储在 `input_buffer->buffer_length` 中。并且将返回结构存储在 `input_buffer->input_length`。

初始环境下，缓冲区是空的，所以 getline 需要分配足够的内存用于存放输入并将指针指向缓冲区。

```c
void read_input(InputBuffer* input_buffer) {
  ssize_t bytes_read =
      getline(&(input_buffer->buffer), &(input_buffer->buffer_length), stdin);

  if (bytes_read <= 0) {
    printf("Error reading input\n");
    exit(EXIT_FAILURE);
  }

  // Ignore trailing newline
  input_buffer->input_length = bytes_read - 1;
  input_buffer->buffer[bytes_read - 1] = 0;
}
```

然后就该定义一个函数用来释放 `InputBuffer *` 实例的内存以及相应结构中的元素内存（getline 会在 read_input 的时候分配内存给 input_buffer->buffer）

```c
void close_input_buffer(InputBuffer* input_buffer) {
    free(input_buffer->buffer);
    free(input_buffer);
}
```

最后，我们需要解析并执行命令。目前这只能识别一个命令 `.exit`，用于退出程序。其他的输入命令我们会输出一个错误然后继续读取新的输入。

```c
if (strcmp(input_buffer->buffer, ".exit") == 0) {
  close_input_buffer(input_buffer);
  exit(EXIT_SUCCESS);
} else {
  printf("Unrecognized command '%s'.\n", input_buffer->buffer);
}
```

来尝试一下！

```sql
~ ./db
db > .tables
Unrecognized command '.tables'.
db > .exit
~
```

好了，我们现在有了一个可以使用的 REPL 程序。在下一部分中，我们会开始开发我们的命令语言。同时，在这给出这个部分的完整程序：

```c
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
  char* buffer;
  size_t buffer_length;
  ssize_t input_length;
} InputBuffer;

InputBuffer* new_input_buffer() {
  InputBuffer* input_buffer = malloc(sizeof(InputBuffer));
  input_buffer->buffer = NULL;
  input_buffer->buffer_length = 0;
  input_buffer->input_length = 0;

  return input_buffer;
}

void print_prompt() { printf("db > "); }

void read_input(InputBuffer* input_buffer) {
  ssize_t bytes_read =
      getline(&(input_buffer->buffer), &(input_buffer->buffer_length), stdin);

  if (bytes_read <= 0) {
    printf("Error reading input\n");
    exit(EXIT_FAILURE);
  }

  // Ignore trailing newline
  input_buffer->input_length = bytes_read - 1;
  input_buffer->buffer[bytes_read - 1] = 0;
}

void close_input_buffer(InputBuffer* input_buffer) {
    free(input_buffer->buffer);
    free(input_buffer);
}

int main(int argc, char* argv[]) {
  InputBuffer* input_buffer = new_input_buffer();
  while (true) {
    print_prompt();
    read_input(input_buffer);

    if (strcmp(input_buffer->buffer, ".exit") == 0) {
      close_input_buffer(input_buffer);
      exit(EXIT_SUCCESS);
    } else {
      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
    }
  }
}
```