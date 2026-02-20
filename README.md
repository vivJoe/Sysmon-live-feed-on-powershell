Real-Time Sysmon Monitoring with PowerShell — No SIEM Required

A practical guide to setting up live endpoint visibility on a home lab using Sysmon and a custom PowerShell monitoring script — with colour-coded alerts, noise filtering, and zero cost.
Table of Contents
Introduction & Motivation
What is Sysmon?
Prerequisites
Step 1 — Installing Sysmon
Step 2 — Applying a Sysmon XML Configuration
Step 3 — Viewing Events in Windows Event Viewer
Step 4 — Live Monitoring with PowerShell

Understanding the Script
Sysmon Event IDs — Quick Reference
Why This Matters
Next Steps & Improvements

Introduction
When most people think about endpoint monitoring and detection, they immediately think of a SIEM — Splunk, Microsoft Sentinel, Elastic, and so on. These are powerful platforms, but for a home lab or a small environment, they can be overkill, expensive, or simply inaccessible.
This write-up documents a lightweight alternative: using Sysmon (System Monitor) combined with a custom PowerShell script to achieve live, colour-coded, filtered endpoint visibility — directly in your terminal.
No SIEM. No forwarding agent. No licensing cost.
The result is a monitoring setup that:

Highlights network connections in Cyan
Highlights file creation events in Yellow
Filters out noise so you only see what matters
Updates every 2 seconds in real time
What is Sysmon?
Sysmon (System Monitor) is a free Windows system service and device driver from Microsoft's Sysinternals suite. Once installed, it stays resident across system reboots and logs detailed information about process creations, network connections, file creation timestamps, registry changes, and more — all to the Windows Event Log.
It is widely used by blue teamers, threat hunters, and SOC analysts as a primary source of telemetry on Windows endpoints.
Key things Sysmon logs (by Event ID):
Event IDDescription1Process Creation3Network Connection5Process Terminated11File Created12/13Registry Events22DNS QueryPrerequisites
Before starting, make sure you have:

A Windows machine (Windows 10/11 or Windows Server)
Administrator privileges
PowerShell (built-in on all modern Windows systems)
Internet access to download Sysmon
Step 1 — Installing Sysmon
Download Sysmon from the official Microsoft Sysinternals page:

👉 https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
Extract the downloaded ZIP file. You will find Sysmon64.exe (for 64-bit systems).
Open PowerShell as Administrator and navigate to the folder where you extracted Sysmon:
cd C:\Path\To\Sysmon
Install Sysmon with a basic configuration (we will apply a proper XML config in the next step):
.\Sysmon64.exe -accepteula -i
Sysmon is now installed as a Windows service and will start logging events immediately.
To confirm it is running:

Get-Service Sysmon64
You should see a Running status.
Step 2 — Applying a Sysmon XML Configuration
Out of the box, Sysmon logs a lot — including significant noise. A configuration XML file tells Sysmon exactly what to log and what to ignore.
A popular and well-maintained configuration is the SwiftOnSecurity Sysmon config, which is a great starting point:

👉 https://github.com/SwiftOnSecurity/sysmon-config
Downloading and applying the config
Download the sysmonconfig-export.xml file from the repository above (or create your own).
Apply it to your running Sysmon installation:
.\Sysmon64.exe -c C:\Path\To\sysmonconfig-export.xml
Confirm the config was applied:
.\Sysmon64.exe -c
This will print the active configuration.

Why this matters: Without a good config, Sysmon will log thousands of benign events per minute. The XML config allows you to include or exclude events based on process names, paths, IP addresses, and other fields — giving you signal, not noise.
Step 3 — Viewing Events in Windows Event Viewer
Once Sysmon is running, you can explore the raw events through the Windows built-in Event Viewer.

Open Event Viewer: press Win + R, type eventvwr.msc, and press Enter.
In the left panel, navigate to:

Applications and Services Logs
└── Microsoft
    └── Windows
        └── Sysmon
            └── Operational
You will see a list of Sysmon events. Click any event to read its details in the bottom pane.
Limitations of Event Viewer
While useful for ad-hoc lookups, Event Viewer has some friction for live monitoring:

It does not auto-refresh — you must manually press F5 or click Refresh
There is no colour coding to quickly distinguish event types
Filtering is possible but clunky for real-time use
This is exactly why the PowerShell approach below is more practical for active monitoring.
Step 4 — Live Monitoring with PowerShell
The following script polls the Sysmon event log every 2 seconds and prints new events to the terminal — with colour coding by event type.

The Script
Save this as SysmonMonitor.ps1:

$lastTime = Get-Date
Write-Host "Monitoring started. Watching for Network (Cyan) and Files (Yellow)..." -ForegroundColor Green

while ($true) {
    # 1. Grab new events since the last check
    $events = Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" | 
              Where-Object { $_.TimeCreated -gt $lastTime } | 
              Select-Object TimeCreated, Id, Message, @{Name="Details"; Expression={$_.Message}}

    if ($events) {
        foreach ($e in $events) {
            # 2. Check the ID and apply colours
            if ($e.Id -eq 3) { 
                Write-Host "`n[!] NETWORK CONNECTION DETECTED" -ForegroundColor Cyan -BackgroundColor Black
                $e | Format-List TimeCreated, Id, Message
            }
            elseif ($e.Id -eq 11) { 
                Write-Host "`n[+] FILE CREATION DETECTED" -ForegroundColor Yellow -BackgroundColor Black
                $e | Format-List TimeCreated, Id, Message
            }
            else {
                # Standard output for everything else (Process creation, etc.)
                $e | Format-List TimeCreated, Id, Message
            }
        }
    }

    # 3. Update the timestamp and wait
    $lastTime = Get-Date
    Start-Sleep -Seconds 2
}
Running the Script
Open PowerShell as Administrator and run:

Set-ExecutionPolicy RemoteSigned -Scope CurrentUser   # Only needed once
.\SysmonMonitor.ps1
You will immediately see:

Monitoring started. Watching for Network (Cyan) and Files (Yellow)...
From this point, any new Sysmon events will print to the screen in real time.
Understanding the Script
Let's break down what the script does, section by section.

Initialising the Time Baseline
$lastTime = Get-Date
This captures the current timestamp when the script starts. It is used as the baseline — only events that occur after this moment will be displayed. This prevents the script from flooding your terminal with historical events on startup.

The Polling Loop
while ($true) {
    ...
    Start-Sleep -Seconds 2
}
The script runs in an infinite loop, checking for new events every 2 seconds. You can adjust this interval — lower values give you faster updates but use slightly more CPU; higher values reduce load but introduce latency.

Querying the Sysmon Log
$events = Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" | 
          Where-Object { $_.TimeCreated -gt $lastTime }
Get-WinEvent reads directly from the Windows Event Log. The Where-Object filter ensures we only pull events that are newer than our last check — this is what makes the monitoring incremental rather than retrieving the full log every loop.

Colour-Coded Output by Event ID
if ($e.Id -eq 3) { 
    Write-Host "`n[!] NETWORK CONNECTION DETECTED" -ForegroundColor Cyan -BackgroundColor Black
}
elseif ($e.Id -eq 11) { 
    Write-Host "`n[+] FILE CREATION DETECTED" -ForegroundColor Yellow -BackgroundColor Black
}

This is the core of the visibility improvement. The script checks the Event ID of each log entry and applies a different colour:
ColourEventWhat it means🔵 CyanEvent ID 3A process established a network connection🟡 YellowEvent ID 11A file was created on diskDefaultEverything elseProcess creation, DNS queries, registry events, etc.Updating the Timestamp
$lastTime = Get-Date
At the end of each loop, the timestamp is updated so the next iteration only picks up events newer than the current check — preventing duplicate entries.
Sysmon Event IDs — Quick Reference
You can extend the script to handle additional event types by adding more elseif blocks. Here are the most commonly used IDs to build on:
Event IDNameDetection Value1Process CreateDetect suspicious process launches3Network ConnectionDetect C2 callbacks, lateral movement5Process TerminateTrack process lifetimes7Image LoadedDetect DLL injection8CreateRemoteThreadClassic injection technique10ProcessAccessLSASS credential dumping11FileCreateDetect dropped payloads12/13RegistryEventDetect persistence mechanisms15FileCreateStreamHashDetect ADS (Alternate Data Streams)22DNSEventDetect DNS-based C2Why This Matters
This setup demonstrates something important for anyone building security skills in a home lab:
You do not need enterprise tooling to develop detection intuitions.
By working directly with Sysmon and PowerShell, you gain:

A deeper understanding of what Windows actually logs and when
Exposure to raw event data before it is abstracted by a SIEM
The ability to write your own detection logic from scratch
A transferable foundation — the same Event IDs, the same log structure, the same telemetry that Splunk, Sentinel, and Elastic ingest
SIEMs are aggregation and visualisation layers. The data underneath them is what you're working with here, unfiltered.

Resources
Sysmon Download — Microsoft Sysinternals
SwiftOnSecurity Sysmon Config
Sysmon Event IDs Reference — UltimateWindowsSecurity
MITRE ATT&CK Framework

⚠️ Disclaimer
This script is intended for educational and lab purposes only. While it provides excellent visibility into system activity, it is not a replacement for an enterprise-grade EDR or SIEM in a production environment. Always run scripts from the internet in a controlled environment first.
Written as part of a home lab security monitoring project. All monitoring was performed on a personal lab environment.

Author

Vivian Iwegbu

Entry-Level SOC Analyst | Cybersecurity Researcher
