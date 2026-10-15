# 在 Windows 下使用 MSBuild 编译 Rufus

本文说明如何在 Windows 上用 **MSBuild** 编译 Rufus，内容基于本机实测环境。

> **关于终端：** 下面所有命令按 **PowerShell** 和 **cmd.exe** 分别给出。
> 两者语法不同，**请勿混用**（尤其是路径引号、`.exe` 的调用方式和 `||`/`&&` 这类连接符）。

---

## 1. 环境要求

| 项目 | 本机实测值 |
| --- | --- |
| Visual Studio | Visual Studio 生成工具 2026（BuildTools），版本 18.10.1 |
| 平台工具集（PlatformToolset） | `v145` |
| MSVC 工具链 | 14.51.36231 |
| Windows SDK | 10.0.26100.0 |
| MSBuild | 18.10.1.42706 |

只需要 **BuildTools**（生成工具）即可编译，**不需要**安装完整的 Visual Studio IDE（即不需要 `devenv.exe`）。

### 检查是否已安装

**PowerShell**

```powershell
& "C:\Program Files (x86)\Microsoft Visual Studio\Installer\vswhere.exe" -products * -property displayName
```

**cmd.exe**

```cmd
"%ProgramFiles(x86)%\Microsoft Visual Studio\Installer\vswhere.exe" -products * -property displayName
```

如果提示找不到文件，请先下载安装 [Visual Studio 生成工具](https://visualstudio.microsoft.com/zh-hans/downloads/)，安装时勾选 **“使用 C++ 的桌面开发”** 工作负载。

---

## 2. 找到 MSBuild

MSBuild 的路径随安装位置变化，建议**每次都先确认路径**，不要在文档里硬记。

### 方法一：用 vswhere 自动定位（推荐）

**PowerShell**

```powershell
$vswhere = "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe"
$msbuild = & $vswhere -latest -products * -requires Microsoft.Component.MSBuild -find "MSBuild\**\Bin\MSBuild.exe"
Write-Host "MSBuild 路径: $msbuild"
```

**cmd.exe**

```cmd
for /f "usebackq tokens=*" %i in (`"%ProgramFiles(x86)%\Microsoft Visual Studio\Installer\vswhere.exe" -latest -products * -requires Microsoft.Component.MSBuild -find "MSBuild\**\Bin\MSBuild.exe"`) do set MSBUILD=%i
echo MSBuild 路径: %MSBUILD%
```

> 在 `.bat`/`.cmd` 脚本里，`%i` 要写成 `%%i`。

### 方法二：直接用完整路径

本机实测路径：

```
C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\MSBuild\Current\Bin\MSBuild.exe
```

注意路径里有 **空格** 和 **括号**，所以：

- **PowerShell**：用 `&` 调用操作符 + 双引号
  ```powershell
  & "C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\MSBuild\Current\Bin\MSBuild.exe" -version
  ```
- **cmd.exe**：直接用双引号包住即可
  ```cmd
  "C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\MSBuild\Current\Bin\MSBuild.exe" -version
  ```

### 方法三：初始化命令行环境后直接用 `msbuild`

如果你希望直接用 `msbuild` 这个短命令，需要先运行 VS 的环境脚本。
**这一步只在 cmd.exe 下有意义**，具体见 [第 6 节](#6-cmdexe-下编译)。

---

## 3. 解决方案与配置

解决方案文件：[rufus.sln](rufus.sln)

它支持 **6 种** 配置组合（定义在 [rufus.sln](rufus.sln) 的 `SolutionConfigurationPlatforms` 段）：

| 解决方案的 Platform | 项目实际使用的 Platform | Configuration | 输出目录 | 输出文件 |
| --- | --- | --- | --- | --- |
| `x64` | `x64` | `Release` / `Debug` | `x64\Release\`、`x64\Debug\` | `rufus.exe` |
| `x86` | `Win32` | `Release` / `Debug` | `x86\Release\`、`x86\Debug\` | `rufus.exe` |
| `arm64` | `ARM64` | `Release` / `Debug` | `arm64\Release\`、`arm64\Debug\` | `rufus.exe` |

**要点：** 命令行里 `Platform` 传 `x86`（不是 `Win32`）也可以，MSBuild 会自动映射；但直接对 `.vs\rufus.vcxproj` 编译时要传 `Win32`。

---

## 4. PowerShell 下编译（推荐）

PowerShell 不需要初始化任何环境变量，直接用 `&` 调用完整路径即可。

### 4.1 标准编译（Release / x64）

```powershell
cd C:\work\github\rufus

& "C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\MSBuild\Current\Bin\MSBuild.exe" `
    rufus.sln /p:Configuration=Release /p:Platform=x64 /m
```

产物：`C:\work\github\rufus\x64\Release\rufus.exe`

> **PowerShell 的关键点：** 参数是用 `/p:Configuration=Release /p:Platform=x64` **两个独立参数** 传的。
> 不要写成 `/p:Configuration=Release,Platform=x64`（逗号形式是给 cmd.exe 用的，且 `,` 在 PowerShell 里是数组分隔符）。
> 换行符是反引号 ``` ` ```，**不是** `\`。

### 4.2 编译 32 位与 ARM64

```powershell
# 32 位
& $msbuild rufus.sln /p:Configuration=Release /p:Platform=x86 /m

# ARM64
& $msbuild rufus.sln /p:Configuration=Release /p:Platform=arm64 /m
```

### 4.3 全量重编

改动头文件后如果增量编译出现诡异的报错，用 `/t:Rebuild`：

```powershell
& $msbuild rufus.sln /t:Rebuild /p:Configuration=Release /p:Platform=x64 /m
```

### 4.4 只看错误

```powershell
& $msbuild rufus.sln /p:Configuration=Release /p:Platform=x64 /m /v:m /nologo 2>&1 |
    Select-String ": error"
```

**PowerShell 的坑：** `2>&1` 必须写（否则 MSBuild 的 stderr 不会进入管道），而 `&&`、`||`、`;` 的语义与 cmd 不同。
在 Windows PowerShell 5.1 里 `&&`/`||` **不支持**，请用分号或 `if ($LASTEXITCODE -eq 0) { ... }`。

### 4.5 判断编译是否成功

```powershell
& $msbuild rufus.sln /p:Configuration=Release /p:Platform=x64 /m /v:m /nologo
if ($LASTEXITCODE -ne 0) { Write-Host "编译失败" -ForegroundColor Red } else { Write-Host "编译成功" -ForegroundColor Green }
```

### 4.6 可直接复制的一整段

```powershell
cd C:\work\github\rufus
$msbuild = & "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" `
    -latest -products * -requires Microsoft.Component.MSBuild `
    -find "MSBuild\**\Bin\MSBuild.exe"

& $msbuild rufus.sln /p:Configuration=Release /p:Platform=x64 /m /v:m /nologo
if ($LASTEXITCODE -ne 0) { throw "编译失败" }
Write-Host "产物: $(Resolve-Path .\x64\Release\rufus.exe)"
```

---

## 5. MSBuild 常用参数

| 参数 | 说明 |
| --- | --- |
| `/p:Configuration=Release` | 指定配置（`Debug` / `Release`） |
| `/p:Platform=x64` | 指定平台（`x64` / `x86` / `arm64`） |
| `/m` | 多进程并行编译，明显加快速度 |
| `/t:Rebuild` | 先清理再全量编译 |
| `/t:Clean` | 只清理（删除 `x64\Release\` 下的中间文件） |
| `/v:m` | 输出详细度：minimal，适合看摘要 |
| `/v:q` | 输出详细度：quiet，只看错误 |
| `/nologo` | 不打印 MSBuild 版本横幅 |
| `/p:ForceImportBeforeCppTargets=...` | 注入额外属性表（CI 用它生成完整 PDB，本地一般不需要） |

---

## 6. cmd.exe 下编译

cmd.exe 有两种用法，**区别在于是否需要先用 `vcvars`/`VsDevCmd` 初始化环境**。

### 6.1 方式 A：不初始化环境（最简单）

和 PowerShell 一样直接用完整路径，只是调用方式不同（不需要 `&`）：

```cmd
cd /d C:\work\github\rufus

"C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\MSBuild\Current\Bin\MSBuild.exe" rufus.sln /p:Configuration=Release /p:Platform=x64 /m
```

**cmd 的关键点：**
- 路径直接加双引号，**不需要** PowerShell 的 `&` 操作符。
- `cd /d` 才能跨盘符切换目录。
- 参数也可以用逗号形式：`/p:Configuration=Release,Platform=x64`（Linux CI 上也是这么写的）。
- 可以用 `&&` 串联命令，例如：
  ```cmd
  msbuild rufus.sln /p:Configuration=Release /p:Platform=x64 /m && echo 编译成功
  ```

### 6.2 方式 B：先用 vcvars 初始化，之后直接用 `msbuild`

如果你想用短命令 `msbuild`，就需要先执行 VS 的环境脚本。

**注意：** `vcvars64.bat` / `VsDevCmd.bat` **必须和编译命令在同一个 cmd.exe 进程里执行**，
因为环境变量（`PATH`、`LIB`、`INCLUDE`）不能跨进程保留。
所以**不要**“先在一个窗口里跑 `vcvars`，再在另一个窗口里跑 `msbuild`”。

**在 cmd.exe 里（交互式，先初始化再编译）：**

```cmd
call "C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\VC\Auxiliary\Build\vcvars64.bat"
cd /d C:\work\github\rufus
msbuild rufus.sln /p:Configuration=Release /p:Platform=x64 /m
```

同理：
- 32 位用 `vcvars32.bat`
- ARM64 用 `vcvarsall.bat arm64`
- 也可以用 `VsDevCmd.bat`：
  ```cmd
  call "C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\Common7\Tools\VsDevCmd.bat" -arch=x64
  ```

**在 PowerShell 里调用 cmd 脚本（谨慎）：**
PowerShell 无法真正“继承” `.bat` 设置的环境变量，一次性执行时要把初始化和编译放进**同一个** cmd 调用里：

```powershell
cd C:\work\github\rufus
& $env:ComSpec /c 'call "C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\VC\Auxiliary\Build\vcvars64.bat" >nul 2>&1 && msbuild rufus.sln /p:Configuration=Release /p:Platform=x64 /m'
```

这段其实等价于方式 A，只是多绕了一层，**日常建议直接用 4.1 的写法**。

### 6.3 PowerShell 与 cmd.exe 语法对照

| 场景 | PowerShell | cmd.exe |
| --- | --- | --- |
| 调用带空格路径的程序 | `& "C:\...\MSBuild.exe" args` | `"C:\...\MSBuild.exe" args` |
| 切换盘符+目录 | `cd C:\work\github\rufus` | `cd /d C:\work\github\rufus` |
| 传两个属性 | `/p:Configuration=Release /p:Platform=x64` | 同上，或 `/p:Configuration=Release,Platform=x64` |
| 串联命令 | `cmd1; cmd2`（5.1 不支持 `&&`） | `cmd1 && cmd2` |
| 命令换行 | 反引号 `` ` `` | 尖角号 `^` |
| 捕获 stderr | `2>&1` | `2>&1` |
| 取退出码 | `$LASTEXITCODE` | `%ERRORLEVEL%` |

---

## 7. 编译产物

| 路径 | 说明 |
| --- | --- |
| `x64\Release\rufus.exe` | 主程序（x64 Release） |
| `x64\Release\rufus.pdb` | 调试符号 |
| `res\loc\embedded.loc` | 编译时由 pre-build 事件生成的本地化数据 |

`res\loc\embedded.loc` 是 **自动生成** 的（已被 `.gitignore` 忽略），
由 [.vs/rufus.vcxproj](.vs/rufus.vcxproj) 的 pre-build 步骤产生：

```cmd
type $(SolutionDir)res\loc\rufus.loc | findstr /v MSG_9 > $(SolutionDir)res\loc\embedded.loc
```

即：把 [res/loc/rufus.loc](res/loc/rufus.loc) 里 `MSG_9xx`（仅用于 Microsoft Store 页面文案）的行过滤掉。
**所以改完 `rufus.loc` 后重新编译就会自动生效，不必手动生成。**

---

## 8. 常见问题

### LNK1104：无法打开文件 rufus.exe

```
LINK : fatal error LNK1104: 无法打开文件“...\x64\Release\rufus.exe”
```

**原因：** 上一个编译出来的 `rufus.exe` **正在运行**，文件被占用，链接器无法覆盖。

**解决：** 关闭正在运行的 Rufus，然后重新编译。

**PowerShell 下查找并结束占用进程：**

```powershell
Get-Process rufus -ErrorAction SilentlyContinue | Select-Object Id, Path
Stop-Process -Id <上面列出的 PID>
```

### warning C4819：文件包含不能在当前代码页中表示的字符

这是**上游原有的告警**（提示源文件含非当前代码页-936 能表示的字符），与你的改动无关，**可以安全忽略**。

> 提醒：网上常见的 `/p:AdditionalOptions="/utf-8"` 写法对本项目**无效**（已实测），
> 因为 `AdditionalOptions` 是编译项（item）的元数据，而不是可由命令行覆盖的项目属性。
> 若确实想消除该告警，只能修改 `.vcxproj`，在对应配置的 `<ClCompile>` 里加入 `<AdditionalOptions>/utf-8</AdditionalOptions>`——这属于改动项目文件，一般不建议。

### 提示“找不到 Windows SDK”或“找不到 v145 工具集”

说明 BuildTools 安装不完整。请在 Visual Studio Installer 里确认已勾选：

- **使用 C++ 的桌面开发**（Desktop development with C++）
- 对应的 **Windows 10/11 SDK**（本机为 10.0.26100.0）

### 只编译单个项目

对 `.vcxproj` 单独编译时，`$(SolutionDir)` 是**未定义**的，会导致 pre-build 事件失败：

```
error MSB3073: 命令“type ...\.vs\res\loc\rufus.loc | findstr /v MSG_9 > ...\.vs\res\loc\embedded.loc”已退出，代码为 255
```

必须显式传入 `SolutionDir`（**注意结尾要带反斜杠**）：

**PowerShell**

```powershell
& $msbuild .vs\rufus.vcxproj /p:Configuration=Release /p:Platform=x64 `
    /p:SolutionDir=C:\work\github\rufus\ /m
```

**cmd.exe**

```cmd
"C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\MSBuild\Current\Bin\MSBuild.exe" .vs\rufus.vcxproj /p:Configuration=Release,Platform=x64 /p:SolutionDir=C:\work\github\rufus\ /m
```

注意此时 Platform 要写**项目自身**的取值（`x64`、`Win32`、`ARM64`），而不是解决方案里的 `x86`。

> 日常编译**建议直接编译 `rufus.sln`**，可省去这些麻烦。

### 更详细的日志

```powershell
& $msbuild rufus.sln /p:Configuration=Release /p:Platform=x64 /v:diag > build.log 2>&1
```

---

## 9. 关于 MinGW

[README.md](README.md) 也提到可以用 MinGW（`configure` + `make`）。
**本机没有安装 gcc/mingw32-make**（只有 Gow 提供的 `make.exe`），所以这套流程未经验证。
若确实需要，请先安装 [MinGW-w64](https://www.mingw-w64.org/)，再参考仓库里的 `configure.ac` / `Makefile.am`。

---

## 10. 速查

**PowerShell（推荐）**

```powershell
cd C:\work\github\rufus
& "C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\MSBuild\Current\Bin\MSBuild.exe" rufus.sln /p:Configuration=Release /p:Platform=x64 /m
# 产物：x64\Release\rufus.exe
```

**cmd.exe（不初始化环境）**

```cmd
cd /d C:\work\github\rufus
"C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\MSBuild\Current\Bin\MSBuild.exe" rufus.sln /p:Configuration=Release /p:Platform=x64 /m
```

**cmd.exe（用 vcvars 初始化后直接用 msbuild）**

```cmd
call "C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\VC\Auxiliary\Build\vcvars64.bat"
cd /d C:\work\github\rufus
msbuild rufus.sln /p:Configuration=Release /p:Platform=x64 /m
```
