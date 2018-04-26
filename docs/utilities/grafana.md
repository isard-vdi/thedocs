# Monitoring with Grafana

Grafana is a web ui with configurable panels that can graph data send
to a carbon service listening on 2003/tcp for key/value timestamps.

CARBON: Recevies key/value timestamps and inserts on Whisper database.
WHISPER: Key/Value database.
GRAPHITE: Api that brings timed series from data stored in whisper.
GRAFANA: Web UI that allows configuring graphs from graphite data series.

## Set up Grafana environment

Using docker-compose (https://docs.docker.com/compose/install/) you can
start a complete grafana setup:

```
git clone https://github.com/kamon-io/docker-grafana-graphite.git
cd docker-grafana-graphite
docker-compose up -d
```

You can access grafana web ui on port 80 (user admin, password admin)

## Carbon client script

In order to monitor services, disk IO, etc... you need a client script
that will run on client machines and will send key/value timestamps to
your grafana host on port 2003/tcp.

Here you have an example script:

```python
import time
import subprocess,os
import socket
import platform
import struct 
import re

HOSTNAME                = platform.node().split('.')[0]
SLEEP_SECONDS           = 1
WRITEBOOST_CACHE_DEVICE = ''        # Or empty string ''
DISKS                   = 'sda,sb'  # Comma spaced list of block devs or empty string ''
NETS                    = 'eh0,lo'  # Comma spaced list of net devs or empty string ''
CARBON_SERVER           = 'mygrafana.mydomain.com'
CARBON_PORT             = 2003


#### SYSTEM INFO PARSER
def get_sys():
    sysdata = {}
    with open('/proc/loadavg', 'r') as f:
        for line in f:
            splitLine = line.split()
    sysdata['load']=splitLine[0]
    with open('/proc/meminfo', 'r') as f:
        i=0
        for line in f:
            splitLine = line.split()
            if i==0: sysdata['memTotal']=splitLine[1]
            if i==1: sysdata['memFree']=splitLine[1]
            if i==2: sysdata['memAvail']=splitLine[1]
            if i==3: sysdata['memCached']=splitLine[1]
            if i==3: break
            i+=1
    return sysdata

#### DM-WRITEBOOST CACHE PARSER
def get_writeboost():
        if WRITEBOOST_CACHE_DEVICE == '': return False
        raw_lines=subprocess.check_output(['dmsetup status '+WRITEBOOST_CACHE_DEVICE+' | wbstatus'], shell=True).decode('utf-8').split('\n')
        data={}
        for line in raw_lines:
                if '=' in line:
                        splitLine = line.split('=')
                        data[splitLine[0].strip().replace('# of','nr').replace(' ','_')] = splitLine[1].strip()
        return data



#### NFS Stats
def get_nfs():
        raw_lines=subprocess.check_output(['nfsstat','-l']).split('\n')
        data={}
        for line in raw_lines:
          try:
                if '-' in line[1]: continue
                line=chunkstring(line,13)
                data[line[1].strip()] = line[2].replace(':','').strip()
          except:
                continue
        return data

def chunkstring(string, length):
    return list(string[0+i:length+i].strip() for i in range(0, len(string), length))
    

#### DSTAT disk and net metrics    
def execute(cmd):
    popen = subprocess.Popen(cmd, stdout=subprocess.PIPE, universal_newlines=True)
    for stdout_line in iter(popen.stdout.readline, ""):
        yield stdout_line 
    popen.stdout.close()
    return_code = popen.wait()
    if return_code:
        raise subprocess.CalledProcessError(return_code, cmd)
        
def cu(value):
    try:
        if 'B' in value: return value.split('B')[0]
        if 'k' in value:
            return str(1024*int(value.split('k')[0]))
        if 'M' in value:
            return str(1048576*int(value.split('M')[0]))
        if 'G' in value:
            return str(1073741824*int(value.split('M')[0]))
    except Exception as e:
        return '0'
    return value

def get_dstat(line):
    line=line.strip()
    k1=line.split('|')
    data={}
    i=0
    if DISKS is not '':
        disks_data=k1[0].split(':')
        for d in DISKS.split(','):
            values=disks_data[i].split()
            data['disks.'+d+'.read']=cu(values[0])
            data['disks.'+d+'.write']=cu(values[1])
            i=i+1
    i=0
    if NETS is not '':
        nets_data=k1[len(k1)-1].split(':')
        for n in NETS.split(','):
            values=nets_data[i].split()
            data['nets.'+n+'.recv']=cu(values[0])
            data['nets.'+n+'.send']=cu(values[1])
            i=i+1
    return data

#### 
def send_stats2carbon(sysdata,dstat,cache):
    sock=False
    while sock is False:
        try:
            sock = socket.socket()
            sock.connect((CARBON_SERVER, CARBON_PORT))
        except Exception as e:
            print('Carbon server '+CARBON_SERVER+' or carbon port '+str(CARBON_PORT)+' not reacheable!')
            sock=False
            time.sleep(5)
    timestamp = int(time.time())
    msg=''
    for key,value in sysdata.items():
        msg+='%s.sysdata.%s %s %d\n' % (HOSTNAME, key, value, timestamp)
    if dstat is not False:
        for key,value in dstat.items():
            msg+='%s.dstat.%s %s %d\n' % (HOSTNAME, key, value, timestamp)
    if cache is not False:
        if cache is not False:
            for key,value in cache.items():
                msg+='%s.writeboost.%s %s %d\n' % (HOSTNAME, key, value, timestamp)
    sock.sendall(msg.encode('utf-8'))
    sock.close()

def escape_ansi(line):
    ansi_escape = re.compile(r'(\x9B|\x1B\[)[0-?]*[ -/]*[@-~]')
    return ansi_escape.sub('', line)


## Main
if DISKS is not '' or NETS is not '':
    cmd=cmd=["dstat","-dnD",DISKS,"-N",NETS,"--nocolor","--noheaders"]
    if DISKS == '': cmd=["dstat","-nN",NETS,"--nocolor","--noheaders"]
    if NETS  == '': cmd=["dstat","-dD",DISKS,"--nocolor","--noheaders"]
    for line in execute(cmd):
        line=escape_ansi(line)
        if '-' in line or 'read' in line: 
            continue
        send_stats2carbon(get_sys(),get_dstat(line),get_writeboost())

else:
    while True:
        send_stats2carbon(get_sys(),False,get_writeboost())


```

## Carbon client systemd service

First try your carbon-client.py script and when it runs with your configured variables then you can install systemd service and enable it:

```bash
vi /etc/systemd/system/carbon.service 
[Unit]
Description=Carbon Service
After=multi-user.target

[Service]
Type=idle
ExecStart=/usr/bin/python3 /opt/carbon/carbon-client.py

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl start carbon
systemctl enable carbon
```