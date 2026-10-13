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

- What is a Zeek / Cluster? - (15 mins) Fatema
- Become familiar with Cluster Configs - (45 mins) Fatema
- Zeek’s Filesystem Layout - (30 mins) Fatema
- (break 10:30am - 10:45am)
- Running Zeek and log analysis - (45mins) Christian
- Customizing Zeek - (30mins) CHristian
- (Lunch 1hr 12:00pm - 1pm)

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

# A standard bullet slide

The following items appear bit-by-bit:

* An item with text
* And another
  * with a subitem

---

# Section level 1

## Section level 2

### Section level 3

#### Section level 4

##### Section level 5

---

## A multi section slide

These all appear at once:

- An item with text
- And another

## With another section

- An item with text
- And another

---

# A two-column slide

<div class="columns">
<div>

* An item
* And another

</div>
<div>

* And yet another
* And the last

</div>
</div>

---

# Text on the left, code on the right

<div class="columns">
<div>

This is just text that explains what the thing on the right is all about.

In this case, it's some JSON.

</div>
<div>

<style scoped>
pre { font-size: 65%; }
</style>

```json
{
  "ts": 1117503119.471231,
  "uid": "CuOgIYhboLbVMhTnh",
  "id.orig_h": "192.150.186.238",
  "id.orig_p": 42762,
  "id.resp_h": "66.35.250.209",
  "id.resp_p": 80,
  "proto": "tcp",
  "service": "http",
  "duration": 6.483856916427612,
  "orig_bytes": 377,
  "resp_bytes": 10041,
  "conn_state": "SF",
  "local_orig": false,
  "local_resp": false,
  "missed_bytes": 0,
  "history": "ShADadfF",
  "orig_pkts": 11,
  "orig_ip_bytes": 957,
  "resp_pkts": 10,
  "resp_ip_bytes": 10569,
  "ip_proto": 6
}
```

</div>
</div>

---

# Text on top, code below

For example, for console output:

```console
$ while true; do ls -la | foo; done
bar
```

---

# A big image with some text

![center width:400px](zeek-logo.png)

This is one of several ways to center an image.

---

# Text on the left, image right

<div class="columns">
<div>

This is just text that explains what the thing on the right is all about.

</div>
<div>

![center width:300px](zeek-logo.png)

</div>
</div>

---

<!-- _class: chapter -->
# Moving on!

---

# Let's figure out Zeek

We can mention some inline code, like `&redef`, and we can highlight Zeek script:

```zeek
@load ./foo

event zeek_init() &priority=-5 {
    print "hello world!";
}
```
