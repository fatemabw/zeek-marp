---
theme: zeek
_class: lead
---

![bg](https://marp.app/assets/hero-background.svg)

![width:250px](./zeek-logo.png)
# Introduction to Zeek
### A Zeek training
*NSF Cybersecurity Summmit, Oct 27 2026*

---

# Who Are We

Christian Kreibich
- Zeek Technical Lead and LT member
- Engineer at Corelight
- 20+ years of network security, research, and engineering

Fatema Bannat Wala
- Security Engineer @ ESnet/LBNL
- 11+ yrs of hands-on network defense
- Zeek LT Member / Zeek Training Subgroup Lead

---

# Agenda

- What is a Zeek / Cluster? - Fatema
- Become familiar with Cluster Configs - Fatema
- Zeek’s Filesystem Layout - Fatema
- Running Zeek in a Cluster - Fatema
(break 10:30am - 10:45am)
- Invoking Zeek via cmd line - Christian
- Zeek Package Manager - Christian
- Zeek log analysis - Christian

(Lunch 12:00pm - 1pm)

---

# Housekeeping

- Zeek’s Official Documentation: https://docs.zeek.org
- Join Zeek Slack channel #training
- Slide deck for this training can be found here: https://github.com/zeek/zeek-training/tree/master/Intro-to-Zeek
- Highly recommend to have a copy of slide deck open in a browser tab, for easy copy paste of the commands in exercises.

---

# Docker/Zeek usage

- Setup your Zeek Docker image
  - Grab the latest Zeek LTS image from https://hub.docker.com/u/zeek
```console
# docker pull zeek/zeek:lts
# docker run -it zeek/zeek:lts /bin/bash
```
- Inside the container, set up resources used during this training:
```console
# git clone https://github.com/zeek/zeek-training
# ln -sfn /usr/local/zeek zeek
# ln -sfn /zeek-training/Intro-to-Zeek/training-res training-res
# cd zeek-training
# ./setup.sh all
```
- To run additional shells in the container:
```console
  # docker exec -it <container name> /bin/bash 
  ```
- Verify that Zeek runs correctly: 
```console
# zeek --version
```

---
![center width:400px](zeek.png)

---
![center width:400px](zeek-pipeline.png)

---
# What is a Zeek Cluster?

---

## Why/What?

- A Zeek Cluster is a set of system processes, all running Zeek, with a set of scripts that adjust their purpose
  - As opposed to running a single Zeek process, reading from a pcap, for instance
  - Can run on one or more host environments — machines, VMs, containers, etc
- To scale Zeek to keep up with more traffic, one adds processes (“nodes”) to the cluster
- Supporting tools handle tasks like monitoring processes and log rotation
- If you need 24x7 network monitoring, clustering is your answer

---
![center width:400px](zeek-cluster.png)

---

# Cluster Componenets

- Manager
- Worker
- Proxy
- Logger
---

# Cluster Componenets

- Manager
  - Every cluster has exactly one
  - Writing logs from workers (when not using separate logger nodes)
  - Useful for smaller volumes or limited hardware
  - Can also run scripts when you want to conduct analysis across multiple workers
  - Natural point in the cluster for centralized state

---

# Cluster Componenets

- Worker
  - Analyze network packets to create (most) events & logs
  - Run scripts to alert on interesting traffic

---

# Cluster Componenets

- Proxy
  - State sharing among cluster workers
  - Start with one, then add more if load is an issue

---

# Cluster Componenets

- Logger
  - Write logs via configurable writer plugins for files, Kafka, Elastic, etc
  - Easy support to load balancing among multiple logger nodes
    - Log rates vary depending on traffic and writer technology
    - Very roughly, up to 100K events/sec per logger should work
  - Good way to scale log I/O in high volume environments
  - Defaults to TSV-format file logging, but easy to switch to JSON or alternatives

---

# Performance Tuning

- LBL 100G Paper
  - Good architecture overviews
- SEPTun-Mark-II
  - Originally written for Suricata, but all concepts on packet queues, NUMA, OS tuning, etc apply
- CPU Pinning
  - Avoid using hyperthreaded cores for workers
- Fine tuning via ethtool - tro/tso/rx/tx/ etc..
- Consider Shunting - Dumbno script (for Arista N/W LB)

---

# Zeek's Filesystem Layout

---
# Filesystem Layout

- Not an exhaustive tour!
- Default top level path: /opt/zeek/ or /usr/local/zeek
  - bin/
  - etc/
  - include/ - ignoring today
  - lib/  lib64/ var/ - ignoring today
  - logs/
  - share/
  - spool/

---
# Filesystem Layout

- bin/
   - zeek-cut
    - Extract columns from Zeek’s TSV logs (non-JSON)
    - Convert Unix epoch
  - zeek
    - Zeek executable with various options to run
  - zkg
    - Zeek package manager executable
  - zeekctl
    - Cluster management tool

---
# Filesystem Layout

- etc/
  - network.cfg
    - Define your local networks
  - node.cfg
    - Configure a cluster
    - Extra features (lb_procs, pin_cpu, lb_method)
  - zeekctl.cfg
    - Configure cluster management tool
    - Set shorter rotation time

---
# Filesystem Layout

- logs/
  - current
    - Default Log Path (cluster-mode)
  - current directory
    - Default Log Path (CLI mode, logs overwritten each run)
  - YYYY-MM-DD
    - Logs archived to this dir,rotated hourly by default

---
# Filesystem Layout

 - spool/
  - state.db
    - Cluster worker state (zeekctl)
  - zeekctl-config.sh
    - Cluster management variables (zeekctl)
  - worker-<n>
    - .pid
    - .cmdline
    - .env_vars
  - logger
    - logs (“current” is a symlink)
  - extract_files/node/
    - Extracted (“carved”) files, when configured

--- 
# Filesystem Layout

- share/
  - zeek/base/
    - Base scripts
  - zeek/policy/
    - Additional scripts
  - zeek/site/
    - Site-specific scripts
    - Packages installed via zkg, our package manager
  - hilti/ and spicy/
    - Analyzers written in Spicy, our parser generator
  - zeekctl/
    - Helper scripts for cluster management (e.g. log post-processing)

---
# Exercise 1 - fire up a cluster (10 mins)

- Edit node.cfg - comment out the standalone section, and uncomment the logger, proxy, manager and worker-1 sections.
```console
$ vi /zeek/etc/node.cfg
[worker-1]
type=worker
host=localhost
interface=af_packet::eth0
lb_method=custom
lb_procs=2
pin_cpus=0,1
```
- Set shorter rotation time
```console
$ vi /zeek/etc/zeekctl.cfg
LogRotationInterval = 300 //(5mins)
```

---
- turn off the checksum with Zeek redef
```console
$ vi /zeek/share/zeek/site/local.zeek
     redef ignore_checksums = T;
```
- Then run:
```console 
$ zeekctl deploy
```
- Make sure cluster is running:
```console
$ zeekctl status
```
- Create some traffic for the cluster to sniff
```console
$ wget google.com
$ ls /zeek/logs/current/
```
- Send few pcaps to the sniffing interface:
```console
$ tcpreplay -i eth0 --mbps=500 /training-res/traces/*.pcap
$ ls /zeek/logs/current/
```
- Stop the cluster
```console
$ zeekctl stop
```
---

# MORNING BREAK [15 mins] 10:30-10:45am

---
# Invoking Zeek via cmd line

- For offline analysis use zeek -r on a packet capture:
```console
 # zeek -r /zeek-training/traces/zeek-doc/quickstart.pcap
 # rm -f *.log
```
- Zeek cmd to analyze live traffic
```console
 # zeek -C -i eth0
 # listening on eth0
```
- Generate some test traffic in another terminal
```console
 $ docker exec -it zeek-tutorial /bin/bash
 # ping example.com -c 1
 # exit
```
---
# Providing Script Values

- Modify tunable variables directly from the command line
```console
 # zeek -r /zeek-training/traces/zeek-doc/quickstart.pcap LogAscii::use_json=T
```
- Change the FTP analyzer to capture passwords and log the necessary FTP command:
```console
 # zeek -Cr /zeek-training/traces/zeek-testing/ftp
/ftp-password-pass-command.pcap FTP::default_capture_password=T  ‘FTP::logged_commands+={“PASS”}’

 # cat ftp.log
```
---
# Providing Script Values

- Use zeek -e and provide arbitrary quoted Zeek script segments:
```console
 # zeek -e "print \"hello\"; print\"there\";"
 hello
 there

```
- Read script code from standard input:
```console
 # echo "print \"Hello, world\!\";" | zeek
 Hello, world!
```
---
# Analyzing Zeek Logs

- Zeek’s default log format is tab-separated values, or TSV
```console
 # zeek -r /zeek-training/traces/zeek-doc/quickstart.pcap
```
Using zeek-cut for more condensed log format
```console
 # cat conn.log | zeek-cut -m
```
Filtering on interesting fields 
```console
 # cat conn.log | zeek-cut -m uid id.orig_h id.orig_p id.resp_h id.resp_p
```
Looking at the weird.log file, notice the uid field 
```console
 # cat weird.log | zeek-cut -m uid name
```
convert the epoch timestamps to a human readable format:
```console
 # cat conn.log | zeek-cut -m ts
 # cat conn.log | zeek-cut -m -d ts
```
---
# Zeek Log Schemas

- Zeek’s logschema package helps you understand your Zeek logs. 
- It produces log schemas that detail your installation’s set of logs and their fields. 
- For each field, the schemas provide rich metadata including name, type, and docstrings. 
- They can also explain the source of a field, such as the specific script or the name of the Zeek package that added it. 
- Log schemas are also a great way to understand how and whether your logs change when you upgrade to a newer version of Zeek.

---
# Zeek Log Schemas
```console
# zkg install logschema
The following packages will be INSTALLED:
  zeek/zeek/logschema (v2.0.0)

Proceed? [Y/n] y
Running unit tests for "zeek/zeek/logschema"
Installing "zeek/zeek/logschema"
Installed "zeek/zeek/logschema" (v2.0.0)
Loaded "zeek/zeek/logschema"

# zeek logschema/export/jsonschema packages
```
---
# Zeek Log Schemas Example

Your local directory will now contain a JSON Schema description for each of your installation’s logs. If we want to find more about Zeek’s DNS log, we can do the following:
```console
#cat zeek-dns-log.schema.json | jq
```
Let’s use one of the testing PCAPs to make some DNS logs:
 ```console
# zeek -r /zeek-training/traces/zeek-testing/dns/naptr.pcap
```
Then, we can find these in our DNS logs and examine the logs via zeek-cut:
```console
# cat dns.log | zeek-cut -m uid answers
```
---
# Zeek JSON Logs

Zeek also allows emitting logs in JSON form, rather than as TSV files.
```console
# zeek -r /zeek-training/traces/zeek-doc/quickstart.pcap LogAscii::use_json=T
# jq . -c conn.log

#jq -c '[."id.orig_h", ."id.resp_h"]' conn.log
```
You can do the same in a cluster by modifying your site’s local.zeek:
```console
vim $PREFIX/share/zeek/site/local.zeek
```
Then add a line (anywhere!) that loads the policy/tuning/json-logs script:
```console
@load policy/tuning/json-logs
```
---
# Zeek logs IR Example

- Looking at some malicious traffic
```console
# zeek -Cr /training-res/traces/2022-02-23-traffic-analysis-exercise.pcap local
```
- Find suspicious connections
```console
# grep -v "^#" conn.log | zeek-cut id.orig_h id.resp_h id.resp_p service conn_state | grep S0 | sort | uniq -c | sort -rn | head
```
- Find external connections
```console
# zeek-cut id.orig_h id.resp_h id.resp_p service < conn.log | sort | uniq -c | sort -rn | head -30
```
- Once you find a suspicious IP, look at EVERYTHING about it
```console
# SUSPECT="<suspicious-ip>"
# grep "$SUSPECT" *.log
```
---
# Package Manager (zkg)

- Package Manager
  - zkg
  - https://packages.zeek.org

- Can add new scripts or analyzers
  - Analyzer examples :
    - QUIC
    - PerfSonar
  - Scripts : 
    - BZAR 
    - DHCP OUI data

---
# Let’s Use zkg!

- List all zeek packages and install a few
```console
# zkg refresh
# zkg list all
# zkg install cve-2020-0601
```
- Check if they got installed
```console
# zkg list installed
# ls /zeek/share/zeek/site
```
- Let’s load the packages
  uncomment @load packages in site/local.zeek

---
# Let’s Use zkg!

- zkg load/unload and info
```console
# zkg unload cve-2020-0601
# zkg info cve-2020-0601
```
- Remove and uninstall all the packages
```console
# zkg purge
# zkg list installed
```
- Other useful zkg commands
```console
remove, pin, unpin, upgrade, refresh
```
---
# Exercise #3: Zkg 

Running Zeek with zkg installed packages, specifically with JA3.
```console
# zkg install ja3
# zeekctl deploy
# tcpreplay -i eth0 --mbps=500 /training-res/traces/extract.pcap
# less /zeek/logs/current/ssl.log
# cat ssl.log | zeek-cut -d ja3 ja3s
```
---
# Exercise #4: Zkg 

Extract a file from a file transfer connection using file-extract zkg package 
```console
# zkg install file-extraction
```
Add all file types extraction to the config of the package:
```console
# vi /zeek/share/zeek/site/packages/file-extraction/config.zeek
redef path = "/extracted_files/";
@load ./plugins/extract-all-files.zeek


# mkdir /extracted_files
# zeekctl deploy 
# tcpreplay -i eth0 --mbps=500 /training-res/traces/extract.pcap
# ls /extracted_files/ 
# less files.log
```
---
# Lunch Break
---