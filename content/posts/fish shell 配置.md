---
date: 2024-12-11T19:12:38+08:00
tags:
  - linux
  - shell
title: fish shell 配置
share: true
canonicalURL: ""
keywords: 
description: 
series: 系列
lastmod: 
lang: cn
cover:
  image: 
author: heqi
dir: posts
---



### **1. 安装 Fish Shell**

#### **步骤**

1. 更新系统：
    
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```
    
2. 安装 Fish：
    
    ```bash
    sudo apt install fish -y
    ```
    
3. 验证安装：
    
    ```bash
    fish --version
    ```
    

---

### **2. 设置 Fish 为默认 Shell**

1. 确认 Fish 的路径：
    
    ```bash
    which fish
    ```
    
    通常路径为 `/usr/bin/fish`。
2. 将 Fish 设置为默认 Shell：
    
    ```bash
    chsh -s /usr/bin/fish
    ```
    
3. 重新启动终端以应用更改。

---

### **3. 安装 Fisher (Fish 的插件管理器)**

Fisher 是一个流行的插件管理器，用于轻松安装和管理 Fish 插件。

1. 安装 Fisher：
    
    ```bash
    curl -sL https://git.io/fisher | source && fisher install jorgebucaran/fisher
    ```
    
2. 验证安装：
    
    ```bash
    fisher --version
    ```
    

---

### **4. 安装常用插件**

推荐以下常用插件：

1. **插件安装命令：**
    
    ```bash
    fisher install <插件名>
    ```
    
2. **推荐插件列表：**
    - `oh-my-fish/theme-bobthefish`: 美观的主题。
        
        ```bash
        fisher install oh-my-fish/theme-bobthefish
        ```
        
    - `jorgebucaran/nvm.fish`: Node.js 版本管理。
        
        ```bash
        fisher install jorgebucaran/nvm.fish
        ```
        
    - `PatrickF1/fzf.fish`: 模糊查找。
        
        ```bash
        fisher install PatrickF1/fzf.fish
        ```
        
    - `jethrokuan/z`: 快速目录跳转。
        
        ```bash
        fisher install jethrokuan/z
        ```
        
    - `franciscolourenco/done`: 长时间命令完成时提醒。
        
        ```bash
        fisher install franciscolourenco/done
        ```
        

---

### **5. 配置 Fish Shell**

Fish 使用 `$HOME/.config/fish/config.fish` 文件来存储配置。可以编辑该文件以自定义你的 Shell。

1. 打开配置文件：
    
    ```bash
    nano ~/.config/fish/config.fish
    ```
    
2. 添加一些常用配置：
    
    ```fish
    # 启用插件
    fisher install oh-my-fish/theme-bobthefish
    
    # 设置别名
    alias ll="ls -lh"
    alias gs="git status"
    
    # 设置 PATH
    set -x PATH $PATH /usr/local/bin
    
    # 自动跳转插件
    jethrokuan/z
    
    # 启用语法高亮
    fisher install ilancosman/tide
    ```
    

---

### **6. 美化 Shell**

1. **安装 Nerd Fonts**: Nerd Fonts 提供丰富的图标支持。可以从 [Nerd Fonts 官网](https://www.nerdfonts.com/) 下载适合你的字体，并安装到系统中。
    
2. **更改终端字体**：
    
    - 打开终端设置。
    - 在外观设置中选择刚刚安装的 Nerd Font 字体。
3. **启用主题**：
    
    - 使用 `bobthefish` 或者 `tide` 主题进行美化：
        
        ```bash
        fisher install ilancosman/tide
        ```
        
    - 运行 `tide configure`，根据提示完成配置。

---

### **7. 启用自动补全和语法高亮**

1. Fish 自带强大的自动补全功能，可以直接使用。
2. 如果需要更高级的语法高亮，可以使用插件 `tide`。

---

### **8. 安装常用工具**

1. **fzf (模糊查找工具)**:
    
    ```bash
    sudo apt install fzf -y
    ```
    
2. **ripgrep (快速搜索工具)**:
    
    ```bash
    sudo apt install ripgrep -y
    ```
    

---

### **9. 备份和恢复配置**

#### **备份**

将 `~/.config/fish` 目录打包保存：

```bash
tar -czvf fish-config-backup.tar.gz ~/.config/fish
```

#### **恢复**

将备份的文件解压到对应目录：

```bash
tar -xzvf fish-config-backup.tar.gz -C ~/
```

---

### **10. 测试配置**

1. 打开一个新的终端，确认配置生效。
2. 运行以下命令，验证所有功能：
    
    ```bash
    fish
    fisher list
    echo $PATH
    ```
    
