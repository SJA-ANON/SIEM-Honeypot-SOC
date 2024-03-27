![](IMAGES/Honeypot.png)
# Honeypot & SIEM | Microsoft Azure Sentinel Attack Map

### Summary
SIEM stands for Security Information and Event Management System. It is a solution that helps organizations detect, analyze, and respond to security threats before they harm business operations. It is a tool that collects event log data from a range of sources within a network such as Firewalls, IDS/IPS, Identity solutions, etc. This allows the security professionals to monitor, prioritize and remediate potential threats in real-time. A honeypot is a security mechanism that creates a virtual trap in a controlled and safe environment to lure attackers. An intentionally compromised computer system to study how attackers work, examine different types of threats and improve security policies. This lab's purpose is to understand how to collect honeypot attack log data, query it in a SIEM and display it in a manner that is easy to understand such data. In this case it will be displayed in a world map by event count and geolocation.

### Learning Objectives:
- Configuration & Deployment of Azure resources such as virtual machines, Log Analytics Workspaces, and Azure Sentinel
- Hands-on experience and working knowledge of a SIEM Log Management Tool (Microsoft's Azure Sentinel)
- Understand Windows Security Event logs
- Utilization of KQL to query logs
- Display attack data on a dashboard with Workbooks (World Map)

### Tools & Requirements:

1. Microsoft Azure Subscription 
2. Azure Sentinel
3. Kusto Query Language (KQL - Used to build world map)
4. Network Security Groups (Layer 4/3 Firewall in Azure)
5. Remote Desktop Protocol (RDP)
6. 3rd Party API: [ipgeolocation.io](https://ipgeolocation.io/)
7. Custom [Powershell Script](https://github.com/joshmadakor1/Sentinel-Lab/blob/main/Custom_Security_Log_Exporter.ps1) written by Josh Madakor



### Overview:
![](IMAGES/SIEM%20Lab.png)

## Step 1: Create a Microsoft Azure Subscription: [Azure](https://azure.microsoft.com/en-us/free/)
> Free 200$ Credit for 30 days

![](IMAGES/Azure_Subscription.png)

## Step 2: Create a Honeypot Virtual Machine
> Exposed Windows VM

![](IMAGES/VM_Creation.png)

### Basics

- Go to `portal.azure.com`
- Search for "virtual machines" 
- Create > Azure virtual machine
#### Project details
- Create new resource group and name it (Honeypot-Lab)
> A resource group is a collection of resources that share the same lifecycle, permissions, and policies.
#### Instance details
- Name your VM (honeypot-vm)
- Select a recommended region ((Asia Pacific) Central India)
- Availability options: No infrastructure redundancy required
- Security type: Standard
- Image: Windows 10 Pro, version 22H2 - x64 Gen2
- VM Architecture: x64
- Size: Default is okay (Standard_D2s_v3 – 2vcpus, 8 GiB memory)
#### Administrator account
- Create a username and password for virtual machine
> IMPORTANT NOTE: These credentials will be used to log into the virtual machine (Keep them handy)
#### Inbound port rules
- Public inbound ports: Allow RDP (3389)
#### Licensing
- Confirm licensing 
- Select **Next : Disks >**

![](IMAGES/VM_Creation01.png)

### Disks 
- Leave all defaults
- Select **Next : Networking >**

### Networking
#### Network interface
- NIC network security group: Advanced > Create new
> A network security group contains security rules that allow or deny inbound network traffic to, or outbound network traffic from, the virtual machine. In other words, security rules management.
- Remove Inbound rules (1000: default-allow-rdp) by clicking three dots
- Add an inbound rule
- Destination port ranges: * (wildcard for anything)
- Protocol: Any
- Action: Allow
- Priority: 100 (low)
- Name: Anything (ALLOW_ANY_IN)
- Select **Review + create**

![](IMAGES/inbound_any.png)

> Configuring the firewall to allow traffic from anywhere will make the VM easily discoverable.

## Step 3: Create a Log Analytics Workspace
- Search for "Log analytics workspaces"
- Select **Create Log Analytics workspace**
- Put it in the same resource group as VM (Honeypot-Lab)
- Give it a desired name (law-honeypot)
- Add to same region (Central India)
- Select **Review + create**

![](IMAGES/Log_Analytics_Workspace.png)

> The Windows Event Viewer logs will be ingested into Log Analytics workspaces in addition to custom logs with geographic data to map attacker locations.

## Step 4: Configure Microsoft Defender for Cloud
- Search for "Microsoft Defender for Cloud"
- Scroll down to "Environment settings" > subscription name > log analytics workspace name (law-honeypot)

![](IMAGES/Defender_azure.png)

#### Settings | Defender plans
- Cloud Security Posture Management: ON
- Servers: ON
- SQL servers on machines: OFF
- Hit **Save**

#### Settings | Data collection
- Select "All Events" 
- Hit **Save**

![](IMAGES/Enable_all_events.png)

## Step 5: Connect Log Analytics Workspace to Virtual Machine
- Search for "Log Analytics workspaces"
- Select workspace name (law-honeypot) > "Virtual machines" > virtual machine name (honeypot-vm)
- Click **Connect**

![](IMAGES/Law_VM_Connect.png)

## Step 6: Configure Microsoft Sentinel
- Search for "Microsoft Sentinel"
- Click **Create Microsoft Sentinel**
- Select Log Analytics workspace name (law-honeypot)
- Click **Add**

![](IMAGES/Add_Sentinel.png)

## Step 7: Disable the Firewall in Virtual Machine
- Go to Virtual Machines and find the honeypot VM (honeypot-vm)
- By clicking on the VM copy the IP address
- Log into the VM via Remote Desktop Protocol (RDP) with credentials from step 2
- Accept Certificate warning
- Select NO for all **Choose privacy settings for your device**
- Click **Start** and search for "wf.msc" (Windows Defender Firewall)
- Click "Windows Defender Firewall Properties"
- Turn Firewall State OFF for **Domain Profile** **Private Profile** and **Public Profile**
- Hit **Apply** and **Ok**
- Ping VM via Host's command line to make sure it is reachable `ping -t <VM IP>`

![](IMAGES/defender_off.png)

## Step 8: Scripting the Security Log Exporter
- In VM open Powershell ISE
- Set up Edge without signing in
- Copy [Powershell script](https://github.com/joshmadakor1/Sentinel-Lab/blob/main/Custom_Security_Log_Exporter.ps1) into VM's Powershell (Written by Josh Madakor)
- Select **New Script** in Powershell ISE and paste script
- Save to Desktop and give it a name (Log_Exporter)

![](IMAGES/Script_runs.PNG)

- Make an account with [Free IP Geolocation API and Accurate IP Lookup Database](https://ipgeolocation.io/)
> This account is free for 1000 API calls per day. Paying 15.00$ will allow 150,000 API calls per month.
- Copy API key once logged in and paste into script line 2: `$API_KEY = "<API key>"`
- Hit **Save**
- Run the PowerShell ISE script (Green play button) in the virtual machine to continuously produce log data

![](IMAGES/IP_geoloc.png)

> The script will export data from the Windows Event Viewer to then import into the IP Geolocation service. It will then extract the latitude and longitude and then create a new log called failed_rdp.log in the following location: C:\ProgramData\failed_rdp.log

## Step 9: Create Custom Log in Log Analytics Workspace
- Create a custom log to import the additional data from the IP Geolocation service into Azure Sentinel
- Search "Run" in VM and type "C:\ProgramData"
- Open file named "failed_rdp" hit **CTRL + A** to select all and **CTRL + C** to copy selection
- Open notepad on Host PC and paste contents
- Save to desktop as "failed_rdp.log"
- In Azure go to Log Analytics Workspaces > Log Analytics workspace name (honeypot-log) > Custom logs > **Add custom log**
#### Sample
- Select Sample log saved to Desktop (failed_rdp.log) and hit **Next**
#### Record delimiter
- Review sample logs in Record delimiter and hit **Next** 
#### Collection paths
- Type > Windows
- Path > "C:\ProgramData\failed_rdp.log"
#### Details
- Give the custom log a name and provide description (FAILED_RDP_WITH_GEO) and hit **Next**
- Hit **Create**

![](IMAGES/Custom_logs.png)

## Step 10: Query the Custom Log
- In Log Analytics Workspaces go to the created workspace (honeypot-log) > Logs
- Run a query to see the available data (FAILED_RDP_WITH_GEO_CL)
> May take some time for Azure to sync VM and Log Analytics

![](IMAGES/Working_CL.png)

## Step 11: Mapping the Data in Microsoft Sentinel

1. Navigate to Microsoft Sentinel > Workbooks > Add workbook
2. Edit the workbook and remove the default widgets
3. Add a new query and paste the KQL query below:
 
```
FAILED_RDP_WITH_GEO_CL | extend username = extract(@"username:([^,]+)", 1, RawData),
         timestamp = extract(@"timestamp:([^,]+)", 1, RawData),
         latitude = extract(@"latitude:([^,]+)", 1, RawData),
         longitude = extract(@"longitude:([^,]+)", 1, RawData),
         sourcehost = extract(@"sourcehost:([^,]+)", 1, RawData),
         state = extract(@"state:([^,]+)", 1, RawData),
         label = extract(@"label:([^,]+)", 1, RawData),
         destination = extract(@"destinationhost:([^,]+)", 1, RawData),
         country = extract(@"country:([^,]+)", 1, RawData)
| where destination != "samplehost"
| where sourcehost != ""
| summarize event_count=count() by latitude, longitude, sourcehost, label, destination, country 
```
4. Run the Query!
5. You can continue refreshing the map to display more failed RDP attacks. Here the honey pot has was ran over night.
![Alt text](IMAGES/map.png)

## Wrapping up

I hope you have enjoyed going through the lab, as much as I did! Don't forget to
deprovision your Azure services when you have decided to stop gathering failed
RDP attacks so you don't get charged!
