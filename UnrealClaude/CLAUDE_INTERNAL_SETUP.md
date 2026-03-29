# 使用 claude-internal 的配置方法与问题修复

## 配置方法

### 第一步：将插件注册到项目

在 `Lyra.uproject`（或你的项目 `.uproject`）的 `Plugins` 列表中添加：

```json
{
    "Name": "UnrealClaude",
    "Enabled": true
}
```

> **说明：** 本插件仓库根目录下有一层 `UnrealClaude/` 子目录，实际 `.uplugin` 在第二层。UE 支持递归扫描，能正确找到，但必须在 `.uproject` 中显式注册插件名才会编译。

### 第二步：创建配置文件

在插件目录下创建配置文件（**此文件不会提交到 git，需手动创建**）：

```
插件目录/Config/UnrealClaude.ini
```

内容如下：

```ini
[UnrealClaude]
ClaudeExecutablePath=C:\Users\你的用户名\AppData\Roaming\npm\claude-internal.cmd
```

将路径替换为你机器上实际的 `claude-internal.cmd` 路径。

**查找路径方法（Windows）：**
```cmd
where claude-internal.cmd
```

### 第三步：编译并重启编辑器

编译项目后重启 UE 编辑器，插件会自动加载配置。

---

## 已知问题与修复

### 问题 1：插件一直显示 "Claude is thinking"，没有任何回复

**症状：**
- UI 上永远显示 "Claude is thinking"
- 日志中出现 `exit code: 1`，输出为乱码（约 19 字符）

**根本原因：**

`claude-internal` 有一个嵌套会话保护机制：启动时检测 `CLAUDECODE` 环境变量，若存在则认为自己是在另一个 Claude Code 会话中被嵌套启动，直接拒绝运行并退出（exit code 1）。

错误信息原文（在终端中运行可见）：
```
Error: Claude Code cannot be launched inside another Claude Code session.
Nested sessions share runtime resources and will crash all active sessions.
To bypass this check, unset the CLAUDECODE environment variable.
```

UE 编辑器通过 Rider（或其他 IDE）启动时，会继承操作系统/用户环境变量。如果当前环境中存在 `CLAUDECODE`（claude-internal 运行时会自动设置），插件每次启动 claude-internal 子进程都会被这个检测挡住。

**修复方案（已应用到 `ClaudeCodeRunner.cpp`）：**

在 `LaunchProcess()` 中，`CreateProc` 调用前临时清空 `CLAUDECODE`，子进程创建完成后立即恢复：

```cpp
FString PrevClaudeCode = FPlatformMisc::GetEnvironmentVariable(TEXT("CLAUDECODE"));
if (!PrevClaudeCode.IsEmpty())
{
    FPlatformMisc::SetEnvironmentVar(TEXT("CLAUDECODE"), TEXT(""));
    UE_LOG(LogUnrealClaude, Log, TEXT("Temporarily cleared CLAUDECODE env var to allow nested claude-internal launch"));
}

ProcessHandle = FPlatformProcess::CreateProc(...);

// 恢复原值
if (!PrevClaudeCode.IsEmpty())
{
    FPlatformMisc::SetEnvironmentVar(TEXT("CLAUDECODE"), *PrevClaudeCode);
}
```

这样子进程看不到 `CLAUDECODE`，不触发嵌套检测；UE 编辑器自身的环境不受影响。

---

### 问题 2：配置文件中的 `ClaudeExecutablePath` 被忽略，自动使用了标准 `claude.cmd`

**症状：**
- `UnrealClaude.ini` 中配置了 `claude-internal.cmd`
- 但日志显示实际调用的是 `claude.cmd`

**根本原因：**

原版 `GetClaudePath()` 中没有读取 ini 配置文件的逻辑，直接按固定顺序扫描常见路径，找到标准 `claude.cmd` 就返回了。

**修复方案（已应用到 `ClaudeCodeRunner.cpp`）：**

在 `GetClaudePath()` 自动扫描之前，先通过 `IPluginManager` 获取插件真实目录，再读取 ini 配置：

```cpp
FString ConfigPath;
TSharedPtr<IPlugin> Plugin = IPluginManager::Get().FindPlugin(TEXT("UnrealClaude"));
if (Plugin.IsValid())
{
    ConfigPath = FPaths::Combine(Plugin->GetBaseDir(), TEXT("Config"), TEXT("UnrealClaude.ini"));
}

FString ConfiguredPath;
if (GConfig->GetString(TEXT("UnrealClaude"), TEXT("ClaudeExecutablePath"), ConfiguredPath, ConfigPath))
{
    ConfiguredPath.TrimStartAndEndInline();
    if (!ConfiguredPath.IsEmpty() && IFileManager::Get().FileExists(*ConfiguredPath))
    {
        UE_LOG(LogUnrealClaude, Log, TEXT("Found Claude CLI from config: %s"), *ConfiguredPath);
        CachedClaudePath = ConfiguredPath;
        return CachedClaudePath;
    }
}
// 配置不存在或文件不存在时，回退到自动扫描
```

---

### 问题 3：MCP bridge 找不到，日志报 "MCP bridge not found"

**症状：**
```
LogUnrealClaude: Warning: MCP bridge not found at: .../Plugins/UnrealClaude/Resources/mcp-bridge/index.js
```
- 命令行中缺少 `--mcp-config` 参数
- Claude 没有 `mcp__unrealclaude__*` 工具，无法操作编辑器

**根本原因：**

原版 `GetPluginDirectory()` 将插件目录硬编码为 `ProjectPluginsDir() / "UnrealClaude"`。但本仓库的结构是两层嵌套：

```
Plugins/
  UnrealClaude/          ← 仓库根目录（无 .uplugin）
    UnrealClaude/        ← 实际插件目录（有 .uplugin、Resources/）
```

导致路径少了一层 `UnrealClaude/`，找不到 `Resources/mcp-bridge/index.js`。

**修复方案（已应用到 `ClaudeCodeRunner.cpp`）：**

改用 `IPluginManager` 获取插件真实目录，不依赖硬编码路径：

```cpp
static FString GetPluginDirectory()
{
    TSharedPtr<IPlugin> Plugin = IPluginManager::Get().FindPlugin(TEXT("UnrealClaude"));
    if (Plugin.IsValid())
    {
        return Plugin->GetBaseDir();
    }
    UE_LOG(LogUnrealClaude, Warning, TEXT("Could not find UnrealClaude plugin via IPluginManager"));
    return FString();
}
```

`IPluginManager` 在插件加载时已经解析了真实路径，无论插件嵌套多少层都能正确返回。

---

## 修复后的验证

重启 UE 编辑器后，在 Output Log 中确认以下日志：

```
LogUnrealClaude: Found Claude CLI from config: C:\Users\...\claude-internal.cmd
LogUnrealClaude: MCP config written to: .../Saved/UnrealClaude/mcp-config.json
LogUnrealClaude: Async executing Claude: ...claude-internal.cmd ... --mcp-config ...
LogUnrealClaude: NDJSON Result: subtype=success, is_error=0, duration=...ms, cost=$...
```

关键检查点：
- 执行的是 `claude-internal.cmd` 而非 `claude.cmd`
- 命令行中包含 `--mcp-config` 参数
- 出现 `subtype=success` 说明通信正常
