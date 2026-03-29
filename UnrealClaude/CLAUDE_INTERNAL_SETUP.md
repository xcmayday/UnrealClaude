# 使用 claude-internal 的配置方法与问题修复

## 配置方法

编辑插件配置文件：

```
UnrealClaude/Config/UnrealClaude.ini
```

内容如下：

```ini
[UnrealClaude]
ClaudeExecutablePath=C:\Users\你的用户名\AppData\Roaming\npm\claude-internal.cmd
```

将路径替换为你机器上实际的 `claude-internal.cmd` 路径。

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

在 `GetClaudePath()` 自动扫描之前，先尝试读取 ini 配置：

```cpp
FString ConfigPath = FPaths::Combine(FPaths::ProjectPluginsDir(), TEXT("UnrealClaude"), TEXT("Config"), TEXT("UnrealClaude.ini"));
FString ConfiguredPath;
if (GConfig->GetString(TEXT("UnrealClaude"), TEXT("ClaudeExecutablePath"), ConfiguredPath, ConfigPath))
{
    ConfiguredPath.TrimStartAndEndInline();
    if (!ConfiguredPath.IsEmpty() && IFileManager::Get().FileExists(*ConfiguredPath))
    {
        CachedClaudePath = ConfiguredPath;
        return CachedClaudePath;
    }
}
// 配置不存在或文件不存在时，回退到自动扫描
```

---

## 修复后的验证

重启 UE 编辑器后，在 Output Log 中确认以下日志：

```
LogUnrealClaude: Found Claude CLI from config: C:\Users\...\claude-internal.cmd
LogUnrealClaude: Temporarily cleared CLAUDECODE env var to allow nested claude-internal launch
LogUnrealClaude: NDJSON Result: subtype=success, is_error=0, ...
```

出现 `subtype=success` 说明通信正常。
