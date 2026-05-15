<#
Script Name : Collect-SystemInstallationInfo.ps1
Purpose     : Collect system installation/audit details safely without downtime
Impact      : Read-only. No restart, no service restart, no configuration changes.
Output      : HTML report, TXT summary, CSV files
Author      : IT Admin
#>

param(
    [string]$OutputRoot = "C:\Temp\IT-System-Inventory",
    [switch]$SkipInstalledSoftware
)

$ErrorActionPreference = "SilentlyContinue"

# -----------------------------
# Basic setup
# -----------------------------

$Timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
$ComputerName = $env:COMPUTERNAME
$ReportFolder = Join-Path $OutputRoot "$ComputerName`_$Timestamp"

New-Item -ItemType Directory -Path $ReportFolder -Force | Out-Null

$SummaryTxt = Join-Path $ReportFolder "SystemInstallationSummary.txt"
$HtmlReport = Join-Path $ReportFolder "SystemInstallationReport.html"

function Write-Section {
    param(
        [string]$Title,
        $Data
    )

    "`n==================== $Title ====================" | Out-File $SummaryTxt -Append -Encoding UTF8

    if ($null -eq $Data) {
        "No data found or access denied." | Out-File $SummaryTxt -Append -Encoding UTF8
    }
    else {
        $Data | Format-List * | Out-String | Out-File $SummaryTxt -Append -Encoding UTF8
    }
}

function Export-SafeCsv {
    param(
        [string]$Name,
        $Data
    )

    if ($null -ne $Data) {
        $CsvPath = Join-Path $ReportFolder "$Name.csv"
        $Data | Export-Csv -Path $CsvPath -NoTypeInformation -Encoding UTF8
    }
}

function ConvertTo-SafeHtmlSection {
    param(
        [string]$Title,
        $Data
    )

    $html = "<h2>$Title</h2>"

    if ($null -eq $Data) {
        $html += "<p>No data found or access denied.</p>"
    }
    else {
        $html += ($Data | ConvertTo-Html -Fragment)
    }

    return $html
}

function Test-IsAdmin {
    $CurrentUser = [Security.Principal.WindowsIdentity]::GetCurrent()
    $Principal = New-Object Security.Principal.WindowsPrincipal($CurrentUser)
    return $Principal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
}

$IsAdmin = Test-IsAdmin

# -----------------------------
# 1. Collection metadata
# -----------------------------

$CollectionInfo = [PSCustomObject]@{
    ComputerName       = $ComputerName
    CollectedBy        = "$env:USERDOMAIN\$env:USERNAME"
    CollectionTime     = Get-Date
    RunningAsAdmin     = $IsAdmin
    OutputFolder       = $ReportFolder
    ScriptImpact       = "Read-only collection. No changes made."
}

# -----------------------------
# 2. Computer, OS, BIOS details
# -----------------------------

$ComputerSystem = Get-CimInstance Win32_ComputerSystem | Select-Object `
    Name,
    Domain,
    PartOfDomain,
    Workgroup,
    Manufacturer,
    Model,
    UserName,
    NumberOfProcessors,
    NumberOfLogicalProcessors,
    @{Name="RAM_GB";Expression={[math]::Round($_.TotalPhysicalMemory / 1GB, 2)}}

$BIOS = Get-CimInstance Win32_BIOS | Select-Object `
    SerialNumber,
    SMBIOSBIOSVersion,
    Manufacturer,
    ReleaseDate

$OS = Get-CimInstance Win32_OperatingSystem | Select-Object `
    Caption,
    Version,
    BuildNumber,
    OSArchitecture,
    InstallDate,
    LastBootUpTime,
    WindowsDirectory,
    SystemDirectory

# -----------------------------
# 3. Hostname and domain details
# -----------------------------

$DomainInfo = [PSCustomObject]@{
    HostName       = $ComputerName
    Domain         = $ComputerSystem.Domain
    PartOfDomain   = $ComputerSystem.PartOfDomain
    LoggedInUser   = $ComputerSystem.UserName
}

# -----------------------------
# 4. Network, IP, DNS details
# -----------------------------

$NetworkConfig = Get-NetIPConfiguration | Where-Object {
    $_.IPv4Address -ne $null
} | Select-Object `
    InterfaceAlias,
    InterfaceDescription,
    @{Name="IPv4Address";Expression={$_.IPv4Address.IPAddress -join ", "}},
    @{Name="IPv4PrefixLength";Expression={$_.IPv4Address.PrefixLength -join ", "}},
    @{Name="DefaultGateway";Expression={$_.IPv4DefaultGateway.NextHop -join ", "}},
    @{Name="DNSServers";Expression={$_.DNSServer.ServerAddresses -join ", "}}

$IPAdapterDetails = Get-CimInstance Win32_NetworkAdapterConfiguration | Where-Object {
    $_.IPEnabled -eq $true
} | Select-Object `
    Description,
    DHCPEnabled,
    DHCPServer,
    @{Name="IPAddress";Expression={$_.IPAddress -join ", "}},
    @{Name="SubnetMask";Expression={$_.IPSubnet -join ", "}},
    @{Name="DefaultGateway";Expression={$_.DefaultIPGateway -join ", "}},
    @{Name="DNSServers";Expression={$_.DNSServerSearchOrder -join ", "}},
    MACAddress

# -----------------------------
# 5. Disk and file system details
# -----------------------------

$DiskVolumes = Get-Volume | Select-Object `
    DriveLetter,
    FileSystemLabel,
    FileSystem,
    DriveType,
    HealthStatus,
    OperationalStatus,
    @{Name="Size_GB";Expression={[math]::Round($_.Size / 1GB, 2)}},
    @{Name="FreeSpace_GB";Expression={[math]::Round($_.SizeRemaining / 1GB, 2)}}

$PhysicalDisks = Get-CimInstance Win32_DiskDrive | Select-Object `
    Model,
    SerialNumber,
    MediaType,
    InterfaceType,
    @{Name="Size_GB";Expression={[math]::Round($_.Size / 1GB, 2)}}

# -----------------------------
# 6. Firewall status
# -----------------------------

$FirewallProfiles = Get-NetFirewallProfile | Select-Object `
    Name,
    Enabled,
    DefaultInboundAction,
    DefaultOutboundAction

# -----------------------------
# 7. Local users and administrators
# -----------------------------

$LocalUsers = Get-LocalUser | Select-Object `
    Name,
    Enabled,
    LastLogon,
    PasswordRequired,
    PasswordLastSet

$LocalAdministrators = Get-LocalGroupMember -Group "Administrators" | Select-Object `
    Name,
    ObjectClass,
    PrincipalSource

# -----------------------------
# 8. Printers
# -----------------------------

$Printers = Get-Printer | Select-Object `
    Name,
    DriverName,
    PortName,
    PrinterStatus,
    Shared,
    Published

# -----------------------------
# 9. Shared folders and mapped drives
# -----------------------------

$LocalShares = Get-SmbShare | Select-Object `
    Name,
    Path,
    Description,
    ShareState,
    ShareType

$MappedDrives = Get-PSDrive -PSProvider FileSystem | Where-Object {
    $_.DisplayRoot -ne $null
} | Select-Object `
    Name,
    Root,
    DisplayRoot,
    Description

$NetUseOutput = cmd /c "net use"

# -----------------------------
# 10. BitLocker / endpoint encryption
# -----------------------------

$BitLockerStatus = Get-BitLockerVolume | Select-Object `
    MountPoint,
    VolumeStatus,
    ProtectionStatus,
    EncryptionMethod,
    EncryptionPercentage,
    LockStatus

# -----------------------------
# 11. Antivirus / Defender status
# -----------------------------

$DefenderStatus = Get-MpComputerStatus | Select-Object `
    AMServiceEnabled,
    AntivirusEnabled,
    AntispywareEnabled,
    RealTimeProtectionEnabled,
    BehaviorMonitorEnabled,
    IoavProtectionEnabled,
    NISEnabled,
    AntivirusSignatureLastUpdated,
    QuickScanEndTime,
    FullScanEndTime

# -----------------------------
# 12. Common security / IT agents
# -----------------------------

$AgentRegex = "CrowdStrike|Falcon|Sophos|Forti|Defender|Sentinel|ManageEngine|Endpoint|EDR|XDR|Trellix|McAfee|Symantec|Carbon|Qualys|Nessus|Rapid7|Ivanti|SCCM|MECM|Intune"

$SecurityAndITAgents = Get-Service | Where-Object {
    $_.DisplayName -match $AgentRegex -or $_.Name -match $AgentRegex
} | Select-Object `
    Name,
    DisplayName,
    Status,
    StartType

# -----------------------------
# 13. Installed software
# -----------------------------

$InstalledSoftware = $null

if (-not $SkipInstalledSoftware) {
    $UninstallPaths = @(
        "HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*",
        "HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*",
        "HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*"
    )

    $InstalledSoftware = foreach ($Path in $UninstallPaths) {
        Get-ItemProperty $Path | Where-Object {
            $_.DisplayName
        } | Select-Object `
            DisplayName,
            DisplayVersion,
            Publisher,
            InstallDate
    }

    $InstalledSoftware = $InstalledSoftware | Sort-Object DisplayName -Unique
}

# -----------------------------
# 14. Outlook profile check
# -----------------------------

$OutlookProfiles = Get-ChildItem "HKCU:\Software\Microsoft\Office" -Recurse | Where-Object {
    $_.Name -match "Outlook\\Profiles"
} | Select-Object `
    Name

# -----------------------------
# 15. Windows activation / license status
# -----------------------------

$WindowsLicense = Get-CimInstance SoftwareLicensingProduct | Where-Object {
    $_.PartialProductKey -and $_.Name -match "Windows"
} | Select-Object `
    Name,
    Description,
    LicenseStatus,
    PartialProductKey

# -----------------------------
# 16. Recent boot and uptime
# -----------------------------

$LastBoot = $OS.LastBootUpTime
$Uptime = New-TimeSpan -Start $LastBoot -End (Get-Date)

$UptimeInfo = [PSCustomObject]@{
    LastBootTime = $LastBoot
    UptimeDays   = $Uptime.Days
    UptimeHours  = $Uptime.Hours
    UptimeMins   = $Uptime.Minutes
}

# -----------------------------
# 17. Export TXT summary
# -----------------------------

"System Installation Information Collection" | Out-File $SummaryTxt -Encoding UTF8
"Generated On: $(Get-Date)" | Out-File $SummaryTxt -Append -Encoding UTF8
"Computer Name: $ComputerName" | Out-File $SummaryTxt -Append -Encoding UTF8
"Read-only script. No changes made." | Out-File $SummaryTxt -Append -Encoding UTF8

Write-Section "Collection Info" $CollectionInfo
Write-Section "Computer System" $ComputerSystem
Write-Section "BIOS / Serial Number" $BIOS
Write-Section "Operating System" $OS
Write-Section "Domain and Hostname" $DomainInfo
Write-Section "Network Configuration" $NetworkConfig
Write-Section "IP Adapter Details" $IPAdapterDetails
Write-Section "Disk Volumes and File System" $DiskVolumes
Write-Section "Physical Disks" $PhysicalDisks
Write-Section "Firewall Profiles" $FirewallProfiles
Write-Section "Local Users" $LocalUsers
Write-Section "Local Administrators" $LocalAdministrators
Write-Section "Printers" $Printers
Write-Section "Local Shares" $LocalShares
Write-Section "Mapped Drives" $MappedDrives
Write-Section "Net Use Output" $NetUseOutput
Write-Section "BitLocker / Encryption Status" $BitLockerStatus
Write-Section "Microsoft Defender Status" $DefenderStatus
Write-Section "Security and IT Agents" $SecurityAndITAgents
Write-Section "Outlook Profiles" $OutlookProfiles
Write-Section "Windows License" $WindowsLicense
Write-Section "Uptime Information" $UptimeInfo

if (-not $SkipInstalledSoftware) {
    Write-Section "Installed Software" $InstalledSoftware
}

# -----------------------------
# 18. Export CSV files
# -----------------------------

Export-SafeCsv "CollectionInfo" $CollectionInfo
Export-SafeCsv "ComputerSystem" $ComputerSystem
Export-SafeCsv "BIOS" $BIOS
Export-SafeCsv "OperatingSystem" $OS
Export-SafeCsv "DomainInfo" $DomainInfo
Export-SafeCsv "NetworkConfig" $NetworkConfig
Export-SafeCsv "IPAdapterDetails" $IPAdapterDetails
Export-SafeCsv "DiskVolumes" $DiskVolumes
Export-SafeCsv "PhysicalDisks" $PhysicalDisks
Export-SafeCsv "FirewallProfiles" $FirewallProfiles
Export-SafeCsv "LocalUsers" $LocalUsers
Export-SafeCsv "LocalAdministrators" $LocalAdministrators
Export-SafeCsv "Printers" $Printers
Export-SafeCsv "LocalShares" $LocalShares
Export-SafeCsv "MappedDrives" $MappedDrives
Export-SafeCsv "BitLockerStatus" $BitLockerStatus
Export-SafeCsv "DefenderStatus" $DefenderStatus
Export-SafeCsv "SecurityAndITAgents" $SecurityAndITAgents
Export-SafeCsv "OutlookProfiles" $OutlookProfiles
Export-SafeCsv "WindowsLicense" $WindowsLicense
Export-SafeCsv "UptimeInfo" $UptimeInfo

if (-not $SkipInstalledSoftware) {
    Export-SafeCsv "InstalledSoftware" $InstalledSoftware
}

# -----------------------------
# 19. Build HTML report
# -----------------------------

$HtmlSections = @()

$HtmlSections += ConvertTo-SafeHtmlSection "Collection Info" $CollectionInfo
$HtmlSections += ConvertTo-SafeHtmlSection "Computer System" $ComputerSystem
$HtmlSections += ConvertTo-SafeHtmlSection "BIOS / Serial Number" $BIOS
$HtmlSections += ConvertTo-SafeHtmlSection "Operating System" $OS
$HtmlSections += ConvertTo-SafeHtmlSection "Domain and Hostname" $DomainInfo
$HtmlSections += ConvertTo-SafeHtmlSection "Network Configuration" $NetworkConfig
$HtmlSections += ConvertTo-SafeHtmlSection "IP Adapter Details" $IPAdapterDetails
$HtmlSections += ConvertTo-SafeHtmlSection "Disk Volumes and File System" $DiskVolumes
$HtmlSections += ConvertTo-SafeHtmlSection "Physical Disks" $PhysicalDisks
$HtmlSections += ConvertTo-SafeHtmlSection "Firewall Profiles" $FirewallProfiles
$HtmlSections += ConvertTo-SafeHtmlSection "Local Users" $LocalUsers
$HtmlSections += ConvertTo-SafeHtmlSection "Local Administrators" $LocalAdministrators
$HtmlSections += ConvertTo-SafeHtmlSection "Printers" $Printers
$HtmlSections += ConvertTo-SafeHtmlSection "Local Shares" $LocalShares
$HtmlSections += ConvertTo-SafeHtmlSection "Mapped Drives" $MappedDrives
$HtmlSections += ConvertTo-SafeHtmlSection "BitLocker / Encryption Status" $BitLockerStatus
$HtmlSections += ConvertTo-SafeHtmlSection "Microsoft Defender Status" $DefenderStatus
$HtmlSections += ConvertTo-SafeHtmlSection "Security and IT Agents" $SecurityAndITAgents
$HtmlSections += ConvertTo-SafeHtmlSection "Outlook Profiles" $OutlookProfiles
$HtmlSections += ConvertTo-SafeHtmlSection "Windows License" $WindowsLicense
$HtmlSections += ConvertTo-SafeHtmlSection "Uptime Information" $UptimeInfo

if (-not $SkipInstalledSoftware) {
    $HtmlSections += ConvertTo-SafeHtmlSection "Installed Software" $InstalledSoftware
}

$HtmlStyle = @"
<style>
body {
    font-family: Segoe UI, Arial, sans-serif;
    font-size: 13px;
    margin: 20px;
    background-color: #ffffff;
}
h1 {
    color: #1f4e79;
}
h2 {
    color: #2f5597;
    border-bottom: 1px solid #cccccc;
    padding-bottom: 4px;
}
table {
    border-collapse: collapse;
    width: 100%;
    margin-bottom: 25px;
}
th {
    background-color: #d9eaf7;
    border: 1px solid #999999;
    padding: 6px;
    text-align: left;
}
td {
    border: 1px solid #cccccc;
    padding: 6px;
}
p {
    color: #666666;
}
</style>
"@

$HtmlBody = @"
<html>
<head>
<title>System Installation Report - $ComputerName</title>
$HtmlStyle
</head>
<body>
<h1>System Installation Report</h1>
<p><b>Computer:</b> $ComputerName</p>
<p><b>Collected By:</b> $env:USERDOMAIN\$env:USERNAME</p>
<p><b>Generated On:</b> $(Get-Date)</p>
<p><b>Impact:</b> Read-only collection. No configuration changes made.</p>
$($HtmlSections -join "`n")
</body>
</html>
"@

$HtmlBody | Out-File $HtmlReport -Encoding UTF8

# -----------------------------
# 20. Final output
# -----------------------------

Write-Host ""
Write-Host "Collection completed successfully." -ForegroundColor Green
Write-Host "No changes were made to this system." -ForegroundColor Green
Write-Host ""
Write-Host "Report Folder : $ReportFolder"
Write-Host "HTML Report   : $HtmlReport"
Write-Host "TXT Summary   : $SummaryTxt"
Write-Host ""