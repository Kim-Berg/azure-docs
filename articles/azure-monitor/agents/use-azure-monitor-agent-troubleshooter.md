---
title: Use Azure Monitor Troubleshooter
description: Detailed instructions on using the on agent monitoring tool to diagnose potential issue.
ms.topic: conceptual
author: guywi-ms
ms.author: guywild
ms.date: 4/28/2023
ms.custom: references_regions
ms.reviewer: jeffwo

# customer-intent: As an IT manager, I want to investigate agent issue on a particular virtual machine and determine if I can resolve the issue on my own.
---

# Use the Azure Monitor Agent Troubleshooter
The Azure Monitor Agent isn't a service that runs in the context of an Azure Resource Provider. It might even be running in on-premises machines within a customer network boundary. The Azure Monitor Agent Troubleshooter is designed to help diagnose issues with the agent, and general agent health checks. It can run checks to verify agent installation, connection, general heartbeat, and collect AMA-related logs automatically from the affected Windows or Linux VM. More scenarios will be added over time to increase the number of issues that can be diagnosed.
> [!Note]
> Troubleshooter is a command line executable that's shipped with the agent for all versions newer than **1.12.0.0** for Windows and **1.25.1** for Linux. 
> You can't copy the troubleshooter to a VM to diagnose an older agent version.


## Prerequisites
- Ensure that the AMA agent is installed by looking for the directory C:/Packages/Plugins/Microsoft.Azure.Monitor.AzureMonitorWindowsAgent on the Windows OS and /var/lib/waagent/Microsoft.Azure.Monitor.AzureMonitorLinuxAgent-* on the Linux agent. 
- The linux Troubleshooter requires Python 2.6+ or any Python3 installed on the machine. In addition, the following Python packages are required to run (all should be present on a default install of Python2 or Python3):

|Python Package|	Required for Python2?	|Required for Python3?|
|:---|:---|:---|
|copy|	                               yes|	yes|
|datetime|	               yes|	yes|
|json|	                               yes|	yes|
|os|	                               yes|	yes|
|platform|	               yes|	yes|
|re|	                               yes|	yes|
|requests|	                 no|	yes|
|shutil|	                                yes|	yes|
|subprocess|	                yes|	yes|
|url lib|	                                yes|	no|
|xml.dom.minidom|	yes|	yes|

## Windows Troubleshooter
### Run Windows Troubleshooter
1. Log in to the machine to be diagnosed as Administator
2. Run script in PowerShell
   
   ```PowerShell
   $currentVersion = ((Get-ChildItem -Path "Registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows Azure\HandlerState\" `
    | where Name -like "*AzureMonitorWindowsAgent*" `
    | ForEach-Object {$_ | Get-ItemProperty} `
    | where InstallState -eq "Enabled").PSChildName -split('_'))[1]
   $troubleshooterPath = "C:\Packages\Plugins\Microsoft.Azure.Monitor.AzureMonitorWindowsAgent\$currentVersion\Troubleshooter"
   Start-Process -FilePath $troubleshooterPath\AgentTroubleshooter.exe -ArgumentList "--ama"
   Invoke-Item $troubleshooterPath
   ```
> [!Note]
> Note: If you use the client installer, the troubleshooter path is `C:\Program Files\Azure Monitor Agent\Troubleshooter`.

### Evaluate the Windows Results
The Troubleshooter runs two tests and collects several diagnostic logs.

|Test| Description|
|:---|:---|
|Machine Network Configuration (Configuration) | This test checks the basic network connection including IPV 4 and IPV 6 address resolutions.  If IPV6 isn't available on the machine, you see a warning.|
|Connection to Control Plan (MCS)              | This test checks to see if the agent configuration information can be retrieved from the central data control plan. Controlling information includes which source data to collect and where it should be sent to. All agent configuration is done through Data Collection Rules.|



### Share the Windows Results
The detailed data collected by the troubleshooter include system configuration, network configuration, environment variables, and agent configuration that can aid the customer in finding any issues.  The troubleshooter make is easy to send this data to customer support by creating a Zip file that should be attached to any customer support request. The file is located in C:/Packages/Plugins/Microsoft.Azure.Monitor.AzureMonitorWindowsAgent/{version}/Troubleshooter.  The agent logs can be cryptic but they can give you insight into problems that you might be experiencing.

|Logfile                              | Contents|
|:---|:---|
|Curl.exe                           | results of basic network connectivity for the agent using the Curl command that isn't dependent on any agent software. |
|AgentProcesses           | checks that all the agent processes are running and collects the environment variables that were used for each process. |
|NetworkDiagnositc  | this file has information on the SSL version and certificate information.|
|Table2csv.exe                | snapshot of all the data streams and tables that are configured in the agent along with general information about the time range over which events were seen. |
|ImdsMetadataResponse.json | contains the results of the request for Instance Metadata Service that contains information about the VM on which the agent is running. |
|TroubleshootingLogs | contains a useful table in the Customer Data Statistics section for events that were collected in each local table over different time buckets. |



## Linux Troubleshooter
### Run Linux Troubleshooter
1. Log in to the machine to be diagnosed
2. Go to the location where the troubleshooter is automatically installed: cd /var/lib/waagent/Microsoft.Azure.Monitor.AzureMonitorLinuxAgent-{version}/ama_tst
3. Run the Troubleshooter: sudo sh ama_troubleshooter.sh -A

There are six sections that cover different scenarios that customers have historically had issues with. By enter 1-6 or A, customer is able to diagnose issues with the agent. Adding an L creates a zip file that can be shared if technical support in needed.  

### Evaluate Linux Results
The details for the covered scenarios are below:

|Scenario | Tests|
|:---|:---|
|Agent having installation issues|Supported OS / version, Available disk space, Package manager is available (dpkg/rpm), Submodules are installed successfully, AMA installed properly, Syslog available (rsyslog/syslog-ng), Using newest version of AMA, Syslog user generated successfully|
|Agent doesn't start, can't connect to Log Analytics|AMA parameters set up, AMA DCR created successfully, Connectivity to endpoints, Submodules started, IMDS/HIMDS metadata and MSI tokens available|
|Agent is unhealthy, heartbeat doesn't work properly|Submodule status, Parse error files|
|Agent has high CPU / memory usage|Check logrotate, Monitor CPU/memory usage in 5 minutes (interaction mode only)|
|Agent syslog collection doesn't work properly|Rsyslog / syslog-ng setup and running, Syslog configuration being pulled / used, Syslog socket is accessible|
|Agent custom log collection doesn't work properly|Custom log configuration being pulled / used, Log file paths is valid|

### Share Linux Logs
To create a zip file use this command when running the troubleshooter: sudo sh ama_troubleshooter.sh -A L.  You'll be asked for a file location to create the zip file.

## Frequently asked questions

This section provides answers to common questions.

### How can I confirm that the Log Analytics agent can communicate with Azure Monitor?

From Control Panel on the agent computer, select **Security & Settings** > **Microsoft Monitoring Agent**. Under the **Azure Log Analytics (OMS)** tab, a green check mark icon confirms that the agent can communicate with Azure Monitor. A yellow warning icon means the agent is having issues. One common reason is the **Microsoft Monitoring Agent** service has stopped. Use service control manager to restart the service.

## Next steps
- [Install the Azure Monitor Agent](azure-monitor-agent-manage.md) on Windows and Linux virtual machines.
- [Create a data collection rule](data-collection-rule-azure-monitor-agent.md) to collect data from the agent and send it to Azure Monitor.
