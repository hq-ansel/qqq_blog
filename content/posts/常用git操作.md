---
date: 2024-12-10T04:09:30Z
tags:
  - "#开发八股"
  - 开发八股
title: 常用git操作
share: true
canonicalURL: ""
keywords:
  - git
description: 
series: 系列
lastmod: 
lang: cn
cover:
  image: 
author: qqq
dir: posts
---

### 丢弃更改
#### 1. **丢弃工作区的未暂存更改**
   如果你在工作区修改了文件但尚未暂存（使用 `git add`），可以使用以下命令将文件恢复到上一次提交时的状态：
   ```bash
   git checkout -- <file>
   ```
   这个命令会丢弃指定文件的更改，恢复到上次提交的版本。

   如果想丢弃所有文件的更改，可以使用：
   ```bash
   git checkout -- .
   ```

#### 2. **丢弃已暂存的更改**
   如果你已经使用 `git add` 将更改暂存，但还没有提交，可以使用以下命令将更改从暂存区移除，但保留在工作区中：
   ```bash
   git reset HEAD <file>
   ```
   这样做会将文件从暂存区移除，但工作区中的更改仍然存在。

   如果想将所有已暂存的文件移除暂存区，可以使用：
   ```bash
   git reset HEAD .
   ```

#### 3. **丢弃所有未提交的更改（包括工作区和暂存区）**
   如果你想要彻底丢弃所有未提交的更改（包括工作区和暂存区的更改），可以使用以下命令：
   ```bash
   git reset --hard
   ```
   这个命令会将你的工作区和暂存区重置到上一次提交的状态，所有未提交的更改都会丢失。

#### 4. **丢弃已经提交的更改**
   如果你已经提交了更改，但想要撤销提交，可以使用以下命令：

   - **回滚到之前的提交但保留更改在工作区：**
     ```bash
     git reset --soft <commit-hash>
     ```
     这将把当前分支重置到指定的提交，但保留更改在暂存区。

   - **彻底丢弃提交后的更改：**
     ```bash
     git reset --hard <commit-hash>
     ```
     这个命令会将当前分支重置到指定的提交，工作区和暂存区的更改都会被丢弃。

#### 5. **丢弃特定提交的更改**
   如果你想撤销某个特定的提交，可以使用 `git revert`：
   ```bash
   git revert <commit-hash>
   ```
   这个命令会生成一个新的提交来撤销指定的更改，而不会影响其他历史提交。

使用这些命令时需要谨慎，特别是 `git reset --hard`，因为这会丢失所有未提交的更改，无法恢复。

#### 6.代码库行数提交统计
方法 1：使用 git shortlog 查看提交数
运行以下命令可以按作者统计提交记录：
```bash
git shortlog -s -n
```

- s：只显示提交的数量。
- n：按提交数量排序。
输出示例：

```
50  Alice
30  Bob
20  Charlie

```