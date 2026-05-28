---
name: dev-env-guide
description: Guide for finding SDKs/tools via environment variables, and auto-configuring PATH and env vars when installing or uninstalling development tools. Use whenever locating, installing, uninstalling, or configuring any SDK, compiler, runtime, package manager, or build tool — or when PATH/env var setup is needed.
---

# Dev Env Guide

## Core Principle

When locating SDKs, tools, runtimes, or executables, **always check environment variables first** before falling back to filesystem search (grep/find).

## Step 1: Check Environment Variables

### On Windows (PowerShell)

```powershell
# List all environment variables (shows full values, no truncation)
Get-ChildItem env: | Format-List Name, Value

# Check a specific variable
$env:VARIABLE_NAME

# Find executable in PATH
where.exe <command>
```

### On macOS / Linux (Bash)

```bash
# List all environment variables
env | sort

# Check a specific variable
echo $VARIABLE_NAME

# Find executable in PATH (command -v is POSIX standard, which as fallback)
command -v <command> || which <command>
```

## Step 2: Check Common SDK-Specific Environment Variables

| SDK / Tool    | Environment Variables                            |
| ------------- | ------------------------------------------------ |
| Java / JDK    | `JAVA_HOME`, `JDK_HOME`                          |
| Python        | `PYTHON_HOME`, `PYTHONPATH`                      |
| Conda         | `CONDA_PREFIX`, `CONDA_EXE`                      |
| venv          | `VIRTUAL_ENV`                                    |
| Node.js       | `NODE_HOME`, `NODE_PATH`, `NVM_DIR`              |
| Go            | `GOROOT`, `GOPATH`                               |
| Android SDK   | `ANDROID_HOME`, `ANDROID_SDK_ROOT`               |
| Flutter       | `FLUTTER_HOME`, `FLUTTER_ROOT`                   |
| Dart          | `DART_HOME`, `DART_SDK`                          |
| Gradle        | `GRADLE_HOME`, `GRADLE_USER_HOME`                |
| Maven         | `MAVEN_HOME`, `M2_HOME`                          |
| Rust / Cargo  | `CARGO_HOME`, `RUSTUP_HOME`                      |
| .NET          | `DOTNET_ROOT`, `DOTNET_HOME`                     |
| Ruby / Gem    | `RUBY_HOME`, `GEM_HOME`                          |
| LLVM / Clang  | `LLVM_HOME`, `LLVM_PATH`                         |
| CUDA          | `CUDA_HOME`, `CUDA_PATH`                         |
| OpenSSL       | `OPENSSL_ROOT_DIR`                               |
| CMake         | `CMAKE_PREFIX_PATH`                              |
| pkg-config    | `PKG_CONFIG_PATH`                                |
| MinGW / MSYS2 | `MSYSTEM_PREFIX`, `MINGW_HOME`, `MSYS_HOME`      |
| Qt            | `QTDIR`, `QT_PLUGIN_PATH`                        |
| OpenCV        | `OpenCV_DIR`                                     |
| Boost         | `BOOST_ROOT`                                     |
| Protobuf      | `PROTOBUF_HOME`, `PROTOC_HOME`                   |
| npm global    | `NPM_CONFIG_PREFIX`                              |

## Step 3: Search PATH for Executables

If no dedicated env var is found, search `PATH` directly — this is much faster than scanning the filesystem.

```powershell
# Windows PowerShell — find matching executables in PATH
$env:PATH -split ';' | ForEach-Object {
    Get-ChildItem -Path $_ -Filter "<pattern>*" -ErrorAction SilentlyContinue
} | Select-Object FullName
```

```bash
# macOS / Linux — find matching executables in PATH
IFS=':' read -ra DIRS <<< "$PATH"; for dir in "${DIRS[@]}"; do
    ls "$dir"/<pattern>* 2>/dev/null
done
```

## Step 4: Global Filesystem Search (Last Resort)

Only use filesystem search when:
1. No relevant environment variable is set
2. The executable is not found in PATH
3. You need a specific version that is not on PATH

### Search Strategy

Users (especially novices) may install SDKs anywhere — desktop, downloads, custom drive root, etc. **Do not limit search to "standard" directories.** Search globally, but do it smartly:

1. **Search by filename pattern, not file content** — `find` by name is fast; `grep -r` by content is slow and noisy
2. **Exclude system/noise directories** to speed up search and reduce false positives
3. **Limit depth** where reasonable to avoid traversing deep nested structures

### Windows (PowerShell)

```powershell
# Search all drives, exclude system/noise dirs, match by name pattern
$drives = Get-PSDrive -PSProvider FileSystem | ForEach-Object { $_.Root }
foreach ($drive in $drives) {
    Get-ChildItem -Path $drive -Filter "*<sdk-name>*" -Directory -ErrorAction SilentlyContinue -Depth 5 |
        Where-Object { 
            $_.FullName -notlike 'C:\Windows\*' -and 
            $_.FullName -notlike '*\$Recycle.Bin*' -and 
            $_.Name -notin @('node_modules', '.git')
        }
}
```

Or use `cmd` for broader coverage (no depth limit). First list available drives with `wmic logicaldisk get name`, then search each:
```cmd
dir /s /b C:\*<sdk-name>* D:\*<sdk-name>* 2>nul
```
Replace drive letters with those actually present on the system.

### macOS / Linux (Bash)

```bash
# Search from root, exclude system/noise dirs, limit depth
find / -maxdepth 7 -name "*<sdk-name>*" -type d \
    -not -path "*/proc/*" \
    -not -path "*/sys/*" \
    -not -path "*/dev/*" \
    -not -path "*/node_modules/*" \
    -not -path "*/.git/*" \
    2>/dev/null
```

## Step 5: Auto-Configure Environment After Install / Uninstall

When you install or uninstall a tool/SDK, **always manage the user's environment variables** so the tool remains discoverable.

### When Installing a Tool

After downloading/installing a tool, add its bin directory to the user's PATH and set any relevant SDK-specific environment variables.

#### Windows (PowerShell)

```powershell
# Add to user PATH (persistent)
$toolBinPath = "C:\path\to\tool\bin"
$currentUserPath = [Environment]::GetEnvironmentVariable("PATH", [EnvironmentVariableTarget]::User)
if ($currentUserPath -notlike "*$toolBinPath*") {
    [Environment]::SetEnvironmentVariable("PATH", "$currentUserPath;$toolBinPath", [EnvironmentVariableTarget]::User)
}

# Set SDK-specific env var (e.g., JAVA_HOME, ANDROID_HOME)
[Environment]::SetEnvironmentVariable("SDK_NAME_HOME", "C:\path\to\sdk", [EnvironmentVariableTarget]::User)

# Refresh current session so it takes effect immediately
$env:PATH = [Environment]::GetEnvironmentVariable("PATH", [EnvironmentVariableTarget]::User)
```

#### macOS / Linux

```bash
# Determine the right shell config file
SHELL_CONFIG="$HOME/.bashrc"
[ -n "$ZSH_VERSION" ] && SHELL_CONFIG="$HOME/.zshrc"
[ -f "$HOME/.bash_profile" ] && SHELL_CONFIG="$HOME/.bash_profile"
[ -f "$HOME/.profile" ] && SHELL_CONFIG="$HOME/.profile"
[ -n "$FISH_VERSION" ] && SHELL_CONFIG="$HOME/.config/fish/config.fish"

# Add to PATH (avoid duplicate entries)
grep -q 'export PATH="\$PATH:/path/to/tool/bin"' "$SHELL_CONFIG" 2>/dev/null || \
    echo 'export PATH="$PATH:/path/to/tool/bin"' >> "$SHELL_CONFIG"

# Set SDK-specific env var
grep -q 'export SDK_NAME_HOME=' "$SHELL_CONFIG" 2>/dev/null || \
    echo 'export SDK_NAME_HOME="/path/to/sdk"' >> "$SHELL_CONFIG"

# Apply immediately in current session
export PATH="$PATH:/path/to/tool/bin"
```

### When Uninstalling a Tool

When removing a tool, **also clean up its environment entries** — otherwise stale PATH entries and env vars accumulate.

#### Windows (PowerShell)

```powershell
# Remove from user PATH
$toolBinPath = "C:\path\to\tool\bin"
$currentUserPath = [Environment]::GetEnvironmentVariable("PATH", [EnvironmentVariableTarget]::User)
$newPath = ($currentUserPath -split ';' | Where-Object { $_ -ne $toolBinPath }) -join ';'
[Environment]::SetEnvironmentVariable("PATH", $newPath, [EnvironmentVariableTarget]::User)

# Remove SDK env var entirely
[Environment]::SetEnvironmentVariable("SDK_NAME_HOME", $null, [EnvironmentVariableTarget]::User)
```

#### macOS / Linux

```bash
# Remove lines containing the tool path from all shell configs
# macOS sed requires a backup suffix (.bak); Linux sed -i works without one.
# Using -i.bak is safe on both platforms — backup files can be deleted after.
for config in "$HOME/.bashrc" "$HOME/.zshrc" "$HOME/.bash_profile" "$HOME/.profile" "$HOME/.config/fish/config.fish"; do
    [ -f "$config" ] && sed -i.bak '\#/path/to/tool/bin#d' "$config"
    [ -f "$config" ] && sed -i.bak '/export SDK_NAME_HOME=/d' "$config"
done
```

## Step 6: Verify Configuration

After making environment changes, confirm they took effect.

```powershell
# Windows — verify PATH entry was added
[Environment]::GetEnvironmentVariable("PATH", [EnvironmentVariableTarget]::User) -split ';' | Select-String "<tool-name>"

# Verify SDK env var was set
[Environment]::GetEnvironmentVariable("SDK_NAME_HOME", [EnvironmentVariableTarget]::User)

# Verify executable is discoverable in current session
where.exe <command>
```

```bash
# macOS / Linux — verify PATH entry was added
grep -q '/path/to/tool/bin' "$SHELL_CONFIG" && echo "PATH entry found" || echo "PATH entry missing"

# Verify SDK env var was set
grep -q 'SDK_NAME_HOME=' "$SHELL_CONFIG" && echo "env var found" || echo "env var missing"

# Verify executable is callable in current session
command -v <command>
```

> **Note:** Changes to Windows user environment variables via `[Environment]::SetEnvironmentVariable` only affect **new** processes. The current PowerShell session's `$env:PATH` will not reflect the change unless you also run the "Refresh current session" step from Step 5. On macOS/Linux, the `export` command in the current session takes effect immediately, but new terminal windows will pick up the shell config file.

## Anti-Patterns to Avoid

- Running `grep -r "sdk" /` — searching file **contents** globally is extremely slow and noisy; use `find -name` (by filename) instead
- Assuming SDK paths without checking environment variables first
- Scanning the entire filesystem when `PATH` or dedicated env vars are already set — always do env vars first
- Limiting search to only "standard" directories like `/usr/local/` or `C:\Program Files\` — many users install tools in custom locations
- Searching without excluding noise directories (`/proc`, `/sys`, `node_modules`, `.git`) — leads to permission errors and irrelevant results
- Installing a tool without adding it to user PATH or setting relevant env vars — the tool won't be discoverable in future sessions
- Uninstalling a tool without cleaning up its PATH entries and env vars — accumulates stale configuration over time
