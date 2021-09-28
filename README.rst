=========================================
Splunk Data Ingest Optimization
=========================================

Read this document in its entirely before proceeding to reduce data ingest.
Failing to understand why, when, what and how to reduce data ingest can lead to failed compliance audits, termination of employment, loss of business, legal action, loss of data and Splunk Apps, Reports and Dashboard to not function.
You have been warned!!

Learn how to:

* Identify what data to reduce
* Reduce common data sources coming into Splunk such as Windows Event Logs, Linux Events, Network Syslog
* Methods for reducing data by turning on/off inputs and programmatically
* Validate that reduction is happening
* Setting better retention policies for Indexes




=========================================
Precautions:
=========================================

1. Removing data prior to ingestion will raise eyebrows and increase blood pressure with compliance auditors and has a very high potential to fail compliance audits.
2. **Document all data removal reasons.**  Include the data source, reason for removal and have official signatures by accountable stakeholders.

**NOTIONAL EXAMPLE**

* **Title:** Remove Google DNS traffic from firewall syslog.

* **Data Source(s):** Cisco ASA Firewall Traffic Syslog

* **Remove:** dest_port=53, dest_ip=8.8.8.8, dest_ip=8.8.4.4, dest_ip=8.4.4.4 firewall syslog events.

* **Reason:** These are benign, normal firewall events, high volume, requires large amounts of storage and not required for Security use cases

* **Risk(s):** DNS queries to Google DNS servers to malicious sites/services may still occur.  DNS Server logs are being ingested to Splunk to ensure proper monitoring, reporting and alerting

* **Signatures:** <insert Employee names, email addresses and date of signatures>

3. Present Document to Auditor ASAP for transparency and integrity purposes.  Less "WTFs" from Auditors during the audit, the better.

=========================================
Reasons for Data Reduction
=========================================

1. Why keep data events that are not **REQUIRED** or only ingest data that is **REQUIRED**
2. Storing unneeded data events increases storage amounts, TCO.
3. Save on Splunk Licensing to ingest more meaningful and important data sources.



=========================================
"Best" Practices
=========================================

* Ingest ALL events into Splunk first, then determine which events can safely be removed after carefully vetting.
* Splunk UF using the Splunk Add-on for Windows (Splunk_TA_Windows), by default will ingest all historical Application, Security and System Event Logs that currently reside on the Windows Host.  (see https://github.com/PMJeffery/splunk4windows for pre-configured inputs)
* Splunk UF using the Splunk Add-on for \*nix (Splunk_TA_Nix), by default will ingest all historical log files in various folders when enabled that currently reside on the \*nix Host.  
* Firewall syslog is a LIVE data stream, so it would be smart to ingest at least 1 week's worth of syslog before deciding what to remove.

===========================================================================
Methodologies for Determining what Data Events are Required or Necessary
===========================================================================

**1. Validate inputs.conf enabled settings**
#1 cause for exessive data ingest is data inputs that are enabled that shouldn't be - **so do this first!!**
* Review inputs.conf to see what data is being pushed to Splunk.
* Be absolutely sure the inputs are enabled/disabled as required or needed
* Splunk Source Types can be split during ingestion, so verify by looking at the props.conf.  This is common for network syslog and API inputs.

**2. Validate what Source Types are being sent to Splunk by Host and Index:**

.. code-block:: powershell
    
    |tstats values(sourcetype) as Sourcetype latest(_time) as Time groupby index, host | convert ctime(Time) 
    
.. image:: https://user-images.githubusercontent.com/20860518/128613065-3a092b46-d33b-48c1-95a2-b0a56f3139a1.png
    
**3. Practically all Compliance Frameworks dictate what data is required/needed with a few web searches**

**4. Understand your organization's records management policy - retention length**
* For Security and Compliance reasons, the vast majority of them require at least 1 year's worth of relevant data.
* Separate out your indexes based on retention lengths and access control (see Index Retention Settings below)

**5. Do simple Splunk statistical searches against your logs to help identify which events are needed.**

Example SPL:

Windows Event Logs: | stats count by EventCode, EventCodeDescription | sort -count

Firewall Logs: dest_port=53 | stats count by dest_ip | sort -count

General Logs: | stats count by message | sort -count


===========================================================================
How does Programmatic Data Reduction work in Splunk?
===========================================================================

* Removing whole Events or partial Events will require the use of Regular Expression (RegEx) and SEDCMD.
* SEDCMD is used in the props.conf file on the Heavy Forwarder (HF) or Universal Forwarder (UF)
* Best practice is to use a Heavy Forwarder (HF) or Universal Forwarder (UF) to remove data 
* UF has limitations where it does not do Pre-event filtering, Event routing and Event Parsing* https://docs.splunk.com/Documentation/Splunk/8.2.1/Forwarding/Typesofforwarders#Forwarder_comparison
* HF must be used for network syslog and API data inputs filtering/removal.
* Avoid having the Indexers do any removal of data as it can be computational expensive with high velocity data and thus degrading Indexer performance.
* Once data is indexed it cannot be truely deleted unless the entire index is first deleted and the data re-ingested. 


Super quick primer on "sed":

* sed is a *nix command that is basically way to find text and optionally to do substitutions like "find and replace".


SEDCMD is broken down into 2 major parts; before and after the '=': SEDCMD-<any text here> = s/find-this-text-with-RegEx/do-something

SEDCMD can be used to accomplish at least 4 different things in Splunk:

1. Remove specific text

.. code-block:: powershell

   SEDCMD-remove-this-text = s/^(?!(000|666|9))\d{3}-(?!00)\d{2}-(?!0000)\d{4}$//

"s/" looks for a string that resembles a US Social Security Number (SSN)
"//" deletes the text
This is useful for deleting sensitive information contained in the Event

2. Remove Whole Events

.. code-block:: powershell

   SEDCMD-find-8.8.8.8-remove-whole-event = s/.*8\.8\.8\.8*//

"s/" looks for a string "8.8.8.8"  ".*" before and after "8.8.8.8" is RegEx to capture all characters before and after "8.8.8.8"
"//" deletes what is in the RegEx capture
This is useful for deleting entire events 

3. Find and Replace Text

.. code-block:: powershell

   SEDCMD-find-SSN-replace-with-Xs = s/^(?!(000|666|9))\d{3}-(?!00)\d{2}-(?!0000)\d{4}$/xxx-xx-xxxx

"s/" looks for a string that resembles a US Social Security Number (SSN)
"/xxx-xx-xxxx" replaces "SSN" with "xxx-xx-xxxx"
This is useful for obfuscating data 

3. "Compress" Data

.. code-block:: powershell

   SEDCMD-compress-text = s/0x0000000000000000000/0/g

"s/" looks for a string "0x0000000000000000000"
"/0" replaces "0x0000000000000000000" with "0"
"/g" makes the regex global and thus will find all instances 
This is useful for compressing data... I'm look at you, Cisco Firepower logs 


**Examples of Data Flow w/ Filtering/Removal:**

Windows Host (w/ UF) Allow/Block List (Splunk TA Windows, inputs.conf) -> Splunk Indexer

Firewall Syslog -> HF (props.conf filtering) -> Splunk Indexer


* This document will NOT cover Splunk's Data Stream Processor as it is out of scope and as of this writing the vast majority of Splunk Customers does not use it.


..............................................................................................................................

===========================================================================
Windows Event Log Reduction
===========================================================================

**Pro Tip:** Disable XML-based Events (enabled by default) in the inputs.conf for all Windows Event Logs as the XML-based events are roughly 20% to 50% larger than the vanilla Events.
The XML tags and XML metadata eat up that extra space.

Splunk_TA_Windows/local/inputs.conf - Find and Replace renderXml=true to renderXml=false

.. code-block:: powershell

   renderXml=false

Use this GitHub Repo for pre-configured (disabled XML and relevant inputs enabled) Windows Inputs: https://github.com/PMJeffery/splunk4windows

**Windows Event Logs** Determine which EventIDs/EventCodes should only be brought into Splunk.  A fantastic resource by the `Joint Sigint Cyber Unit of the Netherlands <https://github.com/JSCU-NL/logging-essentials>`_ has an amazing collection of Windows Event IDs that should be collected.  They also provide GPO configs to enable them.

Sample SPL (depending upon amount of ingested events, CPU speed and Disk IOPS, this search can take a few minutes to run):


.. code-block:: powershell

   index=wineventlog daysago=7
   | fields EventCode EventCodeDescription
   | stats count by EventCode EventCodeDescription
   | sort -count

Screenshot Example: 

.. image:: https://user-images.githubusercontent.com/20860518/128610030-32ea6db1-f3c0-43a7-9abd-93a986bce5ee.png

Based on the results you should be able to understand which EventCodes are coming in by volume ("count" column) and then determine which EventCodes you can keep or stop ingesting.
Keep in mind that you can "bar napkin math" the ingest savings.  Windows events are roughly 2kb each on average.

(Event Count x 2kb)/7 = estimated daily savings

EventCode=4624 = 91840 Events over 7 days.  91840x2kb=183,680kb/7=26,240KB or ~26MB saved/day for that one specific Event Code.



On your Deployment Server find the Splunk_TA_Windows/local/inputs.conf and use the white/black list section under each Event Log section to make your modifications.

Example: 

.. code-block:: powershell
    
    [WinEventLog://Security]
    disabled = 0
    start_from = oldest
    current_only = 0
    evt_resolve_ad_obj = 1
    checkpointInterval = 5
    whitelist1 = 4624,4625,4770
    blacklist1 = EventCode="4662" Message="Object Type:(?!\s*groupPolicyContainer)"
    blacklist2 = EventCode="566" Message="Object Type:(?!\s*groupPolicyContainer)"
    renderXml=false
    index=wineventlog


Use the following links for more examples:
* https://community.splunk.com/t5/Getting-Data-In/Filter-Windows-EventCode-using-blacklist-and-Whitelist/m-p/191565
* https://docs.splunk.com/Documentation/WindowsAddOn/8.1.2/User/Configuration


====================================================================================================================================================================
Linux Log Reduction
====================================================================================================================================================================
Using the UF and the Splunk_TA_nix Add-on, you can enable specific folders to monitor in real-time.  You can either explicitly list the files you want like /var/log/messages or use RegEx and wildcards to broaden your scope.

Here are some examples:

* https://community.splunk.com/t5/Getting-Data-In/How-to-write-a-monitor-stanza-in-inputs-conf-to-monitor-a-file/m-p/290024

* https://docs.splunk.com/Documentation/Splunk/latest/Data/Specifyinputpathswithwildcards


If you find a lot repeat messages that you are 100% sure you do not need, then use the SEDCMD in local/props.conf to remove them. 


====================================================================================================================================================================
Network Syslog; Firewalls specifically
====================================================================================================================================================================
One of the most valuable and also the most volumous data source comes from your firewall.  Firewall syslog even for small organizations of 150 users can easily hit 20GB+/day.
"Next-Gen" firewalls like Palo Alto, Cisco Firepower, Fortigate, Sonicwall etc. have much larger syslog events than a vanilla firewalls.

It is very important to ingest at least 1 weeks worth of firewall syslog to determine common, normal events that you may want to remove.

**Best Practice** Send firewall syslog to a Linux Syslog Server, configure rsyslog/syslog-ng to write those to a flat file, and set your rotation policy to keep at least 1 day's
worth of logs - 1 live real-time file and a 2nd file of the previous day's logs
Install and configure Splunk Enterprise with the Add-on for your firewall.  Enable the Data Input to monitor that log file and tag it with the correct Sourcetype. You can restart/stop the Splunk service and it will pick up where it left off since Linux Syslog is storing flat files.  You will lose data when you restart/stop the Linux server.  The use of load balancers between the firewall and multiple Linux syslog servers can be used, but outside the scope of this document.

**Risky Practice**  Send Firewall syslog directly to Splunk Indexer or Splunk HF.  Risk here is that when you restart the host or the Splunk service, during that time, all syslog data will be lost.
You will need to document each time this happens and present this to the Auditor to explain the loss of data.


There are 2 distintly different ways to reduce firewall syslog volume with SEDCMD and basic RegEx.

1. Delete/NULL whole events based on "normal, everyday traffic"
2. "Compress" useless data within the event

**Delete/NULL whole Firewall syslog events**
A simple example would be Outbound DNS traffic on port 53 and inbound DNS from designated DNS servers.  Most internal resolvers use specified external DNS servers for forwarding requests and mobile devices will
use common DNS servers like Google (8.8.8.8, 8.8.4.4) or CloudFlare DNS servers (1.1.1.1).  By knowing what your SAFE, DESIGNATED and VETTED External DNS resolvers are, you 
can remove those events.  Unknown, rogue or unvetted DNS servers showing up in your firewall syslog that are both ALLOWED or BLOCKED should be treated with extreme prejudice and must be investigated immediately.


Here is sample SPL for this example

.. code-block:: powershell

   index=your_firewall_index dest_port=53 daysago=7
   | stats count by dest_ip
   | sort -count

Hopefully, only normal DNS servers are found.

With a proper list of DNS server IPs, you can now create your SEDCMD in props.conf. 
This must be done on the Heavy Forwarder.  For this example, we will be using a Cisco ASA and using the Splunk Add-on for Cisco ASA: https://splunkbase.splunk.com/app/1620/
and https://docs.splunk.com/Documentation/AddOns/released/CiscoASA/Distributeddeployment


**These steps should be good for any network device syslog.**
1. Install the ASA Add-on on the appropriate Splunk Servers: https://docs.splunk.com/Documentation/AddOns/released/CiscoASA/Installationoverview
2. Install the ASA add-on on your HF (manually or via Deployment Server)
3. Configure the firewall to send syslog to a remote IP (Linux syslog server/Splunk HF) over UDP 514
4. Create a new Index on your Splunk Indexer or Splunk Cloud to store your firewall syslog
5. **Splunk Cloud Customers Only** ensure that the Splunk Cloud App is installed on your HF and it is showing up in Splunk Cloud before proceeding
5. **On-Prem Splunk Enterprise Only** ensure that the HF is configured to send data to Splunk Indexer(s) before proceeding

Read both of the following options and pick the best one.

**1. Splunk Heavy Forwarder Configuration to monitor flat-file log writen by Linux Syslog (rsyslog/syslog-ng) ASA:**
* **This method reduces the likelihood of data loss when the Splunk server is restarted or stopped as it is stored via Linux syslog**
0. Configure Linux rsyslog/syslog-ng to accept UDP 514 syslog traffic and write it to a file
* rsyslog: https://www.tecmint.com/install-rsyslog-centralized-logging-in-centos-ubuntu/
* rsyslog Log Rotation: https://www.tecmint.com/manage-linux-system-logs-using-rsyslogd-and-logrotate/
* syslog-ng: https://www.syslog-ng.com/technical-documents/doc/syslog-ng-open-source-edition/3.16/administration-guide/12#TOPIC-956429
* syslog-ng Log Rotation: https://www.syslog-ng.com/technical-documents/doc/syslog-ng-open-source-edition/3.16/administration-guide/86
* Example folder/file structure: 
/var/log/sendtosplunk/firewalls/%HOSTNAME%/%$MONTH%-%$DAY%-%$YEAR%.log
/var/log/sendtosplunk/switches/%HOSTNAME%/%$MONTH%-%$DAY%-%$YEAR%.log
** Double-check to make sure the file(s) are being populated with firewall events before proceeding


1. Navigate to $SPLUNK_HOME/etc/apps/Splunk_TA_cisco-asa
2. Create a new folder, local
3. Copy props.conf from Splunk_TA_cisco-asa/default to Splunk_TA_cisco-asa/local
4. Navigate to Splunk_TA_cisco-asa/local
5. Create a new file, inputs.conf
6. Copy and edit the following inputs.conf code:


.. code-block:: bash

    [monitor:///var/log/sendtosplunk/firewalls/.../asa.log]
    disabled = false
    index=<your index for firewall data>
    sourcetype = cisco:asa
    ## Optional parameters ##
    # host_segment = 5
    # whitelist = *.log
    # If you have multiple Firewalls within a single folder
    # recursive = true

More info on how to configure the Monitor function: https://docs.splunk.com/Documentation/Splunk/8.2.1/Data/Monitorfilesanddirectorieswithinputs.conf

**2. Splunk Heavy Forwarder Configuration to accept UDP syslog from the ASA:**
1. Create a new Data Input (Settings->Data Inputs->UDP->Create New Input)
2. Be sure to specify "cisco:asa" as the Sourcetype and the appropriate Index you need the data to go to.
2. Navigate to $SPLUNK_HOME/etc/apps/Splunk_TA_cisco-asa
3. Create a new folder, local
4. Copy Splunk_TA_cisco-asa/default/props.conf to Splunk_TA_cisco-asa/local


**How to configure props.conf to use SEDCMD** 


.. code-block:: powershell

   [cisco:asa]
   # Looks for 8.8.* (8.8.8.8 and 8.8.4.4) specifically
   SEDCMD-remove-legit-google-dns-ip-traffic = s/.*?8\.8\..*//
   # Looks for specific IPs (10.25.34.251 and 1.1.1.1) and Google DNS IPs
   SEDCMD-remove-legit-vetted-dns-ip-traffic = s/(.*?8\.8\..*|.*?10\.25\.34\.251.*|.*?1\.1\.1\.1.*)//

The latter SEDCMD RegEx would work better if you have a definitive list of IPs.
Apply one of the examples above as needed.
To see what it is doing and test your own RegEx: https://regex101.com/r/Af9ol8/1

Additionally, you can add more SEDCMD to the same sourcetype and add more to different sourcetypes as needed.

**Dealing with "trash" data within your syslog events**

Cisco FirePower logs contain multiple "zero" hex values that look something like this "0x0000000000000000000000000000" within a single event.  Multiply that by millions of events per day and will add up very quickly.  Assuming it is 20 characters long, that's roughly 20bytes, times the number of times within a single event times the number of events per day, it could be a few hundred precious megabytes saved.  Looking at Cisco's Firepower documentation this hex value doesn't seem to have any real... value.

You will have to find the correct string and replace it below in the SEDCMD in BOTH sourcetypes.
Copy TA-estreamer/default/props.conf to TA-estreamer/local

Edit TA-estreamer/local/props.conf (NOTE: the actual hex value is this example is WRONG, find the right one first!!)

.. code-block:: powershell

   [cisco:firepower:data]
   # Looks globally for "0x0000000000000000000" and replaces it with "0"
   SEDCMD-compress-zero-hex-value-data = s/0x0000000000000000000/0/g
   
   # Looks globally for "0x0000000000000000000" and replaces it with "0"
   [cisco:firepower:syslog]
   SEDCMD-compress-zero-hex-value-syslog = s/0x0000000000000000000/0/g





..............................................................................................................................
=========================================
Issues, Requests, Help
=========================================

This doc should get most people on the right path, but if you have further questions please contact your Splunk Sales Engineer first.  Else, post in the "Issues" section of this repo.

Further examples may be published in this repo, but not in this doc, so click on Watch, slap that like button and subscribe for more!
