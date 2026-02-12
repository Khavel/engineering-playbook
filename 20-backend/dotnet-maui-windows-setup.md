# .NET MAUI Windows Desktop Setup (.NET 10+)

## Problem

Setting up .NET MAUI Windows-only desktop apps with .NET 10 involves non-obvious pitfalls: workload installer bugs, SDK name confusion, package version conflicts, and runtime identifier requirements.

## Context

Creating a standalone Windows desktop app using .NET MAUI (not cross-platform) targeting .NET 10.0 with modern tooling (CommunityToolkit.Mvvm, CommunityToolkit.Maui).

## Solution

### 1. Workload Installation & Verification

```powershell
# Install MAUI Windows workload
dotnet workload install maui-windows

# Verify installation
dotnet workload list
```

**Critical Bug**: The MSI installer may not create the `InstalledWorkloads` marker file, causing `WorkloadSdkResolver returned null` errors even after successful installation.

**Fix** (requires elevated PowerShell):
```powershell
$markerPath = "C:\Program Files\dotnet\metadata\workloads\10.0.100\InstalledWorkloads"
if (!(Test-Path $markerPath)) {
    New-Item -ItemType Directory -Path $markerPath -Force
}
New-Item -ItemType File -Path "$markerPath\maui-windows" -Force
```

### 2. Project File (.csproj) Structure

**Key Insight**: Use `Sdk="Microsoft.NET.Sdk"` (NOT `Microsoft.NET.Sdk.Maui`) with `<UseMaui>true</UseMaui>`.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFrameworks>net10.0-windows10.0.19041.0</TargetFrameworks>
    <OutputType>Exe</OutputType>
    <UseMaui>true</UseMaui>
    <SingleProject>true</SingleProject>
    
    <!-- Required for Windows-only MSIX packaging -->
    <RuntimeIdentifier>win-x64</RuntimeIdentifier>
    
    <!-- Suppress false-positive package warnings -->
    <NoWarn>$(NoWarn);NU1605</NoWarn>
    <SkipValidateMauiImplicitPackageReferences>true</SkipValidateMauiImplicitPackageReferences>
  </PropertyGroup>

  <ItemGroup>
    <!-- CommunityToolkit.Maui 14.0.0 requires MAUI >= 10.0.30, workload bundles 10.0.20 -->
    <PackageReference Include="CommunityToolkit.Maui" Version="14.0.0" />
    <PackageReference Include="CommunityToolkit.Mvvm" Version="8.*" />
    
    <!-- Explicit references resolve version conflicts -->
    <PackageReference Include="Microsoft.Maui.Controls" Version="10.0.*" />
    <PackageReference Include="Microsoft.Maui.Controls.Compatibility" Version="10.0.*" />
  </ItemGroup>
</Project>
```

### 3. Package Version Conflicts

**Problem**: `CommunityToolkit.Maui` 14.0.0 requires `Microsoft.Maui.Controls >= 10.0.30`, but MAUI workload 10.0.20 bundles version 10.0.20.

**Solution**:
- Use `NoWarn=NU1605` to suppress downgrade warnings
- Use `SkipValidateMauiImplicitPackageReferences=true` to bypass version checks
- Add explicit `Microsoft.Maui.Controls` package reference with wildcard version `10.0.*`

### 4. Build Commands

```powershell
# Clean build (uses RuntimeIdentifier from csproj)
dotnet build src/YourApp/YourApp.csproj

# Explicit runtime targeting (if not set in csproj)
dotnet build src/YourApp/YourApp.csproj -r win-x64

# Run app
dotnet run --project src/YourApp/YourApp.csproj
```

### 5. Platform-Specific Files

**Platforms/Windows/App.xaml**:
```xml
<maui:MauiWinUIApplication
    x:Class="YourNamespace.WinUI.App"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:maui="using:Microsoft.Maui">
</maui:MauiWinUIApplication>
```

**Namespace**: Use `YourNamespace.WinUI` (not `YourNamespace`)

**Code-behind**: Inherit from `MauiWinUIApplication` (not `Application`)

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `WorkloadSdkResolver returned null` | Missing InstalledWorkloads marker | Create marker file manually (see above) |
| `Microsoft.NET.Sdk.Maui not found` | Wrong SDK declaration | Use `Sdk="Microsoft.NET.Sdk"` + `<UseMaui>true</UseMaui>` |
| `NU1605: Detected package downgrade` | CommunityToolkit.Maui version conflict | Add `NoWarn=NU1605` + explicit package refs |
| `MSIX packaging requires RuntimeIdentifier` | Missing RID for Windows packaging | Add `<RuntimeIdentifier>win-x64</RuntimeIdentifier>` |
| `MVVMTK0045: Not AOT compatible` | Using fields with `[ObservableProperty]` | Warning only - safe to ignore for WinUI 3 |

## Verification

```powershell
# Should show 0 errors
dotnet build src/YourApp/YourApp.csproj

# Should launch dark-themed window
dotnet run --project src/YourApp/YourApp.csproj
```

## Related Patterns

- [DI Registration](dotnet-architecture.md#dependency-injection): Register services in `MauiProgram.CreateMauiApp()`
- [MVVM with CommunityToolkit](dotnet-architecture.md#mvvm): Use `[ObservableProperty]`, `[RelayCommand]`
- [Dark Theme Colors](../99-templates/maui-dark-theme.md): GitHub/VS Code palette (if template exists)

## When to Use

- Windows-only desktop apps requiring native OS integration (file pickers, notifications)
- Replacing ASP.NET Core web UIs with native desktop UIs
- Building standalone executables without web server dependencies

## Timing

**Setup time**: ~30 minutes (including troubleshooting workload installer bug)  
**Build time**: ~60 seconds (first build), ~5 seconds (incremental)
