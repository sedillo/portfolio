---
title: Debugging Patterns for HPC
description: Debugging is all about patterns... learning, detecting and using patterns
weight: 2
---

We'll present a common pattern needed when debugging issues with a cluster of nodes

1. Basic node connection
1. Gathering node metrics
1. Node filtering based on metrics

We use the basic top command which gives us an idea of user+sys cpu usage. But the concept can be expanded for any metric
```bash
top -b -n1 | grep 'Cpu(s)' | awk '{print $2 + $4}'
```

# Basic node connections

Most important thing is preparation, setting up the following will make it easier to debug:

* Break up clusters into subnets
* Define nodes with tools like ansible

## Using subnet to find available hosts

This command will create a list of available hosts to debug
```bash
nmap -sn 192.168.100.1-256 | grep "Nmap scan report for" | awk '{print $5}' > up-hosts.txt
```

After this you should check up-hosts.txt against original hosts lists to see if any are not reachable.

## Using ansible to find available hosts

```bash
ansible all -i hosts -m ping
```

Ansible will display a summary of what hosts are down.

# Gathering node metrics

Now that we can gather node metrics we can use tools to gather metrics for all hosts

With bash
```bash
pssh -h up-hosts.txt -i "top -b -n1 | grep 'Cpu(s)' | awk '{print $2 + $4}'"
```

With ansible
```bash
ansible all -i hosts -m shell -a "top -b -n1 | grep 'Cpu(s)' | awk '{print \$2 + \$4}'"
```

# Node filtering based on metrics (ex CPU > 90%)

## CLI (imperative) example

With bash we can first create a script that prints hostname IP if CPU usage is above 90%
```bash
#!/bin/bash
USAGE=$(top -b -n1 | grep 'Cpu(s)' | awk '{print $2 + $4}')
if (( $(echo "$USAGE > 90" | bc -l) )); then
  echo "$(hostname -I) has high CPU utilization: $USAGE%"
fi
```

Then we use scp and ssh (or pscp and pssh) to copy and execute scripts
```bash
pscp -h up-hosts.txt myscript.sh /usr/bin/myscript.sh
pscp -h up-hosts.txt -i /usr/bin/myscript.sh
```

## Ansible (declarative) example

With ansible we could create a playbook to filter CPUs with high CPU usage
```yml
---
- hosts: all
  gather_facts: no
  tasks:
    - name: get cpu utilization
      command: top -b -n1 | grep 'Cpu(s)' | awk '{print $2 + $4}'
      register: cpu_util
      changed_when: false

    - name: Check if CPU utilization is above 90%
      debug:
        msg: "{{ ansible_host }} has high CPU utilization: {{ cpu_util.stdout }}%"
      when: cpu_util.stdout | float > 90
```
