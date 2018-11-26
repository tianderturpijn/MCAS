## Integrating Microsoft Cloud App Security with Azure Security Center

#### Summary
This article describes how to integrate Microsoft Cloud App Security (MCAS) with Azure Security Center (ASC). We will leverage the MCAS SIEM agent capability to forward CEF over syslog events to an ASC Log Analytics (LA) workspace. From the ASC workspace we can create custom alerts and invoke Logic App playbooks for any custom action, like remediation actions or creating a ticket in a ticketing system.

#### Assumptions
This blogpost is assuming that you have the following in place: <br>
*  A configured MCAS environment, if you don't have a MCAS environment, go <a href="https://docs.microsoft.com/en-us/cloud-app-security/getting-started-with-cloud-app-security" target="_blank">here</a>
* A configured ASC environment, if you don't have an ASC environment, go <a href="https://docs.microsoft.com/en-us/azure/security-center/security-center-get-started" target="_blank">here</a>
 
#### High-level steps
The scenario we want to achieve is at a high-level captured in the following figure: <br>

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/highlevel_overview.png)
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

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/create_vm.png)
<br>
<br>

2.	Provide a name and select your hardware configuration
a.	Please note that for a production environment the following is recommended for the MCAS SIEM agent VM:<br>
* CPU: 2 <br>
* Disk space: 20 GB <br>
* RAM: 2 GB <br>
* The server must be running Java 8. Earlier versions are not supported. <br>
* Set your firewall as described in <a href="https://docs.microsoft.com/en-us/cloud-app-security/network-requirements" target="_blank">network requirements</a>

3.	Configure the remainder of the Azure create VM wizard as you see fit, but make sure to add a public port for SSH (port 22) <br>
![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/ssh_port.png)
<br>
<br>

4. When the VM has been deployed, make sure to make a note of the VM’s public IP address, which can be found under the VM overview settings: <br>
![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/public_ip.png)
<br>
<br>

5.	Since we are going to send CEF over syslog data to this VM, we need to add an NSG rule which allows UDP traffic over port 514. Configure the source and destination for better locked down security.
![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/inbound_rules.png)
<br>
<br>

### Step 2 - Configure the SIEM agent in MCAS
For a full description how to configure the MCAS SIEM agent, please go here 
1.	Open the Cloud App Security portal and click on **Settings** -> **Security extensions**
![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/security_extensions.png)
<br>
<br>

2.	In the SIEM agents tab, click on the **"+"** sign in the upper right corner <br> 
![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/siem_agent.png)
<br>
<br>

3.	Click on **Start Wizard** to start the SIEM agent configuration wizard
4.	Provide an Agent Name, select Generic CEF as the SIEM format and make sure to select **RFC 3164, include PRI, include system name**
a.	These settings ensure the usage of the correct CEF schema, priority and sending hostname (for easier querying) <br>
![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/configure_siem_agent.png)
<br>

Click on **Next**

5.	Fill in the **Remote Syslog host** field by providing the public IP address of the Linux VM you have created in “Create the SIEM Agent VM in Azure”, step 4.
In our example we are using port 514 as the **remote syslog port** and UDP as the **remote syslog protocol**, but this can be any port and either TCP or UDP.

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/configure_siem_agent2.png)
<br>
Click on **Next** <br>

6.	Under **Data Types**, select which data you would like to send to the SIEM agent. In our example we are going to send **all alerts** and **all activities**.
![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/configure_siem_agent3.png)
<br>

Click on **Next** <br>

7.	A token will be generated, and you will see a “congratulations!” confirmation. 
**Make sure you copy the token to your clipboard**. <br>
![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/configure_siem_agent3.png)
<br>

Click on **Close** <br>

8.	You should now see a new SIEM agent entry, notice the **Type** and **Status** <br>
![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/configure_siem_agent4.png)
<br>

### Step 3 - Download and install Java 8
In case your MCAS-SIEM-MMA VM does not have Java 8 installed, we need to download and install Java 8 before continuing. <br>
**Note:**
*If you are using RedHat, you probably already have that configured. There are multiple ways to install Java 8, so you might have your own preferred way.*

1.	Use your favorite SSH remoting tool to logon to the Linux VM
2.	Update the Java repo by executing:
**sudo add-apt-repository ppa:webupd8team/java** <br>

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/add_java.png)
<br>

3.	Click on **ENTER** to continue
4.	Let’s make sure that we’re up to date
Run the following command: **sudo apt-get update**
5.	Install Java 8 by executing:
**sudo apt-get install oracle-java8-installer** <br>

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/add_java2.png)
<br>

6.	Click on **Y**
7.	Click on **OK** to agree and accept the terms
![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/add_java3.png)
<br>

8.	After completion, we are going to set Java8 as the default by running:
**sudo apt-get install oracle-java8-set-default** <br>

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/add_java4.png)
<br>

### Step 4 - Install and configure the MCAS SIEM agent
In this section we are going to install and configure the MCAS SIEM agent on the MCAS-SIEM-MMA VM you have created in step 1
1.	Download the MCAS SIEM agent from the Microsoft Download Center, this will be a Java **AR**chive file (JAR), and save it locally

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/mcas_siem.png)
<br>

2.	Switch back to your SSH session from step 3
3.	Upload the JAR file you have downloaded to your MCAS-SIEM-MMA VM
4.	Execute the following command within the folder where you have uploaded the JAR file <br>
**java -jar mcas-siemagent-0.87.20-signed.jar [--logsDirectory DIRNAME] [--proxy ADDRESS[:PORT]] --token TOKEN**

Please note the following:
a.	Update the JAR filename to match with what you have downloaded
b.	The parameter **--logsDirectory** is an optional parameter in case you want to store the MCAS logs in a specific folder
c.	The **--proxy** parameter is required if you use a proxy server
d.	The **--token** parameter must be passed using the token you have saved under **Step 2 - Configure the SIEM agent in MCAS** (7. A token will be generated)
e.	If you are using a specific folder you might need elevated privileges to have write access 
5.	In my example I’m using <br>
**java -jar mcas-siemagent-0.111.126-signed.jar --logsDirectory /var/log/mcas-logs --token <myToken>**

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/token.png)
<br>

6.	The MCAS SIEM agent is now running, let’s verify the same at the MCAS portal, switch to your MCAS Portal, Settings -> Security Extensions -> SIEM Agents. You should see **Connected** under the **Status** column
![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/siem_status.png)
<br>

### Step 5 - Install and configure the Microsoft Management Agent
Now that we have installed and configured the MCAS SIEM agent, it’s time to install the Microsoft Management Agent (MMA) which will send the MCAS alerts and activities to the Azure Security Center (ASC) Log Analytics workspace.
Before we can start the installation of the MMA, we need to know which workspace ASC is using.
Note: skip the following steps if you already know which Log Analytics workspace ASC is using

1.	Navigate to the ASC blade in the Azure Portal
2.	Under **Resource Security Hygiene**, click on **Security solutions**

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/asc_solutions.png)
<br>

3.	Look for **CEF** and click on **Add**
![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/cef.png)
<br>

4.	Under the steps that are listed, click on **Download & install Agent for Linux** <br>
![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/cef_logs.png)
<br>

5.	Under **Add new non-Azure computers**, select the workspace that you want to leverage for the MCAS integration and click on **Add Computers**
Note: you might see just one or multiple Log Analytics workspaces <br>
![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/add_non_azure_computers.png)
<br>

6.	If you click on **Add Computers** another blade opens which will provide you a command line to install the Direct Agent on Linux which includes the workspaceID and workspace key
Click on the **Click to copy** button and save this command line <br>
![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/direct_agent.png)
<br>

7.	Switch to your SSH session connected to the MCAS-SIEM-MMA VM
8.	Copy and paste the **wget** command line from the previous step in your SSH session and execute it.
Note: the MMA installation requires elevated privileges. In case you need to troubleshoot the Linux agent, refer to this link
9.	If the MMA installation was successful you should see your new onboarded Linux VM showing up in Log Analytics within a few minutes. You can leverage Log Search in Log Analytics to verify:

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/heartbeat.png)
<br>

### Step 6 - Configure the MMA to forward MCAS alerts and activities to Log Analytics
In this section we will configure the MMA to forward MCAS alerts and activities to Log Analytics, leveraging syslog forwarding. We will first make sure that local syslog entries are being sent to Log Analytics and then we will configure syslog forwarding, so that incoming MCAS events, received by the MCAS SIEM agent, will be forwarded as well, which is disabled by default on Linux. <br>

**Note** <br>
In this blog and section we will configure the Linux facilities to be forwarded to Log Analytics locally on the VM. Please note that this can be configured globally through the Log Analytics blade in the Azure portal as well. Please refer to this section for more information on syslog and facility log collection. <br><br>

**To configure local syslog messages to be sent to Log Analytics**
1.	In your SSH session, edit the file **95-omsagent.conf** with your favorite editor (for example the vi editor), located in the folder **/etc/rsyslog.d** this is based on **rsyslog**. See the link mentioned above to get information for **syslog-ng** based Linux distro’s.
2.	Add at least the following line, which will forward the facility **syslog** to Log Analytics:
**syslog @127.0.0.1:25224**
3.	Save and close the updated **95-omsagent.conf** file
4.	Create a new file called **security_events.conf** in the folder **etc/opt/microsoft/omsagent/{workspaceId}/conf/omsagent.d**
5.	Add the following content: <br>
``` text
<source>
  type syslog
  port 25226
  bind 127.0.0.1
  protocol_type udp
  tag oms.security
  format /^(?<time>(?:\w+ +){2,3}(?:\d+:){2}\d+):? ?(?:(?<host>[^: ]+) ?:?)? (?<ident>[a-zA-Z0-9_%\/\.\-]*)(?:\[(?<pid>[0-9]+)\])?: *(?<message>.*)$/
</source>
<filter oms.security.**>
  type filter_syslog_security
</filter>
```

**Configure forwarding of MCAS SIEM agent events**
1.	In your SSH session, navigate to the folder **/etc/rsyslog.d**  and create a new file called **security-config-omsagent.conf**
Note: this applies to rsyslog based distro’s only
2.	Add the following content:<br>
``` text
#OMS_facility = syslog
syslog       @127.0.0.1:25226
```
### Step 7 - Create a MCAS alert and verify the entry in Log Analytics
Now that we have configured the MCAS SIEM agent, the Microsoft Management Agent for syslog forwarding, it’s time to test-drive if an MCAS alert is being forwarded to the ASC Log Analytics workspace.
1.	Open the MCAS portal and create an Activity policy

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/activity_policy.png)
<br>

2.	Create a Box login Activity policy which will generate an alert <br>

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/activity_policy2.png)
<br><br>

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/alerts.png)
<br>

**Note** <br>
*You can use any activity which raises an alert* <br><br>
3.	Create the alert, login to Box and verify that MCAS generated an alert:

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/alerts2.png)
<br><br>

Since we are also capturing MCAS activities we can see the creation of the alert policy flowing by in our syslog:<br>

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/alerts3.png)
<br><br>

Next, we see the actual alert coming by:<br>

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/alerts4.png)
<br><br>

If we switch over to the Log Analytics workspace and run the following query we can see the alert (I have multiple, so you will see a different timestamp): <br>

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/la_query.png)
<br><br>

**Note** <br>
*Since we are using CEF over Syslog, the Log Analytics data type is **CommonSecurityLog*** <br>

In addition, since we are also sending MCAS activities to Log Analytics (in addition to alerts) we can also see the changes I have made in the MCAS portal. In this example I have updated an activity policy called **CAS alert rule logon from a risky IP address**:

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/cas_alert.png)
<br><br>

### Step 8 - Create a custom alert in Azure Security Center
Now that we have the data in the ASC Log Analytics workspace, our final step is to create an ASC custom alert.
1.	Open the Azure Security Center blade in the Azure Portal
2.	Under **Threat Protection**, click on **Custom alert rules** <br>

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/custom_alert.png)
<br><br>

3.	Click on **New custom alert rule**
4.	Fill in the **Create custom alert rule** properties, but do notice that in the **Search Query**, you can be very specific or generic, that’s up to you

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/custom_alert_rule.png)
<br><br>

5.	You may want to experiment with the **Period, Evaluation and Suppress Alerts** values, what works best for you: <br>

![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/custom_alert_rule2.png)
<br><br>	

6.	After we have created the ASC custom alert, we should get an alert (the number 2, reflects the number of alerts): <br>
![alt text](https://raw.githubusercontent.com/tianderturpijn/MCAS/master/MCAS-ASC-integration/screenshots/mcas_alert.png)
<br><br>

When clicking on the alert, I can do my investigations and invoke a Logic App playbook if I wanted.

### Summary
In this article I have showed you how you can integrate Microsoft Cloud App Security with Azure Security Center by leveraging the MCAS SIEM agent, which can forward CEF events to a Log Analytics workspace. From there you can run deep investigation and correlation queries and build custom ASC alerts on top of that.


















