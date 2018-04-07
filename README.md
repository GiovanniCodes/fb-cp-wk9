# fb-cp-wk9

Week 9 Project: Honeypot
Due: Friday, April 6th at 11:59pm
Summary: Setup a honeypot and intercept some attempted attacks in the wild.

MHN-Attacks
---
Background
A honeypot is a decoy application, server, or other networked resource that intentionally exposes insecure features which, when exploited by an attacker, will reveal information about the methods, tools, and possibly even the identity of that attacker. Honeypots are commonly used by security researchers to understand the threat landscape facing developers and system administrators, collecting data that might include:

Information about sources of malicious network traffic such as IP addresses, geographic origin, targeted ports, etc.
Information used to harden resources against email spammers
Malware samples
DB vulnerabilities such as SQLI techniques
There are two broad categories of honeypots:

Low-interaction honeypots provide simulations of target resources, typically using emulation or virtualization, to reduce resource consumption, simplify configuration and deployment, and provide solid containment features
High-interaction honeypots expose non-simulated target resources in a way that more closely imitates a production environment to attract more sophisticated attackers and understand more complicated exploitation routes
For example, a low-interaction honeypot might emulate a server capable of accepting SSH connections through a combination of exposed ports and decoy responses, whereas the high-interaction version would feature an actual SSH server possibly misconfigured in some way that makes it vulnerable. In the low-interaction example, attempted exploitation would quickly lead to a dead end for the attacker, perhaps revealing only an IP address and a few attempted commands to the honeypot's maintainer. In the high-interaction example, the attacker would potentially be able to compromise the server, wasting more time and giving away more information about his or her goals.

---
Overview
In this assignment, you will stand up a basic honeypot and demonstrate its effectiveness at detecting and/or collecting data about an attack. Guided instructions for doing this using specific software are provided below, but you are free to take any approach you wish that demonstrates the following basic principles:

Successful configuration and deployment of a network-accessible honeypot server with two primary features:
An attack surface that is vulnerable or exposed in some way to network-based attacks
A network security feature such as an IDS configured to detect and log such attacks
Illustration of at least one attack against the honeypot that can be detected or logged in a way that captures information about the attack or the attacker
Walkthrough
Keeping in mind that there are many ways one could fulfill the above requirements, in this section, we will walkthrough a basic honeypot deployment using a well-supported open source honeypot: Modern Honey Network (MHN). MHN's architecture is modular and extensible and comes with many options for deploying different types of honey pots. In MHN architecture, there is a single admin VM which is used to deploy, manage and collect information from the honeypots, which are deployed as separate VMs. Thus to run MHN, we'll need to setup at least two VMs: the single Admin VM and at least one Honeypot VM.

---
Milestone 0: To the Cloud!

To complete this assignment, you'll need access to a cloud hosting provider to provision the VMs. Many providers offer time-limited free trial accounts with resource limitations, and you should easily be able to complete the requirements for this assignment within these limitations -- though you may need to ensure you cleanup before your trial period expires. The setup we'll walkthrough below has been tested to work with micro-sized VMs with < 1GB memory and < 10GB disk space, so you may often be able to use the smallest possible option when provisioning VMs. All servers in this setup use Ubuntu 14.04 (trusty) -- and most cloud providers offer this as an option. Note that Ubuntu 16.04 and 17.04 will not work.

You can use any cloud provider to which you already have access or that offers a free trial, though you'll need to be familiar with its usage and / or limitations. If you're not sure where to start, we recommend Google Cloud Platform's Free Tier, and while we'll provide general guidelines that should work with most cloud providers, the instructions below will also show insets labeled GCP Users with commands and settings specific to Google Cloud Platform. If you are confident about working with an alternate cloud provider such as AWS feel free to adapt the below instructions accordingly. If this is your first foray into the world of cloud computing, consider starting a GCP trial so you can follow the more specific instructions below.

So to get started, make sure you have authenticated access to your cloud provider. You can provision the VMs any way you like, but you will need to be able to access the VMs via SSH.

GCP Users
To begin, download and install the GCP SDK on your local machine and initialize it such that you can use gcloud from the command line. Be sure to set a default region and zone (these instructions were tested against the us-central1 region and us-central1-c zone). Once you've initialized, you should be able to run gcloud config list and see the project, region, and zone configured in the output.

---
Milestone 1: Create MHN Admin VM

Start by creating the MHN Admin VM via your cloud provider. The VM needs to have an internet-facing IP and accessible to you via ssh (or a similar protocol). As specified above, you can use a small or micro-sized VM for this, but the following attributes are required:

Ubuntu 14.04 (trusty)
HTTP traffic allowed (port 80)
TCP ports 3000 and 10000 need to be open to allow incoming (aka 'ingress') traffic
That last requirement is generally the only one that may require a specific firewall rule to configure properly, because those ports are non-standard and specific to MHN. Some cloud providers may require you to create the firewall rules separately and then apply them to the VM. Either way, make sure when you create the VM that you can access it via SSH.

GCP Users
First, create a firewall rule to allow ingress traffic on TCP ports 3000 and 10000. The following command can be used to do this via the command line:

```$ gcloud beta compute firewall-rules create mhn-allow-admin --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:3000,tcp:10000 --source-ranges=0.0.0.0/0 --target-tags=mhn-admin
```

This will prompt you to install the beta SDK; after doing so, the command should complete successfully. You can verify the mhn-allow-admin rule was created via the browser by looking at the VPC Network Firewall Ingress Rules.

Next, create the VM itself, which we'll call mhn-admin:

```$ gcloud compute instances create "mhn-admin" --machine-type "f1-micro" --subnet "default" --maintenance-policy "MIGRATE"  --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring.write","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --tags "mhn-admin","http-server","https-server" --image "ubuntu-1404-trusty-v20171010" --image-project "ubuntu-os-cloud" --boot-disk-size "10" --boot-disk-type "pd-standard" --boot-disk-device-name "mhn-admin"
```

Note the tags value, which controls the applicable firewall rules. The output should show both internal and external IP addresses...make note of the external IP:

```NAME       ZONE        MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP   STATUS
mhn-admin  us-west1-c  f1-micro                   10.138.0.2   35.197.22.12  RUNNING
```

```EXTERNAL IP: 35.197.22.12
```

Finally, establish SSH access to the VM via gcloud compute ssh mhn-admin (which is similar to ssh). You'll be asked to add the fingerprint the first time you connect, and you'll see the Ubuntu welcome message and a command prompt.

COMMAND: ```gcloud compute ssh mhn-admin

Updating project ssh metadata...done.
Waiting for SSH key to propagate.
Warning: Permanently added 'compute.3246857753144767930' (ECDSA) to the list of known hosts.
```

---
Milestone 2: Install the MHN Admin Application
After having established SSH access the MHN Admin VM, the following instructions can be run on the remote VM to install the application. Note: this step may take 30-40 minutes overall. These instructions were adapted from the MHN README.

First, update apt and install git:

```$ sudo apt-get update
$ sudo apt-get install git -y
```
Next, clone the MHN code into /opt, change into the clone directory, and run install.sh:

Note: the instructions below reference a fork version of the main MHN repo containing a patch for a known issue identified with the main code as of 10/28/17.

```$ cd /opt
$ sudo git clone https://github.com/RedolentSun/mhn.git
$ cd mhn
$ sudo ./install.sh
```

This will start the script running and it will take a while (approximately 20 minutes) to complete the first part, after which you'll be prompted to specify a series of values:

Do you wish to run in Debug mode? y/n : n
Superuser email: You can use any email -- this will be your username to login to the admin console.
Superuser password: Choose any password -- you'll be asked to confirm.
You can accept the default values for the rest of the values, either by hitting enter or n for any y/n prompts:

Server base url ["http://#.#.#.#"]:
Honeymap url ["http://#.#.#.#:3000"]:
Mail server address ["localhost"]:
Mail server port [25]:
Use TLS for email?: y/n n
Use SSL for email?: y/n n
Mail server username [""]:
Mail server password [""]:
Mail default sender [""]:
Path for log file ["/var/log/mhn/mhn.log"]:
The script will churn for another 15 minutes or so, and near the end, there will be two more y/n prompts, both of which you can answer n:

Would you like to integrate with Splunk? (y/n) n
Would you like to install ELK? (y/n) n
Now you should be able to load the external IP in a browser and login to the admin console via the "superuser" values you chose above. Have a look around the UI to get oriented; there won't be any data available as we've not deployed any honeypots yet.

---
Milestone 3: Create a MHN Honeypot VM
MHN supports multiple honeypots, each of which has a slightly different purpose you can read about. To start, we'll deploy Dionaea over HTTP, a honeypot used to trap malware samples.

First, create a VM for your first honeypot via your cloud provider. As before, this VM also needs to have an internet-facing IP and accessible to you via ssh (or a similar protocol), and as before, you can use a small or micro-sized VM, but it should be running Ubuntu 14.04 (trusty).

This VM will require different ports open, though which ones depend on the specific honeypot being used. To keep things simple, for this VM (and any additional honeypot VMs you create), simply allow incoming traffic from all ports and protocols. Again, this will likely require a firewall rule.

Create the VM and establish an SSH connection to it before proceeding to the next step.

GCP Users
Run the following commands on your local machine. You can either exit out of the mhn-admin shell or just open a new terminal window.

First, create the firewall rule to allow incoming traffic on all ports:

```$ gcloud beta compute firewall-rules create mhn-allow-honeypot --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=all --source-ranges=0.0.0.0/0 --target-tags=mhn-honeypot
```

```sigintz:~ sigintz$ gcloud beta compute firewall-rules create mhn-allow-honeypot --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=all --source-ranges=0.0.0.0/0 --target-tags=mhn-honeypot
Creating firewall...|Created [https://www.googleapis.com/compute/beta/projects/fb-cp-wk9/global/firewalls/mhn-allow-honeypot].
Creating firewall...done.
```

```NAME                NETWORK  DIRECTION  PRIORITY  ALLOW  DENY
mhn-allow-honeypot  default  INGRESS    1000      all
```

Now, create the VM for our honeypot, called mhn-honeypot-1:

1) Dionea
```$ gcloud compute instances create "mhn-honeypot-1" --machine-type "f1-micro" --subnet "default" --maintenance-policy "MIGRATE"  --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring.write","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --tags "mhn-honeypot","http-server" --image "ubuntu-1404-trusty-v20171010" --image-project "ubuntu-os-cloud" --boot-disk-size "10" --boot-disk-type "pd-standard" --boot-disk-device-name "mhn-honeypot-1"

NAME            ZONE        MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
mhn-honeypot-1  us-west1-a  f1-micro                   10.138.0.3   35.185.194.21  RUNNING
```

2) Dionea with HTTP
```$ gcloud compute instances create "mhn-honeypot-2" --machine-type "f1-micro" --subnet "default" --maintenance-policy "MIGRATE"  --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring.write","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --tags "mhn-honeypot","http-server" --image "ubuntu-1404-trusty-v20171010" --image-project "ubuntu-os-cloud" --boot-disk-size "10" --boot-disk-type "pd-standard" --boot-disk-device-name "mhn-honeypot-2"

NAME            ZONE        MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
mhn-honeypot-2  us-west1-b  f1-micro                   10.138.0.4   35.197.36.229  RUNNING
```

3)
```$ gcloud compute instances create "mhn-honeypot-3" --machine-type "f1-micro" --subnet "default" --maintenance-policy "MIGRATE"  --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring.write","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --tags "mhn-honeypot","http-server" --image "ubuntu-1404-trusty-v20171010" --image-project "ubuntu-os-cloud" --boot-disk-size "10" --boot-disk-type "pd-standard" --boot-disk-device-name "mhn-honeypot-3"

NAME            ZONE        MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
mhn-honeypot-3  us-west1-c  f1-micro                   10.138.0.5   35.185.225.98  RUNNING
```

Again, make note of the external IP, then go ahead and establish SSH access to the VM using gcloud compute ssh mhn-honeypot-1. You will again be asked to add the fingerprint and will see the Ubuntu 14.04 welcome message.

---
COMMAND: ```gcloud compute ssh mhn-honeypot-1```
         ```gcloud compute ssh mhn-honeypot-2```
         ```gcloud compute ssh mhn-honeypot-3
         ```

![Firewall Rules](https://github.com/acary/fb-cp-wk9/blob/master/images/firewall-rules.png?raw=true)

---
Milestone 4: Install the Honeypot Application
After having established SSH access the new honeypot VM, we need to install the honeypot application into the VM and wire it to connect back to the admin server. Fortunately, MHN makes this fairly straightforward. First, in the MHN admin console in your browser, click on Deploy in the top nav, and you'll be asked to select a script. Choose Ubuntu - Dionaea with HTTP, and you'll see a Deploy Command appear with a full deployment script below it. You can ignore the script, which is just for reference, but make a note of the Deploy Command, which is the one-line command you'll need to execute inside the honeypot VM you connected to in the last step.

---
1) Dionea:
```wget "http://35.197.22.12/api/script/?text=true&script_id=2" -O deploy.sh && sudo bash deploy.sh http://35.197.22.12 kRAWjSec
```

2) Dionea with HTTP:
```wget "http://35.197.22.12/api/script/?text=true&script_id=4" -O deploy.sh && sudo bash deploy.sh http://35.197.22.12 kRAWjSec
```

3) ElasticHoney:
```wget "http://35.197.22.12/api/script/?text=true&script_id=6" -O deploy.sh && sudo bash deploy.sh http://35.197.22.12 kRAWjSec
```

Errors encountered when trying to install p0f and ElasticHoney:

```sigintz:~ sigintz$ wget "http://35.197.22.12/api/script/?text=true&script_id=6" -O deploy.sh && sudo bash deploy.sh http://35.197.22.12 kRAWjSec
--2018-04-06 14:19:59--  http://35.197.22.12/api/script/?text=true&script_id=6
Connecting to 35.197.22.12:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1613 (1.6K) [text/html]
Saving to: ‘deploy.sh’

deploy.sh                     100%[=================================================>]   1.58K  --.-KB/s    in 0s

2018-04-06 14:19:59 (90.5 MB/s) - ‘deploy.sh’ saved [1613/1613]

+ '[' 2 -ne 2 ']'
+ server_url=http://35.197.22.12
+ deploy_key=kRAWjSec
+ wget http://35.197.22.12/static/registration.txt -O registration.sh
--2018-04-06 14:20:00--  http://35.197.22.12/static/registration.txt
Connecting to 35.197.22.12:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1430 (1.4K) [text/plain]
Saving to: ‘registration.sh’

registration.sh               100%[=================================================>]   1.40K  --.-KB/s    in 0s

2018-04-06 14:20:01 (90.9 MB/s) - ‘registration.sh’ saved [1430/1430]

+ chmod 755 registration.sh
+ . ./registration.sh http://35.197.22.12 kRAWjSec elastichoney
++ '[' 3 -ne 3 ']'
++ server_url=http://35.197.22.12
++ deploy_key=kRAWjSec
++ honeypot=elastichoney
+++ hostname
++ hostname=sigintz.lan
++ '[' -f /etc/debian_version ']'
++ '[' -f /etc/redhat-release ']'
++ echo -e 'ERROR: Unknown OS\nExiting!'
ERROR: Unknown OS
Exiting!
++ exit -1
```

![Sensor 1](https://github.com/acary/fb-cp-wk9/blob/master/images/sensors.png?raw=true)
![Sensor 2](https://github.com/acary/fb-cp-wk9/blob/master/images/sensors-2.png?raw=true)

---
So, copy the command from the browser page. It should start with wget and end with a unique token string. Execute this command inside the honeypot VM to install the Dionaea software. It shouldn't take more than a few minutes to complete. When it's done, click back over to the MHN admin console in your browser. From the top nav, choose Sensors >> View sensors and you should see the new honeypot listed.

```Name	Hostname	IP	Honeypot	UUID	Attacks
mhn-honeypot-1-dionaea
mhn-honeypot-1	35.185.194.21	dionaea	6bd7cc46-39d6-11e8-8530-42010a8a0002	0
```
---
Milestone 5: Attack!
Now for the fun part: let's attack the honeypot to make sure it's all working. You can use nmap and pass it the IP of the honeypot VM (not the IP of the MHN admin VM):

1) Dionea
```sigintz@mhn-honeypot-1:~$ nmap 35.185.194.21

Starting Nmap 6.40 ( http://nmap.org ) at 2018-04-06 20:15 UTC
Nmap scan report for 21.194.185.35.bc.googleusercontent.com (35.185.194.21)
Host is up (0.0022s latency).
Not shown: 988 closed ports
PORT     STATE    SERVICE
21/tcp   open     ftp
22/tcp   open     ssh
25/tcp   filtered smtp
42/tcp   open     nameserver
135/tcp  open     msrpc
445/tcp  open     microsoft-ds
465/tcp  filtered smtps
587/tcp  filtered submission
1433/tcp open     ms-sql-s
3306/tcp open     mysql
5060/tcp open     sip
5061/tcp open     sip-tls

Nmap done: 1 IP address (1 host up) scanned in 1.24 seconds
```

2) Dionea with HTTP:
```sigintz@mhn-honeypot-2:~$ nmap 35.197.36.229

Starting Nmap 6.40 ( http://nmap.org ) at 2018-04-06 20:29 UTC
Nmap scan report for 229.36.197.35.bc.googleusercontent.com (35.197.36.229)
Host is up (0.0028s latency).
Not shown: 986 closed ports
PORT     STATE    SERVICE
21/tcp   open     ftp
22/tcp   open     ssh
25/tcp   filtered smtp
42/tcp   open     nameserver
80/tcp   open     http
135/tcp  open     msrpc
443/tcp  open     https
445/tcp  open     microsoft-ds
465/tcp  filtered smtps
587/tcp  filtered submission
1433/tcp open     ms-sql-s
3306/tcp open     mysql
5060/tcp open     sip
5061/tcp open     sip-tls

Nmap done: 1 IP address (1 host up) scanned in 1.25 seconds
```

3)
Errors occurred with additional attempts

It should show three ports open...these are the services Dionaea is using to attract attackers. Switch back to the MHN Admin console in your browser, and from the top nav, choose Attacks. If everything goes well, you should see your IP address listed with several port scan records. This means the honeypot intercepted your attack.

You may, however, see other attacks as well, from other IPs. In fact, it shouldn't take long at all for this to happen. Port scans should start coming in at an alarming rate, from all over the world, and even with only a single honeypot deployed, MHN will start collecting lots of data. Welcome to the hostile territory that is the Internet.

![Attacks](https://github.com/acary/fb-cp-wk9/blob/master/images/attacks-report.png?raw=true)

---
Needs Moar Honeypot
You can stick with a single honeypot, but we encourage you to try some of the others that MHN supports, after reading up on each to understand a bit more about them. You'll basically repeat the process described in milestones 3-5 for each one (GCP Users: note you won't need to create any additional firewall rules after the one created in Milestone 3, only additional VM instances, which you should name uniquely, i.e. mhn-honeypot-2, mhn-honeypot-3, etc). We recommend sticking with the Ubuntu 14.04 stack unless you're very familiar with Linux. See how much data you can collect! Bonus points for capturing any malware samples.

Submission Instructions
Create a repository on Github with a README.md that includes a brief writeup about your experiment. Include the following details:

Which Honeypot(s) you deployed
Any issues you encountered
A summary of the data collected: number of attacks, number of malware samples, etc.
Any unresolved questions raised by the data collected
Additionally, include a json export of the data you collected in the repo, instructions for which can be found in the next section.

Exporting Data
The submission for this assignment will require an export of the data collected from your honeypot(s). You'll want to leave the honeypots up and running a while -- the longer the better -- and then export the data at the end of the assignment and download it to your host machine.

To export the data to json, ssh into the MHN admin VM and run the following command:

```$ mongoexport --db mnemosyne --collection session > session.json
```
connected to: 127.0.0.1
exported 12828 records
A new file, session.json, should be created in the current directory. You can download this file to your machine and check it into the GitHub repo you create for this assignment along with your README.md write up.

GCP Users
You can do the export directly from your local machine in two steps, so run the following commands on your local machine.

`$ gcloud compute ssh mhn-admin --command="mongoexport --db mnemosyne --collection session > session.json"`
connected to: 127.0.0.1
exported 2831 records

`$ gcloud compute scp mhn-admin:~/session.json ./session.json`
session.json                                                                                                                   100%  961KB 347.9KB/s   00:02
The session.json file should be downloaded successfully.

---
Cleanup
When the assignment is complete, you'll most likely want to remove the VMs created as part of the honeypot network to avoid costs incurred once your trial expires.
---
