# 编译

在仓库根目录用解决方案编译。本机已验证的是 Release x64。

环境：

- Windows，PowerShell
- Visual Studio 2026 Build Tools
- 平台工具集 `v145`
- MSBuild 18.10：`C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\MSBuild\Current\Bin\MSBuild.exe`

在 `c:\work\github\rufus` 执行。`msbuild` 已在 `PATH` 中时，用 cmd：

```bat
msbuild rufus.sln /p:Configuration=Release /p:Platform=x64 /m /v:minimal
```

PowerShell 里若找不到该命令，用完整路径：

```powershell
& "C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\MSBuild\Current\Bin\MSBuild.exe" rufus.sln /p:Configuration=Release /p:Platform=x64 /m /v:minimal
```

生成 `x64\Release\rufus.exe`。

`rufus.sln` 引用的是 `.vs\*.vcxproj`。单独编译 `.vs\rufus.vcxproj` 时，`$(SolutionDir)` 不对，生成 `embedded.loc` 会找不到 `res\loc\rufus.loc`。

Debug 把 `Configuration` 改成 `Debug`。x86 把 `Platform` 改成 `x86`，输出在 `x86\Release\` 或 `x86\Debug\`。
