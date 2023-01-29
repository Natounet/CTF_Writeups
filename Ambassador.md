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
  
After login on the Grafana webapp, we can see in datasources that a mysql server is connected on a <strong>grafana</strong> with the <strong>grafana</strong> user, but the password remain empty.<br>

![image](https://user-images.githubusercontent.com/70477133/215297824-ca212a60-8000-40db-96eb-ccd676ad0fd8.png)
![image](https://user-images.githubusercontent.com/70477133/215297881-932e8c81-d7a9-4f15-b259-b87a8f316564.png)

  
This is because Grafana don't show the password in the webapp, but the password is somewhere, in the <strong>grafana database</strong> that we retrieve earlier.<br>
We can explore the retrieved database using <strong>DB Browser for Sqlite</strong><br>
We know that we need to find the <strong>datasource</strong>, and there is a datasource table in the database<br>

![image](https://user-images.githubusercontent.com/70477133/215297930-8e0bcecf-7949-4e7e-af33-611740c2d946.png)

[Here](https://github.com/pedrohavay/exploit-grafana-CVE-2021-43798) is an automated script that allow to retrieve usefull files from Grafana, search for Grafana tokens and decrypt Grafana passwords.
  
## 5. Mysql
  
With the credentials we found, we can connect to the mysql database.<br>
  
```mysql -u grafana -p -h IP```
  
  We can find an odd databased called <strong>whackywidget</strong>
  This database contain only one table <strong>users</strong>

| user      | pass       |
|-----------|------------|
| developer | [REDACTED] |
  
We have found the password of the developer account ! It is in base64 encoding, we need to decode it and then we can connect with ssh to the server.
  
## 6. Privilege Escalation
  
On the server, we can directly find the <strong>user.txt</strong> on the developer home directory.<br>
When listing all files in the home directory, <strong>including hidden file</strong>, one particular file is present <strong>.gitconfig</strong><br>
  This file leak us an important information : <strong>directory = /opt/my-app</strong><br>
  
  In this directory, we can find an application named <strong>whackywidget</strong> and a <strong>.git</strong> directory, which contain <strong> all changes made to the application</strong><br>

Using a program like [GitKraken](https://www.gitkraken.com/), we can see the changes made to the application.<br>
  For this, you need to transfer the <strong>.git</strong> directory to your attacker machine, and open it with GitKraken<br>

  the actual version of the script <strong>put-config-in-consul.sh</strong> can't be run because it need a token that we don't have.<br>
  But reviewing ancient version of the script using GitKraken, we can find a token that was used before.<br>
  ![image](https://user-images.githubusercontent.com/70477133/215299108-04a0911d-6b62-4f9f-b58f-aa87c1ec99b6.png)

You can then use a RCE like [this](https://github.com/GatoGamer1155/Hashicorp-Consul-RCE-via-API/blob/main/exploit.py) on to get root using consul.<br>
Then get the root flag in the /root folder

