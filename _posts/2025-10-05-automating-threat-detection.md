---
layout: post
title: "Automating Threat Detection with Nmap & Nessus"
date: 2025-10-05
category: "Network Security"
reading_time: 12
excerpt: "Learn how to build an automated vulnerability scanning pipeline using Nmap for reconnaissance and Nessus for in-depth vulnerability assessment."
---

## Introduction

Manual vulnerability scanning is time-consuming and error-prone. This guide demonstrates how to automate threat detection using Nmap and Nessus with Python integration.

## Architecture Overview

Our automated pipeline consists of:

1. **Reconnaissance**: Nmap discovers live hosts and services
2. **Vulnerability Scanning**: Nessus performs deep analysis
3. **Reporting**: Python generates actionable reports

## Setting Up the Environment

### Prerequisites

```bash
# Install required tools
sudo apt-get update
sudo apt-get install nmap python3 python3-pip

# Install Python libraries
pip3 install python-nmap
pip3 install requests  # For Nessus API
```

### Nessus API Setup

Register for a Nessus API key from Tenable and configure:

```python
NESSUS_URL = "https://localhost:8834"
ACCESS_KEY = "your-access-key"
SECRET_KEY = "your-secret-key"
```

## Phase 1: Network Discovery with Nmap

### Basic Host Discovery

```python
import nmap

def discover_hosts(network):
    nm = nmap.PortScanner()
    nm.scan(hosts=network, arguments='-sn')
    
    hosts = []
    for host in nm.all_hosts():
        if nm[host].state() == 'up':
            hosts.append(host)
    return hosts

# Usage
live_hosts = discover_hosts('192.168.1.0/24')
print(f"Found {len(live_hosts)} live hosts")
```

### Service Detection

```python
def scan_services(host):
    nm = nmap.PortScanner()
    nm.scan(host, '1-1000', '-sV -sC')
    
    services = []
    for proto in nm[host].all_protocols():
        ports = nm[host][proto].keys()
        for port in ports:
            service = nm[host][proto][port]
            services.append({
                'port': port,
                'name': service['name'],
                'version': service.get('version', 'unknown')
            })
    return services
```

## Phase 2: Nessus Integration

### Creating a Scan

```python
import requests
import json

class NessusScanner:
    def __init__(self, url, access_key, secret_key):
        self.url = url
        self.headers = {
            'X-ApiKeys': f'accessKey={access_key}; secretKey={secret_key}',
            'Content-Type': 'application/json'
        }
    
    def create_scan(self, name, targets):
        payload = {
            'uuid': 'template-uuid',  # Get from /editor/scan/templates
            'settings': {
                'name': name,
                'text_targets': targets
            }
        }
        
        response = requests.post(
            f'{self.url}/scans',
            headers=self.headers,
            data=json.dumps(payload),
            verify=False
        )
        return response.json()['scan']['id']
    
    def launch_scan(self, scan_id):
        requests.post(
            f'{self.url}/scans/{scan_id}/launch',
            headers=self.headers,
            verify=False
        )
```

## Phase 3: Automated Pipeline

### Complete Workflow

```python
def automated_threat_detection(network):
    # Step 1: Discover hosts
    print("[*] Discovering live hosts...")
    hosts = discover_hosts(network)
    
    # Step 2: Scan services
    print("[*] Scanning services...")
    all_services = {}
    for host in hosts:
        all_services[host] = scan_services(host)
    
    # Step 3: Launch Nessus scan
    print("[*] Launching Nessus scan...")
    nessus = NessusScanner(NESSUS_URL, ACCESS_KEY, SECRET_KEY)
    targets = ','.join(hosts)
    scan_id = nessus.create_scan('Automated Scan', targets)
    nessus.launch_scan(scan_id)
    
    # Step 4: Wait and retrieve results
    print("[*] Waiting for scan to complete...")
    # Add polling logic here
    
    return all_services

# Run the pipeline
results = automated_threat_detection('192.168.1.0/24')
```

## Scheduling with Cron

Automate daily scans:

```bash
# Add to crontab
0 2 * * * /usr/bin/python3 /path/to/scanner.py >> /var/log/scanner.log 2>&1
```

## Best Practices

1. **Rate Limiting**: Don't overwhelm network devices
2. **Authorization**: Always get permission before scanning
3. **Logging**: Maintain detailed logs for compliance
4. **Notification**: Alert teams of critical findings

## Conclusion

Automated threat detection reduces response time and ensures consistent coverage. Combine Nmap's speed with Nessus's depth for comprehensive security monitoring.