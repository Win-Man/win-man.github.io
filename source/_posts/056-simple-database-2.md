---
title: Let's Build a Simple Database - World's Simplest SQL Compiler and Virtual Machine
date: 2021-04-11 21:12:13
tags:
- 翻译
- Database
categories:
- 翻译
---

> 原文地址：https://cstack.github.io/db_tutorial/parts/part2.html
> 原文作者： [cstack](https://github.com/cstack)
> 译者：[Win-Man](https://github.com/Win-Man)

## 世界上最简单的 SQL 编译器和字节码虚拟机

我们正在写一个 sqlite 的克隆版本。sqlite 中 SQL 编译器是一个解析一个 string 输入并输出一个内部可识别的字节码。然后 sqlite 编译之后的字节码传给字节码虚拟机，在虚拟机中执行 SQL。

![](https://cstack.github.io/db_tutorial/assets/images/arch2.gif)
<center>SQLite Architecture (https://www.sqlite.org/arch.html)<center>

将这些内容拆成两个步骤有一些优势点：

* 减少每个部门的复杂度（例如：字节码虚拟机不用考虑语法错误）
* 对于相同的查询可以只需要编译一次，将编译后的字节码缓存住，以此可以提高性能

基于这个想法，我们重构一下我们的主函数来支持两个新的关键字：

```c
int main(int argc, char* argv[]) {
   InputBuffer* input_buffer = new_input_buffer();
   while (true) {
     print_prompt();
     read_input(input_buffer);

-    if (strcmp(input_buffer->buffer, ".exit") == 0) {
-      exit(EXIT_SUCCESS);
-    } else {
-      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
+    if (input_buffer->buffer[0] == '.') {
+      switch (do_meta_command(input_buffer)) {
+        case (META_COMMAND_SUCCESS):
+          continue;
+        case (META_COMMAND_UNRECOGNIZED_COMMAND):
+          printf("Unrecognized command '%s'\n", input_buffer->buffer);
+          continue;
+      }
     }
+
+    Statement statement;
+    switch (prepare_statement(input_buffer, &statement)) {
+      case (PREPARE_SUCCESS):
+        break;
+      case (PREPARE_UNRECOGNIZED_STATEMENT):
+        printf("Unrecognized keyword at start of '%s'.\n",
+               input_buffer->buffer);
+        continue;
+    }
+
+    execute_statement(&statement);
+    printf("Executed.\n");
   }
 }
```

像 `.exit` 这种非 SQL 的语句被称为“元指令”。这些元指令都是以 `.` 开始的，所以我们可以将这类语句单独在一个独立的函数中处理。

接下来，我们增加一个步骤用于将输入的行转换为数据库内部表达式。这就是我们对于 sqlite 前端的 hack 版本。

最后，我们将预处理好的语句传递给 `execute_statement`。这个函数后续会演进成我们的字节码虚拟机。

需要需要的是我们新增加了两个函数，并且这两个函数返回 enum 来表示成功或者失败。

```c
typedef enum {
  META_COMMAND_SUCCESS,
  META_COMMAND_UNRECOGNIZED_COMMAND
} MetaCommandResult;

typedef enum { PREPARE_SUCCESS, PREPARE_UNRECOGNIZED_STATEMENT } PrepareResult;
```

"Unrecognized statement" 语句表示这个语句像是一个表达式，但是表达式有错误，所以我们在任何情况都使用 enum 枚举结果代码。C 语言的编译器会报错如果我的 switch 语句不能处理枚举的项，所以我们可以确保处理了这个函数的每个结果。在将来可能会添加更多的结果代码。

`do_meta_command` 只是现有函数的一个封装，为了之后可处理更多的命令：

```c
MetaCommandResult do_meta_command(InputBuffer* input_buffer) {
  if (strcmp(input_buffer->buffer, ".exit") == 0) {
    exit(EXIT_SUCCESS);
  } else {
    return META_COMMAND_UNRECOGNIZED_COMMAND;
  }
}
```

目前我们的预处理语句只是包含了有两个可能结果的枚举数组。这可能会包含更多的数据如果我们允许在语句中传入参数:

```c
typedef enum { STATEMENT_INSERT, STATEMENT_SELECT } StatementType;

typedef struct {
  StatementType type;
} Statement;
```

`prepare_statement`（我们的 SQL 编译器）目前还无法识别 SQL 语句。事实上这个函数只能识别两个单词：

```
PrepareResult prepare_statement(InputBuffer* input_buffer,
                                Statement* statement) {
  if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
    statement->type = STATEMENT_INSERT;
    return PREPARE_SUCCESS;
  }
  if (strcmp(input_buffer->buffer, "select") == 0) {
    statement->type = STATEMENT_SELECT;
    return PREPARE_SUCCESS;
  }

  return PREPARE_UNRECOGNIZED_STATEMENT;
}
```

需要注意的是我们使用 strncmp 对比 insert 因为 insert 关键字之后会跟着插入的数据。（例如：insert 1 cstack foo@bar.com）

最后，`execute_statement` 包含了一些分支：

```c
void execute_statement(Statement* statement) {
  switch (statement->type) {
    case (STATEMENT_INSERT):
      printf("This is where we would do an insert.\n");
      break;
    case (STATEMENT_SELECT):
      printf("This is where we would do a select.\n");
      break;
  }
}
```

这边并没有返回任何错误码因为目前为止这边还不会发生任何错误。

经过这些重构之后，我们现在的 sqlite 可以认识两个新的关键字。

```sql
~ ./db
db > insert foo bar
This is where we would do an insert.
Executed.
db > delete foo
Unrecognized keyword at start of 'delete foo'.
db > select
This is where we would do a select.
Executed.
db > .tables
Unrecognized command '.tables'
db > .exit
~
```

我们数据库的基本框架基本已经形成了，如果不用存储数据的话，这是不是很好？在下一个部分，我们会去实现 insert 和 select 语句，创造世界上最差劲的存储引擎。同时在这给出这一差异部分的内容：

```c
@@ -10,6 +10,23 @@ struct InputBuffer_t {
 } InputBuffer;
 
+typedef enum {
+  META_COMMAND_SUCCESS,
+  META_COMMAND_UNRECOGNIZED_COMMAND
+} MetaCommandResult;
+
+typedef enum { PREPARE_SUCCESS, PREPARE_UNRECOGNIZED_STATEMENT } PrepareResult;
+
+typedef enum { STATEMENT_INSERT, STATEMENT_SELECT } StatementType;
+
+typedef struct {
+  StatementType type;
+} Statement;
+
 InputBuffer* new_input_buffer() {
   InputBuffer* input_buffer = malloc(sizeof(InputBuffer));
   input_buffer->buffer = NULL;
@@ -40,17 +57,67 @@ void close_input_buffer(InputBuffer* input_buffer) {
     free(input_buffer);
 }
 
+MetaCommandResult do_meta_command(InputBuffer* input_buffer) {
+  if (strcmp(input_buffer->buffer, ".exit") == 0) {
+    close_input_buffer(input_buffer);
+    exit(EXIT_SUCCESS);
+  } else {
+    return META_COMMAND_UNRECOGNIZED_COMMAND;
+  }
+}
+
+PrepareResult prepare_statement(InputBuffer* input_buffer,
+                                Statement* statement) {
+  if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
+    statement->type = STATEMENT_INSERT;
+    return PREPARE_SUCCESS;
+  }
+  if (strcmp(input_buffer->buffer, "select") == 0) {
+    statement->type = STATEMENT_SELECT;
+    return PREPARE_SUCCESS;
+  }
+
+  return PREPARE_UNRECOGNIZED_STATEMENT;
+}
+
+void execute_statement(Statement* statement) {
+  switch (statement->type) {
+    case (STATEMENT_INSERT):
+      printf("This is where we would do an insert.\n");
+      break;
+    case (STATEMENT_SELECT):
+      printf("This is where we would do a select.\n");
+      break;
+  }
+}
+
 int main(int argc, char* argv[]) {
   InputBuffer* input_buffer = new_input_buffer();
   while (true) {
     print_prompt();
     read_input(input_buffer);
 
-    if (strcmp(input_buffer->buffer, ".exit") == 0) {
-      close_input_buffer(input_buffer);
-      exit(EXIT_SUCCESS);
-    } else {
-      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
+    if (input_buffer->buffer[0] == '.') {
+      switch (do_meta_command(input_buffer)) {
+        case (META_COMMAND_SUCCESS):
+          continue;
+        case (META_COMMAND_UNRECOGNIZED_COMMAND):
+          printf("Unrecognized command '%s'\n", input_buffer->buffer);
+          continue;
+      }
     }
+
+    Statement statement;
+    switch (prepare_statement(input_buffer, &statement)) {
+      case (PREPARE_SUCCESS):
+        break;
+      case (PREPARE_UNRECOGNIZED_STATEMENT):
+        printf("Unrecognized keyword at start of '%s'.\n",
+               input_buffer->buffer);
+        continue;
+    }
+
+    execute_statement(&statement);
+    printf("Executed.\n");
   }
 }
```
