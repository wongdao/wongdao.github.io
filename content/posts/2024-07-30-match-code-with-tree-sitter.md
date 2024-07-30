---
title: 用tree-sitter实现对代码的结构化查询
date: 2024-07-30T11:00:00+08:00
slug: 2024-07-30-match-code-with-trees-itter
draft: false
categories:
  - Tech
tags:
  - tree-sitter
---

# Tree-sitter 是什么

第一次听说treesitter是在2020年左右，vim/emacs社区里讨论用它做语法高亮的帖子越来越多。真正引起我兴趣是2022年neovim的大动作：直接集成treesitter，这让我对这个小玩意产生了兴趣，并简单学习了一下。学完后发现它根本就不小，且功能非常强大。

简单来说 Tree-sitter 是一个语法分析/生成器工具，它和其它分析工具如yacc/lex/antlr最大的区别是：增量分析、快、稳。这些特性才令它被各种编辑器/IDE物色作为语法高亮以及其它高级功能的实现。

# Tree-sitter 适合干什么

实现语法高亮、lint、folding、自动格式化等轻量级语法/语义相关的操作。

# 对源码进行模式匹配

当然可以在 Tree-sitter 生成整个 CST 后遍历所有节点来实现模式匹配，然而还有一个更方便的方法：使用 Tree-sitter query。该 query 本质就是 S-expression，因此使用起来比较简单，只需3步：

1. 通过 Treesitter [playground](https://tree-sitter.github.io/tree-sitter/playground) 获取源码的 CST 表示；
2. 根据 CST 编写模式匹配 s-exp
3. 自行编写工具或直接用 tree-sitter 执行查询

下面将使用一个例子来具体说明。

# 例子：使用rust+treesitter列出指定类型所有被调用的方法名称

这里将写一个简单的工具，能列出指定类所有被调用的方法的名称。

```java
# test.java
class Foo {
  
  public void bar() {
    Integer target1 = 42;
    String notTarget = 3;
    Integer target2 = 237.0f;

    target1.toString();
    notTarget.toString();
    target2.clone();
  }
  
}
```

工具运行结果如下：

```
$ chk-method-be-called Integer test.java
local var "target1" invoke method "toString" in the line: 8
local var "target2" invoke method "clone" in the line: 10

$ chk-method-be-called String test.java
local var "notTarget" invoke method "toString", line: 9
```

## 第一步：获取 CST
进入 [playground](https://tree-sitter.github.io/tree-sitter/playground) ，将 test.java 粘贴到 `Code`，然后得到如下 CST：

```
program [0, 0] - [13, 0]
  class_declaration [0, 0] - [12, 1]
    name: identifier [0, 6] - [0, 9]
    body: class_body [0, 10] - [12, 1]
      method_declaration [2, 2] - [10, 3]
        modifiers [2, 2] - [2, 8]
        type: void_type [2, 9] - [2, 13]
        name: identifier [2, 14] - [2, 17]
        parameters: formal_parameters [2, 17] - [2, 19]
        body: block [2, 20] - [10, 3]
          local_variable_declaration [3, 4] - [3, 25]
            type: type_identifier [3, 4] - [3, 11]
            declarator: variable_declarator [3, 12] - [3, 24]
              name: identifier [3, 12] - [3, 19]
              value: decimal_integer_literal [3, 22] - [3, 24]
          local_variable_declaration [4, 4] - [4, 25]
            type: type_identifier [4, 4] - [4, 10]
            declarator: variable_declarator [4, 11] - [4, 24]
              name: identifier [4, 11] - [4, 20]
              value: decimal_integer_literal [4, 23] - [4, 24]
          local_variable_declaration [5, 4] - [5, 29]
            type: type_identifier [5, 4] - [5, 11]
            declarator: variable_declarator [5, 12] - [5, 28]
              name: identifier [5, 12] - [5, 19]
              value: decimal_floating_point_literal [5, 22] - [5, 28]
          expression_statement [7, 4] - [7, 23]
            method_invocation [7, 4] - [7, 22]
              object: identifier [7, 4] - [7, 11]
              name: identifier [7, 12] - [7, 20]
              arguments: argument_list [7, 20] - [7, 22]
          expression_statement [8, 4] - [8, 25]
            method_invocation [8, 4] - [8, 24]
              object: identifier [8, 4] - [8, 13]
              name: identifier [8, 14] - [8, 22]
              arguments: argument_list [8, 22] - [8, 24]
          expression_statement [9, 4] - [9, 20]
            method_invocation [9, 4] - [9, 19]
              object: identifier [9, 4] - [9, 11]
              name: identifier [9, 12] - [9, 17]
              arguments: argument_list [9, 17] - [9, 19]
```

## 第二步：将 CST 修改成为 s-exp

编写 query s-exp，这里用到了两个特性：capture 和 predicates。更多参考官网的章节：[pattern-matching-with-queries](pattern-matching-with-queries) 。
- capture: `@the_type`, `@name`, `@calling`, `@method`
- predicates: `#eq?`

s-exp:
```
(method_declaration
  (block
    (local_variable_declaration
      type: (type_identifier) @the_type (#eq? @the_type {target_class})
      declarator: (variable_declarator
        name: (identifier) @var_name
      )
    )
    (expression_statement
      (method_invocation
        object: (identifier) @invokder
          (#eq? @invokder @var_name)
        name: (identifier) @method
      )
    )
  )
)
```

## 第三步：通过rust编写工具

选用rust主要因为其包管理工具非常方便。首先创建工程，并添加依赖：

```
# 创建工程
cargo new --bin chk-method-be-called

# 添加依赖
cargo add tree-sitter
cargo add --git https://github.com/tree-sitter/tree-sitter-java
```

然后编写代码 main.rs：

```rust
use std::env;
use tree_sitter::Language;
use tree_sitter::Parser;
use tree_sitter::Query;
use tree_sitter::QueryCursor;

fn main() {
    // load list
    let target_class = env::args().nth(1).unwrap();

    // load source file
    let pathname = env::args().nth(2).unwrap();
    let contents = std::fs::read_to_string(&pathname).unwrap();

    // create parser
    let java_lang: Language = tree_sitter_java::language();
    let mut parser = Parser::new();
    parser.set_language(&java_lang).unwrap();

    // create CST and query
    let parse_tree = parser.parse(&contents, None).unwrap();
    let query = gen_query(&java_lang, &target_class);
    let mut query_cursor = QueryCursor::new();
    let all_matches = query_cursor.matches(&query, parse_tree.root_node(), contents.as_bytes());

    // capture node
    let the_type = query.capture_index_for_name("the_type").unwrap();
    let var_name = query.capture_index_for_name("var_name").unwrap();
    let invoker = query.capture_index_for_name("invokder").unwrap();
    let method = query.capture_index_for_name("method").unwrap();

    // matching the pattern
    for each_match in all_matches {
        for capture in each_match.captures.iter().filter(|c| {
            c.index == the_type || c.index == var_name || c.index == invoker || c.index == method
        }) {
            let range = capture.node.range();
            let text = &contents[range.start_byte..range.end_byte];
            let line = range.start_point.row;
            let _col = range.start_point.column;
            if capture.index == var_name {
                print!("local var \"{}\"", text);
            } else if capture.index == method {
                print!(" invoke method \"{}\" in the line: {}\n", text, line + 1);
            }
        }
    }
}

fn gen_query(lang: &Language, target_class: &String) -> Query {
    Query::new(
        lang,
        format!(
            r#"
(method_declaration
  (block
    (local_variable_declaration
      type: (type_identifier) @the_type (#eq? @the_type {target_class})
      declarator: (variable_declarator
        name: (identifier) @var_name
      )
    )
    (expression_statement
      (method_invocation
        object: (identifier) @invokder
          (#eq? @invokder @var_name)
        name: (identifier) @method
      )
    )
  )
)
            "#
        )
        .as_str(),
    )
    .unwrap()
}

```

运行后得到预期结果：
```
chk-method-be-called$ cargo run Integer test.java
local var "target1" invoke method "toString" in the line: 8
local var "target2" invoke method "clone" in the line: 10

chk-method-be-called$ cargo run String test.java
local var "notTarget" invoke method "toString" in the line: 9
```

