---
title: Rust 文件路径
date: 2024-08-07 16:46:00 -0400
tags:
  - rust
  - filepath
description: 本文生成自 ChatGPT Rust 模型（创建者 Widenex），并作了部分修改
---

`std::path` 模块在 Rust 中用于处理文件系统路径。该模块提供了两个主要的类型：`Path` 和 `PathBuf`，分别用于 **不可变** 和 **可变** 路径。下面是对 `std::path` 模块的入门介绍，包括基本用法和常见操作示例。

## 基本类型

`Path`：表示文件系统路径的不可变视图。`Path` 类型是一个引用类型，类似于 `&str`。
`PathBuf`：表示文件系统路径的可变、拥有所有权的字符串。类似于 `String`。

## 创建路径

你可以使用 `Path` 和 `PathBuf` 来创建和操作路径。通常，`Path` 用于 **只读操作**，而 `PathBuf` 用于 **需要修改路径的操作**。

```rust
use std::path::{Path, PathBuf};

fn main() {
    // 使用字符串字面量创建一个不可变路径
    let path = Path::new("/some/path/to/file.txt");

    // 使用 PathBuf 创建一个可变路径
    let mut path_buf = PathBuf::from("/some/path/to");
    path_buf.push("file.txt");

    println!("Path: {:?}", path);
    println!("PathBuf: {:?}", path_buf);
}
```

```terminal
Path: "/some/path/to/file.txt"
PathBuf: "/some/path/to\\file.txt"
```

## 基本操作

### 拼接路径

你可以使用 `push` 方法在 `PathBuf` 上拼接路径。

```rust
use std::path::PathBuf;

fn main() {
    let mut path = PathBuf::from("/some/path");
    path.push("to");
    path.push("file.txt");

    println!("Path: {:?}", path);
}
```

```terminal
Path: "/some/path\\to\\file.txt"
```

> [!tip] 提示
> `push` 在原来的基础上添加路径，`join` 则是返回新的路径。

### 获取路径的各个部分

你可以使用 `Path` 提供的各种方法来获取路径的不同部分，例如文件名、父目录、扩展名等。

```rust
use std::path::Path;

fn main() {
    let path = Path::new("/some/path/to/file.txt");

    println!("File name: {:?}", path.file_name());
    println!("Extension: {:?}", path.extension());
    println!("Parent: {:?}", path.parent());
}
```

```terminal
File name: Some("file.txt")
Extension: Some("txt")
Parent: Some("/some/path/to")
```

### 遍历路径组件

你可以遍历路径的各个组件，例如目录、文件名等。

```rust
use std::path::Path;

fn main() {
    let path = Path::new("/some/path/to/file.txt");

    for component in path.components() {
        println!("{:?}", component);
    }
}
```

```terminal
RootDir
Normal("some")
Normal("path")
Normal("to")
Normal("file.txt")
```

### 转换为字符串

你可以将路径转换为字符串或字符串切片。

```rust
use std::path::Path;

fn main() {
    let path = Path::new("/some/path/to/file.txt");

    // 转换为 &str
    if let Some(path_str) = path.to_str() {
        println!("Path as &str: {}", path_str);
    }

    // 转换为 String
    let path_string = path.to_string_lossy();
    println!("Path as String: {}", path_string);
}
```

```terminal
Path as &str: /some/path/to/file.txt
Path as String: /some/path/to/file.txt
```

### 检查路径属性

你可以检查路径的属性，例如是否是绝对路径，是否是文件或目录等。

```rust
use std::path::Path;

fn main() {
    let path = Path::new("/some/path/to/file.txt");

    println!("Is absolute: {}", path.is_absolute());
    println!("Is relative: {}", path.is_relative());
}
```

```terminal
Is absolute: false
Is relative: true
```

## 示例：合并路径和扩展名

以下是一个示例，展示了如何使用 Path 和 PathBuf 合并路径并修改扩展名。

```rust
use std::path::{Path, PathBuf};

fn main() {
    let path = Path::new("/some/path/to/file");

    // 创建一个新的路径，添加扩展名
    let mut new_path = PathBuf::from(path);
    new_path.set_extension("txt");

    println!("Original path: {:?}", path);
    println!("New path: {:?}", new_path);
}
```

```terminal
Original path: "/some/path/to/file"
New path: "/some/path/to/file.txt"
```

## 总结

`std::path` 模块提供了丰富的功能来处理文件系统路径，无论是简单的路径拼接还是复杂的路径操作都能轻松实现。通过 `Path` 和 `PathBuf` 类型，你可以编写健壮且易于维护的路径操作代码。💡
