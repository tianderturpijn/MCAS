## Integrating Microsoft Cloud App Security with Azure Security Center

##### Summary
This article describes how to integrate Microsoft Cloud App Security (MCAS) with Azure Security Center (ASC). We will leverage the MCAS SIEM agent capability to forward CEF over syslog events to an ASC Log Analytics (LA) workspace. From the ASC workspace we can create custom alerts and invoke Logic App playbooks for any custom action, like remediation actions or creating a ticket in a ticketing system.

##### Assumptions
This blogpost is assuming that you have the following in place: <br>
*  A configured MCAS environment, if you don't have a MCAS environment, go <a href="https://docs.microsoft.com/en-us/cloud-app-security/getting-started-with-cloud-app-security" target="_blank">here</a>
* A configured ASC environment, if you don't have an ASC environment, go <a href="https://docs.microsoft.com/en-us/azure/security-center/security-center-get-started" target="_blank">here</a>
 
##### High-level steps
The scenario we want to achieve is high-level captured in the following figure: <br>

![](https://github.com/tianderturpijn/mcas/blob/master/mcas-asc%20integration/screenshots/highlevel_overview.png)