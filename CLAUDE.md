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

#### 案例 1：读取中文批处理文件bash
# ❌ 错误：Read 工具会中断
Read start.bat.zh

# ✅ 正确：Python 读取
python -c "with open('start.bat.zh', 'r', encoding='utf-8') as f: print(f.read())"

#### 案例 2：创建中文注释的 requirements.txtbash
# ❌ 错误：直接写入会编码错误

# ✅ 正确：Python 写入
python << 'EOF'
content = '# 中文注释\nfastapi==0.109.0\n'
with open('requirements.txt', 'w', encoding='utf-8') as f:
    f.write(content)
EOF

#### 案例 3：检查文件是否存在中文乱码bash
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