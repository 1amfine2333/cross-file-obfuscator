# Go Code Obfuscator (Cross-File Obfuscator)

Go code obfuscation tool using AST technology for cross-file obfuscation, ensuring obfuscated code is **compilable and executable**.

[中文][url-doczh] | [Full Documentation](README_EN_FULL.md)

## Core Features

- **One-Click Auto Mode** - Full obfuscation + automatic compilation
- **Multi-Layer Obfuscation** - Function names + variable names + string encryption + control flow + binary obfuscation
- **Smart Protection** - Auto-protect exported symbols, reflection, JSON/XML, interfaces, CGO
- **Linker-Level Obfuscation** - Directly modify binary files to obfuscate function names and package paths
- **Auto Package Discovery** - Intelligently discover and obfuscate project packages and standard library

## Quick Start

### Build

```bash
go build -o cross-file-obfuscator cmd/main.go
```

### Basic Usage

```bash
# 🚀 Recommended: Auto mode (one command for everything)
./cross-file-obfuscator -auto -output-bin myapp ./my-project

# Multi-entry project (main in subdirectory)
./cross-file-obfuscator -auto -entry "./cmd/server" -output-bin server ./my-project

# Basic obfuscation (source code only)
./cross-file-obfuscator ./my-project

# Full obfuscation (all options)
./cross-file-obfuscator -obfuscate-exported -obfuscate-filenames -encrypt-strings -inject-junk ./my-project
```

## Main Features

### 1. Function and Variable Name Obfuscation
- By default, obfuscate unexported (lowercase) functions and variables
- Optionally obfuscate exported symbols (`-obfuscate-exported`)
- Maintain cross-file reference consistency

### 2. String Encryption
- XOR encrypt all string literals
- Automatic runtime decryption
- 64-byte random key

### 3. Control Flow Obfuscation
- Inject opaque predicates (always-true/false conditions)
- Increase reverse engineering difficulty

### 4. Linker-Level Obfuscation
- Modify function names and package paths in binary files
- Support ELF (Linux), PE (Windows), Mach-O (macOS)
- Automatic package discovery and replacement

### 5. Smart Protection Mechanism
- **Auto-protect**: Struct fields, methods, interfaces, built-in identifiers
- **Reflection Protection**: Auto-detect and protect code using `reflect`
- **Serialization Protection**: Smart handling of JSON/XML tags
- **CGO Skip**: Automatically skip CGO code
- **Generated Code Skip**: Automatically skip protobuf and other generated code

## Command Line Options

### Basic Options
```bash
-o <dir>                     Output directory (default: project_obfuscated)
-obfuscate-exported          Obfuscate exported functions and variables
-obfuscate-filenames         Obfuscate file names
-encrypt-strings             Encrypt strings
-inject-junk                 Inject junk code
-remove-comments             Remove comments (default: true)
-preserve-reflection         Protect reflection (default: true)
-exclude <pattern>           Exclude files (e.g., '*_test.go,*.pb.go')
```

### Auto Mode Options
```bash
-auto                        Auto mode (recommended)
-output-bin <filename>       Output binary filename
-entry <package>             Entry package path (required for multi-entry projects)
-auto-discover-pkgs          Auto-discover and replace project packages (recommended)
-obfuscate-third-party       Obfuscate third-party dependencies (use with caution)
-only-project                Only obfuscate project packages (recommended for Windows)
```

## Usage Examples

### Standard Project
```bash
# main.go in root directory
./cross-file-obfuscator -auto -output-bin myapp ./my-project
```

### Multi-Entry Project
```bash
# main.go in cmd/server/main.go
./cross-file-obfuscator -auto -entry "./cmd/server" -output-bin server ./my-project
```

### Windows Platform (Avoid Antivirus False Positives)
```bash
export GOOS=windows GOARCH=amd64
./cross-file-obfuscator -auto -output-bin app.exe ./my-project
# Automatically uses minimal obfuscation strategy (only project packages)
```

### Step-by-Step Obfuscation
```bash
# Step 1: Source code obfuscation
./cross-file-obfuscator -encrypt-strings -inject-junk ./my-project

# Step 2: Linker obfuscation
./cross-file-obfuscator -build-with-linker -auto-discover-pkgs \
  -output-bin myapp ./my-project_obfuscated
```

## Important Notes

### Multi-Entry Projects Must Specify -entry

**How to determine?**
```bash
# Find main function location
find ./your-project -name "*.go" -exec grep -l "func main()" {} \;
```

| Project Structure | Need -entry? | Command |
|------------------|--------------|---------|
| `main.go` in root | ❌ | `./cross-file-obfuscator -auto -output-bin app ./project` |
| `cmd/app/main.go` | ✅ | `./cross-file-obfuscator -auto -entry "./cmd/app" -output-bin app ./project` |

### pclntab Obfuscation Strategy

| Strategy | Description | Use Case |
|----------|-------------|----------|
| **Full Obfuscation** | Obfuscate all packages (stdlib + project) | Linux/macOS (default) |
| **Minimal Obfuscation** | Only obfuscate project packages | Windows (auto-enabled) |
| **Disabled** | Don't modify pclntab | Conservative approach |

```bash
# Manually specify minimal obfuscation
./cross-file-obfuscator -build-with-linker -auto-discover-pkgs \
  -only-project -output-bin myapp ./my-project

# Completely disable pclntab obfuscation
./cross-file-obfuscator -build-with-linker -auto-discover-pkgs \
  -disable-pclntab -output-bin myapp ./my-project
```

## Troubleshooting

### Archive Output Error

**Symptom**:
```bash
$ file output
output: current ar archive  # Error
```

**Cause**: Main package not in root directory, but `-entry` parameter not specified

**Solution**:
```bash
# 1. Find main package location
find ./project -name "*.go" -exec grep -l "func main()" {} \;

# 2. Add -entry parameter
./cross-file-obfuscator -auto -entry "./cmd/gost" -output-bin output ./project
```

### Compilation Failure

1. Check if `-entry` parameter is needed
2. Ensure `-preserve-reflection=true` (default)
3. Use `-exclude` to exclude problematic files
4. Disable advanced features for testing: don't use `-encrypt-strings` and `-inject-junk`

### Runtime Errors

1. **Reflection Issues**: Check if reflection-related types are protected
2. **Interface Implementation**: Tool automatically protects interface methods
3. **JSON/XML**: Ensure serialization fields have explicit tags
4. **CGO Code**: Tool automatically skips CGO files

## FAQ

**Q: Why isn't my function obfuscated?**  
A: By default, only unexported functions (lowercase) are obfuscated. Use `-obfuscate-exported` for exported functions.

**Q: How to verify obfuscation success?**  
A: Use `strings myapp | grep '^main\.'` to check function name prefixes, should return 0.

**Q: How to exclude a directory?**  
A: Use `-exclude "dirname/*"` pattern.

**Q: Antivirus false positive on Windows?**  
A: AUTO mode automatically uses minimal obfuscation strategy on Windows, or manually add `-only-project` option.

## Technical Principles

### Workflow
1. **Collect Protected Names** - Identify identifiers to protect (struct fields, interface methods, etc.)
2. **Scope Analysis** - Build complete scope tree, handle cross-file references
3. **Build Obfuscation Mapping** - Generate unique random obfuscated names
4. **Copy Project Files** - Filter special files like CGO, generated code
5. **Apply Obfuscation** - Precisely replace identifiers, apply advanced obfuscation techniques

### Protection Mechanism
- **Five-Layer Protection**: Special identifiers → Built-in identifiers → Exported names → Structured protection → Context protection
- **Scope-Aware**: Precisely identify identifier scopes, avoid incorrect replacements
- **Build Tag Support**: Correctly handle platform-specific code (`_linux.go`, `_windows.go`)

## License

MIT License

[url-doczh]: README.md
