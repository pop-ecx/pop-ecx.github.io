---
title: "Writing Custom Wazuh Rules"
date: 2026-01-22T15:48:20+03:00
tags: ["wazuh", "custom rules", "decoders", "siem", "blue team"]
categories: ["wazuh", "custom rules", "decoders", "blue team"]
keywords: ["wazuh", "custom rules", "decoders"]
draft: false
---

[Wazuh](https://wazuh.com/) is a powerful open-source security monitoring platform that acts as a SIEM/XDR
solution. Though it has some [shortcomings](https://zaferbalkan.com/wazuh-pain-points/), it has tons of strengths, my favorite
ones being; its customizability and its open-source nature. This article will be
more or less a tutorial on how to write a simple custom wazuh rule to detect potential
port scans on a monitored host.

The setup is as follows:

- Wazuh manager (4.14.1) running on a Linux VM.
- Wazuh agent (4.14.1) running on a Linux VM.

## Prologue
Before we start, we are going to use sysmon to generate some logs on the agent.
To install sysmon for Linux, follow the instructions [here](https://github.com/microsoft/SysmonForLinux/blob/main/INSTALL.md)
and make sure it is sending logs either to syslog or perhaps via journald.

## Tweaking the Wazuh agent
First, we need to make sure wazuh agent is collecting sysmon logs. To do this,
check for the following configuration in `/var/ossec/etc/ossec.conf` on the agent:

```xml
<localfile>
    <log_format>syslog</log_format>
    <location>/var/log/syslog</location>
</localfile>
```
If you are using journald, you can use the following configuration instead:

```xml
<localfile>
    <log_format>journald</log_format>
    <location>/var/log/journal</location>
</localfile>
```
> From here on, we'll assume syslog is being used.

After making the changes restart the wazuh-agent service, then generate some syslogs by running
nmap against the agent machine. You can grep for eventId 3 which indicates a network connection
event:

```bash
sudo grep -i "<EventID>3" /var/log/syslog
```
If you get hits, move on to the next step.

## Writing custom decoder
Next, we need to write a custom decoder to parse the sysmon logs. You can edit the file
`/var/ossec/etc/decoders/local_decoder.xml` on the manager and add the following decoder:

```xml
<decoder name="sysmon-linux">
  <program_name>sysmon</program_name>
</decoder>

<decoder name="sysmon-linux">
  <parent>sysmon-linux</parent>
  <prematch>SourceIp</prematch>
  <regex type="pcre2" offset="after_prematch">([0-9a-fA-F:.]+)</regex>
  <order>srcip</order>
</decoder>

<decoder name="sysmon-linux">
  <parent>sysmon-linux</parent>
  <regex>EventID>(\d+)</regex>
  <order>id</order>
</decoder>
```

Wazuh uses XML files for decoders, and the above decoder will extract the source IP
address and event ID from sysmon logs. Those fields will be used in the custom rule.

> Wazuh uses various kinds of regex engines, including os regex. In this case we are using pcre2.
> I had a bit of a tough time tweaking my regex to wazuh's liking since os regex engine is a bit
> limited. In the end, I settled for pcre2.

## Writing custom rule
Now we can move on to the custom rule. We'll edit the file
`/var/ossec/etc/rules/local_rules.xml` on the manager and add the following rule:

```xml
<group name="linux,sysmon,">
  <rule id="100200" level="1">
    <decoded_as>sysmon-linux</decoded_as>
    <id>^3$</id>
    <description>Sysmon Event ID 3 - Network connection</description>
    <group>sysmon_event3,</group>
  </rule>

  <rule id="100300" level="12" frequency="4" timeframe="12">
    <if_matched_group>sysmon</if_matched_group>
    <same_srcip>srcip</same_srcip>
    <description>Possible port scan detected from $(srcip)</description>
    <group>sysmon_event3,port_scan,</group>
  </rule>
</group>
```

The first rule (id 100200) is a base rule that matches sysmon event ID 3. The second
(id 100300) is the actual port scan detection rule. It is a correlation rule that triggers
if it sees 4 or more event ID 3 logs from the same source IP within a 12 second
timeframe. When triggered, it raises the alert level to 12 and generates an alert indicating
a possible port scan.

Once done you can locally test the rules using the `wazuh-logtest` utility on the manager

```bash
/var/ossec/bin/wazuh-logtest
```

and enter one of the sysmon logs. If everything went well, you should see this:

```bash
**Phase 1: Completed pre-decoding.
	full event: 'Jan 22 16:17:16 testserver sysmon: <Event><System><Provider Name="Linux-Sysmon" Guid="{ff032593-a8d3-4f13-b0d6-01fc615a0f97}"/><EventID>3</EventID><Version>5</Version><Level>4</Level><Task>3</Task><Opcode>0</Opcode><Keywords>0x8000000000000000</Keywords><TimeCreated SystemTime="2026-01-22T13:17:16.405709000Z"/><EventRecordID>62099</EventRecordID><Correlation/><Execution ProcessID="4107731" ThreadID="4107731"/><Channel>Linux-Sysmon/Operational</Channel><Computer>testserver</Computer><Security UserId="0"/></System><EventData><Data Name="RuleName">-</Data><Data Name="UtcTime">2026-01-22 13:17:16.412</Data><Data Name="ProcessGuid">{fd88a264-6181-6936-5d32-072e16560000}</Data><Data Name="ProcessId">2756479</Data><Data Name="Protocol">tcp</Data><Data Name="Initiated">true</Data><Data Name="SourceIsIpv6">false</Data><Data Name="SourceIp">192.168.2.2</Data><Data Name="SourceHostname">-</Data><Data Name="SourcePort">37678</Data><Data Name="SourcePortName">-</Data><Data Name="DestinationIsIpv6">false</Data><Data Name="DestinationIp">192.168.2.3</Data><Data Name="DestinationHostname">-</Data><Data Name="DestinationPort">3307</Data><Data Name="DestinationPortName">-</Data></EventData></Event>'
	timestamp: 'Jan 22 16:17:16'
	hostname: 'testserver'
	program_name: 'sysmon'

**Phase 2: Completed decoding.
	name: 'sysmon-linux'
	parent: 'sysmon-linux'
	id: '3'
	srcip: '192.168.2.2'

**Phase 3: Completed filtering (rules).
	id: '100200'
	level: '1'
	description: 'Sysmon Event ID 3 - Network connection'
	groups: '["linux","sysmon","sysmon_event3"]'
	firedtimes: '1'
	mail: 'false'
```
If you are using the web interface, if you send the logs repeatedly (4 or more times)
you will see the port scan alert being triggered.

## Epilogue
Now to test it, run nmap or rustscan against the agent and see alerts on the dashboard.

> This tutorial is a basic example and did not take into account the nuances of
> real-world port scans. Also, regex is this horrible thing Satan uses to torment us mortals.

[Wazuh ruleset xml syntax reference](https://documentation.wazuh.com/current/user-manual/ruleset/ruleset-xml-syntax/rules.html#if-matched-group)

[Wazuh decoders syntax reference](https://documentation.wazuh.com/current/user-manual/ruleset/ruleset-xml-syntax/decoders.html)

[Wazuh regex reference](https://documentation.wazuh.com/current/user-manual/ruleset/ruleset-xml-syntax/regex.html)

[Wazuh pcre regex reference](https://documentation.wazuh.com/current/user-manual/ruleset/ruleset-xml-syntax/pcre2.html)
