--- 
title: "LDAP Nightmare"
date: 2025-01-05
--- 

# LDAP Nightmare

SafeBreach recently released a POC on their [GitHub](https://github.com/SafeBreach-Labs/CVE-2024-49113) for the LDAP DoS vulnerability CVE-2024-49113 found in Microsoft Domain Controllers. They put out a [blog post](https://www.safebreach.com/blog/ldapnightmare-safebreach-labs-publishes-first-proof-of-concept-exploit-for-cve-2024-49113/) with their technical analysis of the vulnerability. 

I wanted to build a lab to replicate the attack and see what data could be used for detection or hunting for this attack in the wild. 

## Lab Setup 

I set up my lab in VMware Workstation Pro, which consisted of the following:
- Attacker machine- Kali Linux
- Target Machine- Windows Server 2022
- Logging Server- Ubuntu with Splunk
- Attacker Domain- Domain I already owned



![Lab Setup](https://github.com/user-attachments/assets/e64267fd-3d5e-4bed-9063-51a7cfe6a7b5 "Lab Setup")

### Domain requirements 

For the attack to work, even in a lab, a domain on the internet that the attacker owns is required. Luckily, I have one just lying around that can be used. On the domain, two SRV records are needed that can be queried for the attack:
- `_ldap._tcp.dc._msdcs`.`attacker_domain` `listen_port` `attacker_hostname`
- `_ldap._tcp.default-first-site-name._sites.dc._msdcs` `listen_port` `attacker_hostname`


![image](https://github.com/user-attachments/assets/d936a76f-68dc-4018-ae6b-57f5c57800c1)

### Target Server Requirements

I used a Windows Server 2022, promoted to a DC, and added the Light Weight Directory Services. 

To help with investigating the attack, I installed the following tools:
- Procmon
- Wireshark
- Splunk forwarder- I used this [script](https://github.com/BoredHackerBlog/lazylab/blob/main/scripts/install_splunk.bat) from [BoredHackerBlog](https://github.com/BoredHackerBlog)'s lazylab repo. (if you use this, change the IP in the script to your logging host before running). 
- Enabled Sysmon logging- I used this [script](https://github.com/BoredHackerBlog/lazylab/blob/main/scripts/setup_logging.bat) also from BoredHackerBlog's lazylab repo. 

The POC repository also mentioned ensuring TCP Port `49664` is listening. I ensured the required port and LDAP port `389` were opened and listening on the host. 

<img width="1143" alt="image" src="https://github.com/user-attachments/assets/367dba42-2402-4f0b-89b4-f4ed0bfd95a3" />



_**Note** the POC mentioned that NBNS is used to query the attacker's hostname. As this is just for a lab, I cheated a little and added the Kali machine to the target host's hosts file._ 

![image](https://github.com/user-attachments/assets/9eeef528-88e6-4174-b001-3c735fd686e0)

### Attacker Setup

The only requirement for the attacker is to clone the POC repo from [here](https://github.com/SafeBreach-Labs/CVE-2024-49113). I placed it into its own directory on the host `CVE-2024-49113` to keep it clean. Once downloaded, install the `requirements.txt` and activate a Python environment, and you are good to go!

<img width="489" alt="image" src="https://github.com/user-attachments/assets/72aaf157-273f-4e27-9d50-a17df021ac38" />

### Splunk Host

For logging, I set up Splunk in a Docker container. The free version of Splunk is more than enough to handle the logs from a single host. The script from BoredHackerBlog's lazylab will set up real-time forwarding to your Splunk instance, so you do not need to wait for logs to populate.

## Running the Attack

Before running the POC, ensure Wireshark is running on the host to capture the attack.

_This is jumping ahead a bit, but to save time, also have Procmon running and filter by lsass; when you return to the server after the exploit, you will notice that lsass has also crashed._

![image](https://github.com/user-attachments/assets/df3a98ff-a42e-4b4a-9596-8212482501a6)


Running the attack is simple once configured correctly. Run the following command and quickly jump back to your target host to save your Wireshark capture because the host is about to crash hard! 


```python
python3 LdapNightmare.py 192.168.111.131 --domain-name <exploit-domain>.com
```

![image](https://github.com/user-attachments/assets/60a4ef2c-c6bb-4251-85e3-14d65242ebd7)

The server displays a message that it will reboot in one minute. After this, nothing can be done, and the server will reboot.

![image](https://github.com/user-attachments/assets/50884665-8415-422e-ae83-5872c808c6e4)

View with Procmon opened with lsass gone:

![image](https://github.com/user-attachments/assets/9091c36e-5b00-4b08-aa5d-cb7bb1017202)

That is it! CVE-2024-49113 has successfully been exploited. Next up is investigating the attack in logs. 

## Investigating Attack

### Wireshark

First, let's analyze the captured network events in Wireshark. We can see all the attack traffic by filtering with UDP in Wireshark. DNS and CLDAP traffic were captured at the time of the attack. 

![image](https://github.com/user-attachments/assets/7acd5ceb-df51-42a5-a8c9-bf36dc94f5ce)

In the packets, we can see the connectionless LDAP requests from the victim machine to my domain using Netlogon for the user Administrator. 

![image](https://github.com/user-attachments/assets/6dcb9df2-beed-4f6c-9dea-1cdf18c92f40)

![image](https://github.com/user-attachments/assets/cfeae5b3-0d53-4ee6-b2c3-9ab9902d64ea)

We also get some DNS requests for the SRV records we initially set up in my domain.


![image](https://github.com/user-attachments/assets/7375379a-d6a2-4530-81b0-9d55ddfd1da0)

Now, let's take a gander at what logs picked up during the attack. 

### Splunk

I decided to use Splunk for my log investigation as it is what I am used to. Technically, you can use any log viewer you want, including Event Viewer, if you like pain, but I found Splunk to be the easiest. 

Since we already know there were suspicious DNS requests, let's start there and see what Sysmon picked up. We know there were requests for my domain, and filtering the logs by the domain showed the requests were made by lsass. 


<img width="1111" alt="image" src="https://github.com/user-attachments/assets/8b58d000-bf0a-49bd-8cc0-cf2a958589c0" />

I then pivoted to see what else lsass was up to by filtering on the process id `664`. I noticed WerFault targeting lsass, which is normal since the system crashed. (This is why I mentioned jumping ahead above. This made me filter Procmon on lsass to see if it crashed.) 

<img width="1109" alt="image" src="https://github.com/user-attachments/assets/53da3c80-e1cf-4dce-a162-28d6c505667c" />

![image](https://github.com/user-attachments/assets/73dcb404-d728-4e4c-a819-f1892c21408b)


Fun side note! If you exploit the server with no user logged on, it will not allow you to login since lsass crashed. 

<img width="1149" alt="image" src="https://github.com/user-attachments/assets/d9fc5ee5-f328-4e6c-afe4-9d36116a416c" />

Now that we know how the attack looks in logs, we can use that information to hunt for attempts to exploit this CVE. 

## Hunting the Attack

After a little Splunk-fu we can build a query to help search for unusual DNS requests based on the SRV records we used to exploit the attack. 

```
index=main sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" QueryName=_ldap._tcp.dc._msdcs.* OR QueryName=_ldap._tcp.default-first-site-name._sites.dc._msdcs.*
| search QueryName!=*testing.local* QueryName!=*localdomain*
| eval split_DNS=split(query, ".")
| eval suspicious_domain=mvindex(split_DNS, -3) + "." + mvindex(split_DNS, -2)
| table suspicious_domain, query, answer
```

This takes DNS queries and splits them into the domain, query, query results, process that made the query, and user. (you can also add `_time` to the table command if you want to filter by time). 

![image](https://github.com/user-attachments/assets/9c623c1c-bbf6-4b34-ad50-194893d4d68a)

This query can search for DNS requests to domains not normally seen in the environment with responses with hostnames or domains that would not commonly be seen in the environment. Additionally, lsass should normally not be querying these domains (unless that's what you do in your environment for some reason!) 

Seeing these DNS requests around the time of a server crash could be a strong indication that this exploit has taken place and warrants investigation into the server for any signs of compromise. 

## Mitigation

Microsoft has released a patch, which can be seen [here](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2024-49113).

## Conclusion

This was a pretty fun POC to test out, and shout out to SafeBreach for providing it to test out. While the POC only crashes a server, it would not take an attacker much effort to add this into an exploit chain to cause more havoc. (or they could be jerks and constantly DoS you by setting up a script to constantly crash your server, which is a whole different issue.) 

## References

https://www.safebreach.com/blog/ldapnightmare-safebreach-labs-publishes-first-proof-of-concept-exploit-for-cve-2024-49113/

https://github.com/SafeBreach-Labs/CVE-2024-49113

https://msrc.microsoft.com/update-guide/vulnerability/CVE-2024-49113

https://github.com/BoredHackerBlog/lazylab

