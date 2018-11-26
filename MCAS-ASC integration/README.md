## Integrating Microsoft Cloud App Security with Azure Security Center

#### Summary
This article describes how to integrate Microsoft Cloud App Security (MCAS) with Azure Security Center (ASC). We will leverage the MCAS SIEM agent capability to forward CEF over syslog events to an ASC Log Analytics (LA) workspace. From the ASC workspace we can create custom alerts and invoke Logic App playbooks for any custom action, like remediation actions or creating a ticket in a ticketing system.

#### Assumptions
This blogpost is assuming that you have the following in place: <br>
*  A configured MCAS environment, if you don't have a MCAS environment, go <a href="https://docs.microsoft.com/en-us/cloud-app-security/getting-started-with-cloud-app-security" target="_blank">here</a>
* A configured ASC environment, if you don't have an ASC environment, go <a href="https://docs.microsoft.com/en-us/azure/security-center/security-center-get-started" target="_blank">here</a>
 
#### High-level steps
The scenario we want to achieve is at a high-level captured in the following figure: <br>

![alt text](https://github.com/tianderturpijn/MCAS/blob/master/MCAS-ASC%20integration/screenshots/highlevel_overview.png)
<br>
<br>

To configure the MCAS integration with ASC we are going to perform the following steps.
1.	Create a new VM in Azure, running Ubuntu 18.04 LTS, we will refer to this VM as the MCAS-SIEM-MMA VM
Note: you can also leverage a Windows VM, but this blog leverages Linux as an example
2.	Configure the SIEM agent in MCAS
3.	Download and install Java 8
4.	Install and configure the MCAS SIEM agent
5.	Install and configure the Microsoft Management Agent (MMA)
6.	Configure the MMA to forward MCAS alert and activities to Log Analytics
7.	Create a MCAS alert and verify it appears in the ASC Log Analytics workspace
8.	Create a custom Azure Security Center alert

### Step 1 – Create the SIEM Agent VM in Azure
In this section we are going to create a new Azure Linux VM and configure the prerequisites.
1.	Open the Azure Portal and create a new Ubuntu Server 18.04 LTS VM, using a SSH public key is a security best practice:

![alt text](https://github.com/tianderturpijn/MCAS/blob/master/MCAS-ASC%20integration/screenshots/create_VM.png)
<br>
<br>
