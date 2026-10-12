# Script
ASD



$VideoUrl = "https://www.youtube.com/watch?v=VIDEO_ID"

# Video: 3 minutes; cycle: 10 minutes
$VideoSeconds = 180
$CycleSeconds = 600

$ProfileDir = Join-Path $env:LOCALAPPDATA "VideoLoopProfile"
$StopFile = Join-Path $env:TEMP "video-loop-stop.txt"

$EdgePaths = @(
    "$env:ProgramFiles(x86)\Microsoft\Edge\Application\msedge.exe",
    "$env:ProgramFiles\Microsoft\Edge\Application\msedge.exe",
    "$env:LOCALAPPDATA\Microsoft\Edge\Application\msedge.exe"
)

$Edge = $EdgePaths | Where-Object { Test-Path $_ } |
    Select-Object -First 1

if (-not $Edge) {
    Write-Error "Microsoft Edge not found."
    exit 1
}

Remove-Item $StopFile -Force -ErrorAction SilentlyContinue
$ErrorActionPreference = "Continue"

function Stop-VideoWindow {
    Get-CimInstance Win32_Process -Filter "Name='msedge.exe'" |
        Where-Object {
            $_.CommandLine -and
            $_.CommandLine.Contains($ProfileDir)
        } |
        ForEach-Object {
            Stop-Process -Id $_.ProcessId -Force `
                -ErrorAction SilentlyContinue
        }
}

function Wait-WithStopCheck([int]$Seconds) {
    for ($i = 0; $i -lt $Seconds; $i++) {
        if (Test-Path $StopFile) { return $false }
        Start-Sleep -Seconds 1
    }
    return -not (Test-Path $StopFile)
}

while (-not (Test-Path $StopFile)) {
    $CycleStart = Get-Date

    Start-Process -FilePath $Edge -ArgumentList @(
        "--user-data-dir=$ProfileDir",
        "--new-window",
        $VideoUrl
    )

    if (-not (Wait-WithStopCheck $VideoSeconds)) {
        Stop-VideoWindow
        break
    }

    Stop-VideoWindow

    $Elapsed = ((Get-Date) - $CycleStart).TotalSeconds
    $Remaining = [Math]::Max(
        0, [int]($CycleSeconds - $Elapsed)
    )

    if (-not (Wait-WithStopCheck $Remaining)) { break }
}

Stop-VideoWindow
Remove-Item $StopFile -Force -ErrorAction SilentlyContinue



$scriptPath = (Resolve-Path ".\video-loop.ps1").Path

Start-Process powershell.exe -WindowStyle Hidden -ArgumentList @(
    "-NoProfile",
    "-ExecutionPolicy", "Bypass",
    "-File", "`"$scriptPath`""
)






New-Item -Path "$env:TEMP\video-loop-stop.txt" -ItemType File -Force




______________________________


$scriptPath = (Resolve-Path ".\video-loop.ps1").Path
$startup = [Environment]::GetFolderPath("Startup")
$launcher = Join-Path $startup "VideoLoop.vbs"

@"
Set WshShell = CreateObject("WScript.Shell")
WshShell.Run "powershell.exe -NoProfile -ExecutionPolicy Bypass -File ""$scriptPath""", 0, False
"@ | Set-Content -Path $launcher -Encoding Default

Write-Host "Автозапуск настроен: $launcher" -ForegroundColor Green
