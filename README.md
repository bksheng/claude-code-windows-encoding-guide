# Claude Code Windows 中文编码问题完全解决方案

> **项目定位**：解决 Claude Code 在 Windows 环境下处理中文文件时的系统性编码问题
> 
> **适用场景**：Windows 10/11 中文版、Git Bash、CMD、PowerShell
> 
> **问题严重性**：Anthropic 官方标记为 "Not planned"（不计划修复）

---

## 📋 目录

- [问题概述](#问题概述)
- [错误现象](#错误现象)
- [根本原因分析](#根本原因分析)
- [解决方案汇总](#解决方案汇总)
- [最佳实践指南](#最佳实践指南)
- [实际案例](#实际案例)
- [工具对比](#工具对比)
- [社区资源](#社区资源)
- [CLAUDE.md 配置模板](#claudemd-配置模板)
- [FAQ](#faq)

---

## 问题概述

Claude Code 在 Windows 中文环境下存在**系统性编码问题**，主要影响：

1. **Read 工具**：读取 UTF-8 中文文件时中断
2. **Bash 工具**：执行含中文输出命令时 GBK 解码失败
3. **Write/Edit 工具**：保存文件时 BOM 丢失或编码转换

### 影响范围

| 工具 | 影响程度 | 典型错误 |
|------|----------|----------|
| Read | 🔴 高 | `[Tool use interrupted]` |
| Bash | 🔴 高 | `UnicodeDecodeError: 'gbk' codec can't decode byte 0xae` |
| Write | 🟡 中 | UTF-8 BOM 丢失 |
| Edit | 🟡 中 | 文件编码被强制转为 UTF-8 |

---

## 错误现象

### 错误 1：Read 工具中断

```
● Read(C:\project\file.txt)
  ⎿  [Tool use interrupted]

✻ Baked for 1m 50s
```

**特征**：
- 读取含有中文的 UTF-8 文件时发生
- 无明确错误信息，直接中断
- 超时后自动恢复

### 错误 2：Bash 工具 GBK 解码错误

```
Exception in thread Thread-5 (_readerthread):
Traceback (most recent call last):
  File "C:\Users\sheng\miniconda3\Lib\threading.py", line 1073...
  File "C:\Users\sheng\miniconda3\Lib\subprocess.py", line 1599...
    buffer.append(fh.read())
UnicodeDecodeError: 'gbk' codec can't decode byte 0xae in position 1033: 
  illegal multibyte sequence
Unexpected cygpath error, fallback to manual path conversion
  AttributeError: 'NoneType' object has no attribute 'strip'
```

**特征**：
- 执行 Bash 命令时，子进程输出包含 UTF-8 字符
- Python subprocess 模块用 GBK 解码失败
- 伴随 `cygpath` 路径转换错误

### 错误 3：文件编码损坏

```
# 原始内容（UTF-8）
# 推送渠道配置管理系统

# 被 Claude Code 修改后（GBK 解释）
# ùϵͳ
```

**特征**：
- 文件被强制转为 UTF-8（无 BOM）
- 原有 BOM 被移除
- 非 UTF-8 文件（如 GBK）内容损坏

---

## 根本原因分析

### Windows 编码机制

| 层级 | 默认编码 | 说明 |
|------|----------|------|
| 操作系统（中文版） | GBK (cp936) | Windows 系统默认编码 |
| Git Bash / MinGW | 继承系统编码 | 使用 GBK 作为默认编码 |
| Python subprocess | 继承系统编码 | 默认使用 `locale.getpreferredencoding()` |
| Claude Code 工具 | 强制 UTF-8 | 读写文件时强制使用 UTF-8 |

### 错误触发链

```
用户执行命令
    ↓
Claude Code 调用 Bash 工具
    ↓
Python subprocess 创建子进程
    ↓
子进程输出包含 UTF-8 字节（如中文、特殊符号 ° 等）
    ↓
Python 尝试用 GBK 解码输出
    ↓
GBK 无法解码 UTF-8 字节（如 0xae）
    ↓
UnicodeDecodeError: 'gbk' codec can't decode byte 0xae
```

### 为什么 `0xae` 会触发错误？

- `0xae` 是 UTF-8 编码的某个字节
- GBK 编码中，`0xae` 不是有效的单字节字符开头
- GBK 双字节字符范围：`0x81-0xFE` + `0x40-0xFE`
- `0xae` 单独出现不符合 GBK 规则，导致解码失败

### pip `auto_decode` 源码分析

```python
# pip/_internal/utils/encoding.py
BOMS = [
    (codecs.BOM_UTF8, "utf-8"),        # \xef\xbb\xbf
    (codecs.BOM_UTF16, "utf-16"),      # \xff\xfe
    ...
]

def auto_decode(data: bytes) -> str:
    # 优先级 1：BOM 头检测（最可靠）
    for bom, encoding in BOMS:
        if data.startswith(bom):
            return data[len(bom):].decode(encoding)
    
    # 优先级 2：文件头编码声明（如 # coding: utf-8）
    for line in data.split(b"\n")[:2]:
        if line[0:1] == b"#" and ENCODING_RE.search(line):
            ...
    
    # 优先级 3：系统 locale 编码（Windows 默认 GBK，不可靠）
    return data.decode(locale.getpreferredencoding(False))
```

**关键发现**：`PYTHONUTF8=1` 环境变量**不影响 pip 的文件读取逻辑**！pip 使用的是 `locale.getpreferredencoding()`，而非 Python 的 UTF-8 模式。

---

## 解决方案汇总

### 方案 1：使用 Python 代替 Bash（推荐）

**原理**：Python 可显式指定 `encoding='utf-8'`，绕过系统默认编码。

#### 读取文件

```bash
# ❌ 避免：直接 cat 中文文件（可能导致 Read 工具中断）
cat start.bat.zh

# ✅ 推荐：用 Python 读取
python -c "with open('start.bat.zh', 'r', encoding='utf-8') as f: print(f.read())"
```

#### 写入文件

```bash
# ❌ 避免：echo 中文 > file.txt
echo "中文内容" > file.txt

# ✅ 推荐：用 Python 写入
python -c "with open('file.txt', 'w', encoding='utf-8') as f: f.write('中文内容')"
```

#### 多行 Python 代码（使用 Here Document）

```bash
# ❌ 避免：双引号包裹多行（Bash 会解析特殊字符）
python -c "
with open('file.txt', 'rb') as f:
    data = f.read()
"

# ✅ 推荐：使用 Here Document
python << 'EOF'
with open('file.txt', 'rb') as f:
    data = f.read()
    print('First 20 bytes:', data[:20])
    if data[:3] == b'\xef\xbb\xbf':
        print('Has UTF-8 BOM')
    else:
        print('No BOM')
EOF
```

### 方案 2：添加 UTF-8 BOM 头

当文件需要被其他工具（如 pip）读取时，添加 UTF-8 BOM 头：

```bash
python << 'EOF'
with open('requirements.txt', 'rb') as f:
    data = f.read()

with open('requirements.txt', 'wb') as f:
    f.write(b'\xef\xbb\xbf' + data)

print('UTF-8 BOM added')
EOF
```

**原理**：pip 的 `auto_decode()` 优先检测 BOM 头，能正确识别 UTF-8 编码。

### 方案 3：设置 `PYTHONUTF8=1`

在启动脚本中设置环境变量：

```batch
@echo off
set PYTHONUTF8=1
```

**效果**：
- ✅ 解决 Python 标准输出编码问题
- ❌ 不解决 pip 文件读取问题
- ❌ 不解决 Bash 工具编码问题

### 方案 4：使用 PowerShell

Windows 下 PowerShell 对 UTF-8 支持更好：

```powershell
# 读取文件
Get-Content start.bat.zh -Encoding UTF8

# 写入文件
Set-Content file.txt -Value "中文" -Encoding UTF8
```

### 方案 5：社区工具 `claude_encoding_guard`

GitHub: [ymonster/claude_encoding_guard](https://github.com/ymonster/claude_encoding_guard)

拦截 Claude Code 的文件操作，保留原始编码：

```
PreToolUse (Read)                       PostToolUse (Edit/Write)
    │                                        │
    ├─ binary check                        ├─ read cached encoding + line ending
    ├─ detect encoding (chardet 5.x)       ├─ convert UTF-8 → original encoding
    ├─ detect line ending (CRLF/LF)        ├─ normalize line endings to original
    ├─ convert original → UTF-8           ├─ delete session cache
    ├─ save session cache                  └─ done
    └─ Claude reads correct content
```

**支持编码**：GBK, GB2312, GB18030, Big5, Shift_JIS, EUC-JP, EUC-KR, Windows-1252 等

### 方案 6：系统级 UTF-8 配置

使用 [Code-encoding-fix](https://github.com/hellowind777/Code-encoding-fix) GUI 工具配置所有 Windows 终端：

- PowerShell
- Git Bash
- VS Code
- Windows Console

或手动设置注册表：

```powershell
# 启用 Windows Beta UTF-8 支持
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Nls\CodePage" -Name "ACP" -Value "65001"
```

---

## 最佳实践指南

### 处理规则（强制遵循）

| 优先级 | 场景 | 推荐方案 | 示例 |
|--------|------|----------|------|
| P0 | 读取中文文件 | Python + encoding='utf-8' | `python -c "with open('f.txt', 'r', encoding='utf-8') as f: print(f.read())"` |
| P0 | 写入中文文件 | Python + encoding='utf-8' | `python -c "with open('f.txt', 'w', encoding='utf-8') as f: f.write('中文')"` |
| P1 | 文件列表 | Python os.listdir() | `python -c "import os; print(os.listdir('.'))"` |
| P2 | 简单命令（无中文） | Bash | `python --version` |
| P2 | Windows 特定操作 | PowerShell | `Get-Content file -Encoding UTF8` |

### 禁止事项

1. ❌ **严禁**在 Bash 中直接 `cat` 含有中文的文件（可能导致 Read 工具中断）
2. ❌ **严禁**在 Bash 中 `echo 中文 > 文件`
3. ❌ **严禁**依赖系统默认编码处理文本文件
4. ❌ **严禁**在输出中混用 GBK 和 UTF-8 编码
5. ❌ 文件路径避免使用中文和特殊字符（如 `°`）

### 应急处理流程

当 Read/Write/Bash 工具因编码问题中断时：

1. **立即改用 Python 脚本**
2. **检测文件编码**：
   ```bash
   python << 'EOF'
   with open('file.txt', 'rb') as f:
       data = f.read()
       print('First 20 bytes:', data[:20])
       if data[:3] == b'\xef\xbb\xbf':
           print('Has UTF-8 BOM')
       else:
           print('No BOM')
   EOF
   ```
3. **如需添加 UTF-8 BOM**：
   ```bash
   python << 'EOF'
   with open('file.txt', 'rb') as f:
       data = f.read()
   with open('file.txt', 'wb') as f:
       f.write(b'\xef\xbb\xbf' + data)
   print('UTF-8 BOM added')
   EOF
   ```

---

## 实际案例

### 案例 1：读取中文批处理文件

```bash
# ❌ 错误：Read 工具会中断
Read start.bat.zh

# ✅ 正确：Python 读取
python -c "with open('start.bat.zh', 'r', encoding='utf-8') as f: print(f.read())"
```

### 案例 2：创建中文注释的 requirements.txt

```bash
# ❌ 错误：直接写入会编码错误

# ✅ 正确：Python 写入（单行）
python -c "content = '# 中文注释\nfastapi==0.109.0\n'; open('requirements.txt', 'w', encoding='utf-8').write(content)"

# ✅ 正确：Python 写入（多行，使用 Here Document）
python << 'EOF'
content = '# 中文注释\nfastapi==0.109.0\n'
with open('requirements.txt', 'w', encoding='utf-8') as f:
    f.write(content)
EOF
```

### 案例 3：检查文件是否存在中文乱码

```bash
# ✅ 正确：Python 检查
python << 'EOF'
with open('start.bat', 'rb') as f:
    data = f.read()
    try:
        text = data.decode('utf-8')
        print('UTF-8 OK')
    except:
        print('Not UTF-8')
EOF
```

### 案例 4：pip 安装中文注释的 requirements.txt

```bash
# ❌ 错误：pip 使用 GBK 读取，中文注释导致失败
pip install -r requirements-zh.txt
# UnicodeDecodeError: 'gbk' codec can't decode byte 0x80

# ✅ 解决方案 1：添加 UTF-8 BOM
python << 'EOF'
with open('requirements-zh.txt', 'rb') as f:
    data = f.read()
with open('requirements-zh.txt', 'wb') as f:
    f.write(b'\xef\xbb\xbf' + data)
EOF

# ✅ 解决方案 2：使用纯 ASCII 注释
# 将中文注释改为英文
```

### 案例 5：启动脚本中文乱码

```batch
:: ❌ 错误：chcp 65001 导致批处理文件中文解析异常
@echo off
chcp 65001 >nul
echo 推送渠道配置管理系统
:: 输出：'岃鍏堝畨瑁?Python' 不是内部或外部命令

:: ✅ 解决方案 1：使用英文输出
@echo off
set PYTHONUTF8=1
echo Push Channel Management System

:: ✅ 解决方案 2：使用 PowerShell 脚本
:: 创建 start.ps1 替代 start.bat
```

---

## 工具对比

### Bash vs Python vs PowerShell

| 维度 | Bash | Python | PowerShell |
|------|------|--------|------------|
| **速度** | ⭐⭐⭐⭐⭐ 最快 | ⭐⭐⭐ 启动慢 100ms | ⭐⭐⭐⭐ 中等 |
| **功能** | ⭐⭐⭐ 基础 | ⭐⭐⭐⭐⭐ 最丰富 | ⭐⭐⭐⭐ 丰富 |
| **编码安全** | ⭐⭐ Windows 下易失败 | ⭐⭐⭐⭐⭐ 完全可控 | ⭐⭐⭐⭐⭐ 原生支持 |
| **跨平台** | ⭐⭐ Linux/Mac 优先 | ⭐⭐⭐⭐⭐ 一致 | ⭐⭐⭐ Windows 优先 |
| **中文支持** | ⭐⭐ 易乱码 | ⭐⭐⭐⭐⭐ 完美 | ⭐⭐⭐⭐⭐ 完美 |

### 适用场景

| 场景 | 推荐工具 | 理由 |
|------|----------|------|
| 简单文件操作 | **Bash** | 简洁高效 |
| 处理中文/UTF-8 文件 | **Python** | 编码可控 |
| 复杂数据处理 | **Python** | 功能强大 |
| Windows 系统管理 | **PowerShell** | 原生支持 |
| 跨平台脚本 | **Python** | 行为一致 |

---

## 社区资源

### 官方 Issues（Anthropic）

| Issue | 描述 | 状态 |
|-------|------|------|
| [#7134](https://github.com/anthropics/claude-code/issues/7134) | 不尊重文件编码，破坏 Windows-1252 文件 | Not planned |
| [#13363](https://github.com/anthropics/claude-code/issues/13363) | Write 工具破坏 UTF-8 编码（BOM 丢失） | Not planned |
| [#52761](https://github.com/anthropics/claude-code/issues/52761) | Windows UTF-8 编码损坏 | Not planned |
| [#7332](https://github.com/anthropics/claude-code/issues/7332) | Bash 命令执行中文乱码 | Not planned |
| [#32410](https://github.com/anthropics/claude-code/issues/32410) | skill-creator 在 GBK Windows 上崩溃 | Not planned |

### 社区解决方案

| 项目 | 描述 | 链接 |
|------|------|------|
| `claude_encoding_guard` | 拦截文件操作，保留原始编码 | [ymonster/claude_encoding_guard](https://github.com/ymonster/claude_encoding_guard) |
| `Code-encoding-fix` | Windows 终端 UTF-8 配置 GUI 工具 | [hellowind777/Code-encoding-fix](https://github.com/hellowind777/Code-encoding-fix) |
| `opinionated-basic-cc-setup` | 智能上下文注入，包含编码验证 | [code-yeongyu/opinionated-basic-cc-setup](https://github.com/code-yeongyu/opinionated-basic-cc-setup) |

### 参考文章

- [Fixing Japanese Mojibake in Claude Code on Windows](https://zenn.dev/junko_ai/articles/9d5b2d4da8a353?locale=en) - 日本开发者的解决方案
- [Python Unicode HOWTO](https://docs.python.org/3/howto/unicode.html) - 官方文档
- [Windows UTF-8 Mode](https://docs.python.org/3/using/windows.html#utf-8-mode) - Python 官方

---

## CLAUDE.md 配置模板

在项目的 `CLAUDE.md` 中添加以下章节：

```markdown
## Windows 环境 UTF-8 编码问题处理指南

### 问题背景
Windows 系统默认编码为 GBK (cp936)，Claude Code 的 Bash/Read/Write 工具
在处理 UTF-8 中文文件时会出现编码错误。

### 常见错误现象
1. **Read 工具中断**：`[Tool use interrupted]`
2. **Bash 工具 GBK 解码错误**：
   ```
   UnicodeDecodeError: 'gbk' codec can't decode byte 0xae in position 1033
   Unexpected cygpath error, fallback to manual path conversion
   ```

### 根本原因
- Windows 系统默认编码为 GBK (cp936)
- Git Bash / MinGW 继承系统编码
- Python subprocess 模块使用 `locale.getpreferredencoding(False)` 读取子进程输出，默认是 GBK
- 当子进程输出包含 UTF-8 字节（如中文、特殊符号 `°` 等），GBK 解码失败

### 处理规则（强制遵循）

| 优先级 | 场景 | 推荐方案 | 示例 |
|--------|------|----------|------|
| P0 | 读取中文文件 | Python + encoding='utf-8' | `python -c "with open('f.txt', 'r', encoding='utf-8') as f: print(f.read())"` |
| P0 | 写入中文文件 | Python + encoding='utf-8' | `python -c "with open('f.txt', 'w', encoding='utf-8') as f: f.write('中文')"` |
| P1 | 文件列表 | Python os.listdir() | `python -c "import os; print(os.listdir('.'))"` |
| P2 | 简单命令（无中文） | Bash | `python --version` |
| P2 | Windows 特定操作 | PowerShell | `Get-Content file -Encoding UTF8` |

### 禁止事项
1. ❌ **严禁**在 Bash 中直接 `cat` 含有中文的文件（可能导致 Read 工具中断）
2. ❌ **严禁**在 Bash 中 `echo 中文 > 文件`
3. ❌ **严禁**依赖系统默认编码处理文本文件
4. ❌ **严禁**在输出中混用 GBK 和 UTF-8 编码
5. ❌ 文件路径避免使用中文和特殊字符（如 `°`）

### 应急处理流程
当 Read/Write/Bash 工具因编码问题中断时：

1. **立即改用 Python 脚本**
2. **检测文件编码**：
   ```bash
   python << 'EOF'
   with open('file.txt', 'rb') as f:
       data = f.read()
       print('First 20 bytes:', data[:20])
       if data[:3] == b'\xef\xbb\xbf':
           print('Has UTF-8 BOM')
       else:
           print('No BOM')
   EOF
   ```
3. **如需添加 UTF-8 BOM**：
   ```bash
   python << 'EOF'
   with open('file.txt', 'rb') as f:
       data = f.read()
   with open('file.txt', 'wb') as f:
       f.write(b'\xef\xbb\xbf' + data)
   print('UTF-8 BOM added')
   EOF
   ```

### 最佳实践速查表

| 场景 | 推荐工具 | 原因 |
|------|----------|------|
| 读取/写入 UTF-8 文件 | `python -c` | 可显式指定 encoding='utf-8' |
| 文件系统操作 | `python -c` | 编码可控，避免中文输出 |
| 简单命令（无中文输出） | `Bash` | 安全 |
| 复杂文本处理 | `Python` 脚本 | 编码可控 |
| Windows 特定操作 | `PowerShell` | 原生 UTF-8 支持 |

### 实际案例

#### 案例 1：读取中文批处理文件
```bash
# ❌ 错误：Read 工具会中断
Read start.bat.zh

# ✅ 正确：Python 读取
python -c "with open('start.bat.zh', 'r', encoding='utf-8') as f: print(f.read())"
```

#### 案例 2：创建中文注释的 requirements.txt
```bash
# ❌ 错误：直接写入会编码错误

# ✅ 正确：Python 写入
python << 'EOF'
content = '# 中文注释\nfastapi==0.109.0\n'
with open('requirements.txt', 'w', encoding='utf-8') as f:
    f.write(content)
EOF
```

#### 案例 3：检查文件是否存在中文乱码
```bash
# ✅ 正确：Python 检查
python << 'EOF'
with open('start.bat', 'rb') as f:
    data = f.read()
    try:
        text = data.decode('utf-8')
        print('UTF-8 OK')
    except:
        print('Not UTF-8')
EOF
```

### 环境信息

| 项目 | 值 |
|------|-----|
| 操作系统 | Windows 11 Pro |
| 系统默认编码 | GBK (cp936) |
| Shell | PowerShell / Git Bash |
| Python | 3.12 (miniconda3) |
| Git Bash 编码 | 继承系统 GBK |

### 相关错误码速查

| 错误 | 含义 | 解决方案 |
|------|------|----------|
| `[Tool use interrupted]` | 工具调用超时或中断 | 使用 Python 替代 Bash |
| `UnicodeDecodeError: 'gbk' codec can't decode byte 0xae` | GBK 无法解码 UTF-8 字节 | 使用 Python 显式指定 encoding='utf-8' |
| `cygpath error` | 路径转换工具编码错误 | 避免中文路径，使用 Python 处理路径 |

### 参考文档
- 项目文档：`docs/windows-utf8-encoding-issue.md`
- 技术报告：`docs/encoding-issue-report.md`
```

---

## FAQ

### Q1: 为什么 `PYTHONUTF8=1` 不能解决所有问题？

**A**: `PYTHONUTF8=1` 只影响 Python 解释器的标准流（stdin/stdout/stderr），但：
- ❌ 不影响 pip 的 `auto_decode()` 函数（使用 `locale.getpreferredencoding()`）
- ❌ 不影响 Bash 工具的子进程输出解码
- ❌ 不影响 Read/Write 工具的底层文件操作

### Q2: 为什么 `chcp 65001` 有时会导致更多问题？

**A**: `chcp 65001`（UTF-8 代码页）只能改变**终端显示**的编码，但：
- 批处理文件解析器仍然使用系统默认编码（GBK）
- 导致 CMD 期望 UTF-8，但系统组件仍用 GBK
- 混合编码产生更严重的乱码

### Q3: 如何永久解决 Windows 编码问题？

**A**: 推荐方案（按优先级）：
1. **项目级别**：在 `CLAUDE.md` 中添加上述编码规则
2. **文件级别**：对需要中文注释的文件添加 UTF-8 BOM
3. **系统级别**：使用 `Code-encoding-fix` 配置所有终端
4. **工具级别**：安装 `claude_encoding_guard` 拦截文件操作

### Q4: 哪些文件类型最容易受影响？

**A**: 
- 🔴 高风险：`.bat`, `.cmd`, `.ps1`, `.txt`, `.md`, `.py`（含中文注释）
- 🟡 中风险：`requirements.txt`, `README.md`, 配置文件
- 🟢 低风险：纯 ASCII 代码文件（`.py`, `.js`, `.json`）

### Q5: 如何检测文件是否已被编码损坏？

**A**: 使用 Python 检查：

```python
python << 'EOF'
with open('file.txt', 'rb') as f:
    data = f.read()

# 检查 BOM
if data[:3] == b'\xef\xbb\xbf':
    print('UTF-8 BOM detected')
else:
    print('No BOM')

# 尝试 UTF-8 解码
try:
    text = data.decode('utf-8')
    print('Valid UTF-8')
except UnicodeDecodeError as e:
    print(f'Invalid UTF-8: {e}')
    # 尝试 GBK
    try:
        text = data.decode('gbk')
        print('Valid GBK (file is GBK encoded)')
    except:
        print('Unknown encoding')
EOF
```

---

## 总结

### 核心原则

1. **优先使用 Python**：处理中文文件时，始终使用 Python 并显式指定 `encoding='utf-8'`
2. **避免 Bash 中文**：不在 Bash 中输出或处理中文字符
3. **使用 Here Document**：多行 Python 代码使用 `python << 'EOF'` 格式
4. **添加 UTF-8 BOM**：需要被其他工具读取的中文文件，添加 BOM 头
5. **配置 CLAUDE.md**：在项目指南中明确编码处理规则

### 快速决策流程

```
需要处理中文文件？
    ├── 是 → 使用 Python + encoding='utf-8'
    │         ├── 单行代码 → python -c "..."
    │         └── 多行代码 → python << 'EOF' ... EOF
    └── 否 → 使用 Bash（确保无中文输出）
              └── 复杂操作 → 使用 Python
```

---

## 贡献

欢迎提交 Issue 和 PR，分享你的 Windows 编码问题和解决方案！

## 许可证

MIT License

---

*最后更新：2026-07-05*
*适用环境：Windows 10/11 + Python 3.8+ + Claude Code*
