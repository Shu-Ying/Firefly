---
title: Win11 鼠标指针失效
published: 2026-09-28
description: 文章简介
tags: [Windows优化]
category: Windows
---

自 Win11 25H2 26220.9223 就已经有这个BUG了，第三方指针自定义不显示，而是默认指针。新建1.ps1脚本文件，运行即可，不需要管理员权限和重启电脑。
```powershell
# 代码来自Chat GPT6
# 从 HKCU\Control Panel\Cursors 读取当前鼠标方案
# 并通过 SetSystemCursor 强制恢复 Windows 系统鼠标指针

Add-Type @"
using System;
using System.Runtime.InteropServices;

public static class CursorFix
{
    [DllImport("user32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
    public static extern IntPtr LoadCursorFromFile(string lpFileName);

    [DllImport("user32.dll", SetLastError = true)]
    public static extern bool SetSystemCursor(IntPtr hcur, uint id);

    [DllImport("user32.dll", SetLastError = true)]
    public static extern IntPtr CopyImage(
        IntPtr h,
        uint type,
        int cx,
        int cy,
        uint flags
    );
}
"@

# Windows 系统 Cursor ID
$CursorMap = [ordered]@{
    Arrow       = 32512  # OCR_NORMAL
    IBeam       = 32513  # OCR_IBEAM
    Wait        = 32514  # OCR_WAIT
    Crosshair   = 32515  # OCR_CROSS
    UpArrow     = 32516  # OCR_UP
    NWPen       = 32631  # OCR_NWPEN
    SizeNWSE    = 32642  # OCR_SIZENWSE
    SizeNESW    = 32643  # OCR_SIZENESW
    SizeWE      = 32644  # OCR_SIZEWE
    SizeNS      = 32645  # OCR_SIZENS
    SizeAll     = 32646  # OCR_SIZEALL
    No          = 32648  # OCR_NO
    Hand        = 32649  # OCR_HAND
    AppStarting = 32650  # OCR_APPSTARTING
    Help        = 32651  # OCR_HELP
}

$RegPath = "HKCU:\Control Panel\Cursors"

try {
    $Cursors = Get-ItemProperty -Path $RegPath -ErrorAction Stop
}
catch {
    Write-Host "无法读取 $RegPath" -ForegroundColor Red
    exit 1
}

Write-Host ""
Write-Host "正在恢复当前鼠标指针方案..." -ForegroundColor Cyan
Write-Host ""

foreach ($Entry in $CursorMap.GetEnumerator()) {

    $Name = $Entry.Key
    $ID   = [uint32]$Entry.Value

    $Path = $Cursors.$Name

    if ([string]::IsNullOrWhiteSpace($Path)) {
        Write-Host "[跳过] $Name - 注册表中没有路径" -ForegroundColor DarkGray
        continue
    }

    # 展开 %SystemRoot% 等环境变量
    $Path = [Environment]::ExpandEnvironmentVariables($Path)

    if (-not (Test-Path -LiteralPath $Path)) {
        Write-Host "[失败] $Name - 文件不存在: $Path" -ForegroundColor Red
        continue
    }

    # 加载 .cur / .ani
    $Cursor = [CursorFix]::LoadCursorFromFile($Path)

    if ($Cursor -eq [IntPtr]::Zero) {
        Write-Host "[失败] $Name - 无法加载: $Path" -ForegroundColor Red
        continue
    }

    # SetSystemCursor 会接管/销毁传入的 Cursor Handle，
    # 因此先复制一份再交给它。
    $CursorCopy = [CursorFix]::CopyImage(
        $Cursor,
        2,      # IMAGE_CURSOR
        0,
        0,
        0
    )

    if ($CursorCopy -eq [IntPtr]::Zero) {
        Write-Host "[失败] $Name - 无法复制 Cursor Handle" -ForegroundColor Red
        continue
    }

    $Result = [CursorFix]::SetSystemCursor($CursorCopy, $ID)

    if ($Result) {
        Write-Host "[成功] $Name -> $Path" -ForegroundColor Green
    }
    else {
        $ErrorCode = [Runtime.InteropServices.Marshal]::GetLastWin32Error()
        Write-Host "[失败] $Name - Win32 Error: $ErrorCode" -ForegroundColor Red
    }
}

Write-Host ""
Write-Host "鼠标指针恢复完成。" -ForegroundColor Cyan
```