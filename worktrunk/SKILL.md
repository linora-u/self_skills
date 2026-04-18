---
name: worktrunk
description: Git工作区扩展工具 Worktrunk (wt) 的配置说明与中文路径(非ASCII)问题的排错指南
---

# Worktrunk (wt) 工具指南与排错

本 Skill 记录了使用 Worktrunk (`wt`) 进行并行工作区（Git Worktree）管理时的重要配置机制，以及在使用其自动化拷贝缓存数据的钩子（Hooks）时常遇到的中文 / 非 ASCII 路径故障与解决方案。

## 什么是 Worktrunk (wt)？
`wt` 是一个针对并行 AI 代理和多分支开发流程设计的 Git Worktree 管理和增强工具。其核心优势在于提供高效的分支创建与切换体验，并能在新工作区生成时使用自动化 Hooks（例如通过 `wt step copy-ignored` 快速复用缓存/大量数据文件，避免冷启动）。

## 安装与初始化

Worktrunk 主要通过 Rust 包管理器 (Cargo) 进行编译安装，也支持部分系统级包管理器：

1. **通过 Cargo 安装 (推荐):**
   ```bash
   cargo install worktrunk
   ```

2. **通过 Arch Linux 包管理器:**
   ```bash
   sudo pacman -S worktrunk
   ```

3. **初始化 Shell 集成:**
   安装完成后，为了能够在使用 `wt switch` 时动态改变当前终端的所在目录，强烈建议安装 Shell 拦截集成：
   ```bash
   wt config shell install
   ```

## 核心配置

`wt` 支持通过 `~/.config/worktrunk/config.toml` 配置项目钩子和全局行为。
如果要确保在每次切换（或建立）新工作树时，自动继承所有 `gitignored` 的关键本地文件（例如虚拟环境、编译缓存、运行时数据目录）。你需要将配置文件编写如下：

```toml
[post-start]
copy = "wt step copy-ignored"
```
*(注: `[post-start]` 也可以是 `[post-create]` 或其他受支持的生命周期。目前 `post-start` 已具备足够的兼容性，在第一次成功切换或创建后会自动触发)。*

## 典型故障排查：`copy-ignored` 不生效 (数据目录丢失)

### 问题描述
你的项目中使用了 `[post-start]` 绑定了拷贝指令，并在本地拥有大量无需进版（`.gitignore`）的运行时资源（例如项目中的 `./data/` 目录）。
当我们执行 `wt switch -c new_branch` 创建新的工作树后，却发现某些 `ignored` 目录（例如 `data/`）并没有被拷贝过来。如果你单独手动运行 `wt step copy-ignored`，可能会立刻自动退出，并在底层报错：`No such file or directory (os error 2)`。

### 根本原因
这个问题通常是由于被忽略（`.gitignore` 的）的文件或文件夹中包含了**中文名或其他非 ASCII 字符**。
底层机制中，`wt step copy-ignored` 主要是透过执行 `git ls-files -o` (或其他类似方法) 来分析尚未追踪及被忽略的文件。默认情况下，**Git 会将所有中文/非 ASCII 字符转化成八进制编码，并在路径两侧包裹双引号**（例如: `"data/kpl\346\266\250..."`）。
这个双引号转义扰乱了 `wt` 底层的路径解析器逻辑，导致它直接尝试按带有双引号的字面字符串去读取系统目录，从而引发报错中断，背后的所有正常复制指令也就全部被提前停止了。

### 完美解决方案
可以通过更改 Git 选项，关闭这种中文字符强制转义的行为。

1. **设置（只需要执行一次的）全局 Git Config:**
   在终端执行任意一条指令关闭引号包围和八进制转义：
   ```bash
   git config --global core.quotePath false
   ```

2. **验证配置并处理存量环境:**
   此时再通过 `wt switch -c <name>` 创建工作树，`[post-start]` 的宏即可完全正常地识别中文目录，并将所有复杂的 `.gitignore` 数据原样搬运。
   如果你已经在问题工作树内，需要补救，直接在新终端内键入一次单步命令即可修复当前工作区的数据缺失问题：
   ```bash
   wt step copy-ignored
   ```
