# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是 NYU Application Security 课程作业 1 的代码仓库，涉及对一个存在多个安全漏洞的礼品卡读取程序进行审计、测试和修复。

## 常用命令

### 构建命令
```bash
# 构建所有版本（原版、ASAN、UBSAN）
make

# 清理构建文件
make clean

# 单独构建特定版本
make giftcardreader   # 构建原版
make asan            # 构建 AddressSanitizer 版本
make ubsan           # 构建 UndefinedBehaviorSanitizer 版本
```

### 测试命令
```bash
# 运行完整测试套件
make test
# 或直接运行测试脚本
./runtests.sh

# 运行单个礼品卡文件
./giftcardreader.original 1 <giftcard_file.gft>  # 使用原版
./giftcardreader.asan 1 <giftcard_file.gft>      # 使用 ASAN 版本
./giftcardreader.ubsan 1 <giftcard_file.gft>     # 使用 UBSAN 版本

# 运行单个测试用例（JSON 输出）
./giftcardreader.original 2 <giftcard_file.gft>
```

### 代码覆盖率测试
```bash
# 构建带覆盖率的版本
gcc --coverage -g -o giftcardreader giftcardreader.c

# 运行测试后生成覆盖率报告
lcov --capture --directory . --output-file coverage.info
lcov --remove coverage.info '/usr/*' --output-file coverage.info
genhtml coverage.info --output-directory coverage_report
```

### AFL++ 模糊测试
```bash
# 使用 AFL++ 编译器构建
afl-gcc -o giftcardreader_fuzz giftcardreader.c

# 运行模糊测试（主实例）
afl-fuzz -i input -o output -M fuzzer1 ./giftcardreader_fuzz 1 @@

# 运行并行模糊测试（从实例）
afl-fuzz -i input -o output -S fuzzer2 ./giftcardreader_fuzz 1 @@

# 在模糊测试结果上运行 ASAN
for f in output/queue/id*; do ./giftcardreader.asan 1 "$f"; done
```

## 核心架构

### 程序组成部分

1. **礼品卡数据结构** (giftcard.h)
   - `struct this_gift_card`: 顶层容器，包含数据大小和指向实际数据的指针
   - `struct gift_card_data`: 包含商户ID、客户ID和记录数组
   - `struct gift_card_record_data`: 多态记录结构，可以是金额变更、消息或动画程序
   - THX-1138 虚拟机：用于执行礼品卡动画程序的简单解释器

2. **主要功能模块**
   - `main()`: 解析命令行参数，读取文件，调用相应的处理函数
   - `print_gift_card_info()`: 文本格式输出礼品卡信息
   - `print_gift_card_json()`: JSON格式输出礼品卡信息
   - `animate()`: THX-1138 虚拟机实现，执行动画程序
   - `get_gift_card()`: 从二进制文件读取并解析礼品卡数据

### 关键安全问题区域

程序在多个地方存在内存安全和输入验证问题：

1. **内存管理缺陷**
   - 动态分配时缺少边界检查
   - 未验证的指针解引用
   - 潜在的缓冲区溢出

2. **输入验证不足**
   - 文件格式字段未验证（如记录数量、大小字段）
   - 类型字段未进行范围检查
   - 动画程序缺少指令边界检查

3. **THX-1138 虚拟机漏洞**
   - 程序计数器可能超出程序边界
   - 消息缓冲区指针可能越界
   - 潜在的无限循环风险

## 测试策略

### 测试用例组织
- `testcases/valid/`: 应该被正确解析的礼品卡文件
- `testcases/invalid/`: 应该被拒绝的恶意或无效礼品卡文件

### 关键测试场景
1. 边界条件测试（大数值、负数值）
2. 格式错误测试（无效类型、错误的大小字段）
3. 内存安全测试（缓冲区溢出、空指针解引用）
4. 控制流测试（无限循环、递归）

## GitHub Actions 配置

项目使用 GitHub Actions 进行持续集成：
- `.github/workflows/hello.yml`: 基础测试工作流
- `.github/workflows/test.yml`: 完整的构建和测试工作流

## 辅助工具

- `gengift.py`: 生成基本礼品卡文件
- `genanim.py`: 生成带动画程序的礼品卡文件
- `runtests.sh`: 自动化测试脚本，支持超时检测和彩色输出

## 开发建议

1. 修复漏洞时，优先考虑输入验证和边界检查
2. 使用 ASAN 和 UBSAN 版本进行测试，确保修复有效
3. 每个修复都应包含相应的回归测试用例
4. 保持向后兼容性，确保合法的礼品卡仍能正常读取