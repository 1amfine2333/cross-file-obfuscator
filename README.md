# Go 代码混淆器 (Cross-File Obfuscator)

Go 代码混淆工具，使用 AST 技术实现跨文件代码混淆，保证混淆后代码**可编译和可执行**。

[English][url-docen] | [完整文档](README_FULL.md)

## 核心特性

- **一键自动模式** - 全功能混淆 + 自动编译
- **多层混淆** - 函数名 + 变量名 + 字符串加密 + 控制流混淆 + 二进制混淆
- **智能保护** - 自动保护导出符号、反射、JSON/XML、接口、CGO
- **链接器混淆** - 直接修改二进制文件混淆函数名和包路径
- **自动包名发现** - 智能发现并混淆项目包名、标准库包名

## 快速开始

### 编译

```bash
go build -o cross-file-obfuscator cmd/main.go
```

### 基本使用

```bash
# 🚀 推荐：自动模式（一键完成所有混淆）
./cross-file-obfuscator -auto -output-bin myapp ./my-project

# 多入口项目（main在子目录）
./cross-file-obfuscator -auto -entry "./cmd/server" -output-bin server ./my-project

# 基础混淆（仅源码）
./cross-file-obfuscator ./my-project

# 完整混淆（所有选项）
./cross-file-obfuscator -obfuscate-exported -obfuscate-filenames -encrypt-strings -inject-junk ./my-project
```

## 主要功能

### 1. 函数名和变量名混淆
- 默认混淆未导出（小写开头）的函数和变量
- 可选混淆导出符号（`-obfuscate-exported`）
- 保持跨文件引用一致性

### 2. 字符串加密
- XOR 加密所有字符串字面量
- 运行时自动解密
- 64字节随机密钥

### 3. 控制流混淆
- 注入不透明谓词（永真/永假条件）
- 增加逆向分析难度

### 4. 链接器级别混淆
- 修改二进制文件中的函数名和包路径
- 支持 ELF (Linux)、PE (Windows)、Mach-O (macOS)
- 自动包名发现和替换

### 5. 智能保护机制
- **自动保护**：结构体字段、方法、接口、内置标识符
- **反射保护**：自动检测并保护使用 `reflect` 的代码
- **序列化保护**：智能处理 JSON/XML 标签
- **CGO 跳过**：自动跳过 CGO 代码
- **生成代码跳过**：自动跳过 protobuf 等生成的代码

## 命令行选项

### 基础选项
```bash
-o <目录>                    输出目录（默认：项目名_obfuscated）
-obfuscate-exported          混淆导出的函数和变量
-obfuscate-filenames         混淆文件名
-encrypt-strings             加密字符串
-inject-junk                 注入垃圾代码
-remove-comments             删除注释（默认：true）
-preserve-reflection         保护反射（默认：true）
-exclude <模式>              排除文件（如：'*_test.go,*.pb.go'）
```

### 自动模式选项
```bash
-auto                        自动模式（推荐）
-output-bin <文件名>         输出二进制文件名
-entry <包路径>              入口包路径（多入口项目必须指定）
-auto-discover-pkgs          自动发现并替换项目包名（推荐）
-obfuscate-third-party       混淆第三方依赖（谨慎使用）
-only-project                只混淆项目包（Windows 推荐）
```

## 使用示例

### 标准项目
```bash
# main.go 在根目录
./cross-file-obfuscator -auto -output-bin myapp ./my-project
```

### 多入口项目
```bash
# main.go 在 cmd/server/main.go
./cross-file-obfuscator -auto -entry "./cmd/server" -output-bin server ./my-project
```

### Windows 平台（避免杀软误报）
```bash
export GOOS=windows GOARCH=amd64
./cross-file-obfuscator -auto -output-bin app.exe ./my-project
# 自动使用最小化混淆策略（只混淆项目包）
```

### 分步混淆
```bash
# 第一步：源码混淆
./cross-file-obfuscator -encrypt-strings -inject-junk ./my-project

# 第二步：链接器混淆
./cross-file-obfuscator -build-with-linker -auto-discover-pkgs \
  -output-bin myapp ./my-project_obfuscated
```

## 重要说明

### 多入口项目必须指定 -entry

**如何判断？**
```bash
# 查找 main 函数位置
find ./your-project -name "*.go" -exec grep -l "func main()" {} \;
```

| 项目结构 | 是否需要-entry | 命令 |
|---------|---------------|------|
| `main.go` 在根目录 | ❌ | `./cross-file-obfuscator -auto -output-bin app ./project` |
| `cmd/app/main.go` | ✅ | `./cross-file-obfuscator -auto -entry "./cmd/app" -output-bin app ./project` |

### pclntab 混淆策略

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| **完整混淆** | 混淆所有包（标准库+项目） | Linux/macOS（默认） |
| **最小化混淆** | 只混淆项目包 | Windows（自动启用） |
| **禁用混淆** | 不修改 pclntab | 保守方案 |

```bash
# 手动指定最小化混淆
./cross-file-obfuscator -build-with-linker -auto-discover-pkgs \
  -only-project -output-bin myapp ./my-project

# 完全禁用 pclntab 混淆
./cross-file-obfuscator -build-with-linker -auto-discover-pkgs \
  -disable-pclntab -output-bin myapp ./my-project
```

## 故障排除

### Archive 输出错误

**症状**：
```bash
$ file output
output: current ar archive  # 错误
```

**原因**：main 包不在根目录，但没有指定 `-entry` 参数

**解决**：
```bash
# 1. 查找 main 包位置
find ./project -name "*.go" -exec grep -l "func main()" {} \;

# 2. 添加 -entry 参数
./cross-file-obfuscator -auto -entry "./cmd/gost" -output-bin output ./project
```

### 编译失败

1. 检查是否需要指定 `-entry` 参数
2. 确保 `-preserve-reflection=true`（默认）
3. 使用 `-exclude` 排除问题文件
4. 关闭高级功能测试：不使用 `-encrypt-strings` 和 `-inject-junk`

### 运行时错误

1. **反射问题**：检查反射相关的类型是否被保护
2. **接口实现**：工具会自动保护接口方法
3. **JSON/XML**：确保序列化字段有显式标签
4. **CGO 代码**：工具会自动跳过 CGO 文件

## 常见问题

**Q: 为什么我的函数没有被混淆？**  
A: 默认只混淆未导出函数（小写开头）。导出函数需要使用 `-obfuscate-exported` 选项。

**Q: 如何验证混淆是否成功？**  
A: 使用 `strings myapp | grep '^main\.'` 检查函数名前缀，应该返回 0。

**Q: 如何排除某个目录？**  
A: 使用 `-exclude "dirname/*"` 模式。

**Q: Windows 下被杀软误报？**  
A: AUTO 模式在 Windows 下自动使用最小化混淆策略，或手动添加 `-only-project` 选项。

## 技术原理

### 工作流程
1. **收集保护名称** - 识别需要保护的标识符（结构体字段、接口方法等）
2. **作用域分析** - 构建完整的作用域树，处理跨文件引用
3. **构建混淆映射** - 生成唯一的随机混淆名称
4. **复制项目文件** - 过滤 CGO、生成代码等特殊文件
5. **应用混淆** - 精确替换标识符，应用高级混淆技术

### 保护机制
- **五层保护**：特殊标识符 → 内置标识符 → 导出名称 → 结构化保护 → 上下文保护
- **作用域感知**：精确识别标识符作用域，避免错误替换
- **Build 标签支持**：正确处理平台相关代码（`_linux.go`、`_windows.go`）

## 许可证

MIT License

[url-docen]: README_EN.md
