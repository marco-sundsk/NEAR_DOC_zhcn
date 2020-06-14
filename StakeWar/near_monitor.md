## NEAR Node Self-Monitor
### Introduction
There are several ways to monitor a NEAR node process and its ENV:
* Core process log
* Local JSON RPC
* Remote JSON RPC

Combine those information, we can have a clear look at running state of a NEAR node。

### Log Analyse
####  Location of the log
A python3 tool package, called nearup, is used to start and watch the core NEAR peer process. the repo of this tool is https://github.com/near/nearup.

Have a look on that tool, we know that there are two ways to start a peer: docker and non-docker.

At docker mode, we can watch log using docker command, just notice that the name of container is `nearcore`:  
`docker logs --follow nearcore --tail 100`

At non-docker mode, having a peek at `nearuplib/nodelib.py`, we see a function called show_logs, and there are this code:  
```
command += [os.path.expanduser(f'~/.nearup/logs/{net}.log')]
```
which clearly point out where the log file is.

#### Format of the log
There are three types of log in NEAR peer log, see this pic:    
![pic](https://github.com/marco-sundsk/NEAR_DOC_zhcn/blob/master/static/images/StakeWar/blog1.png?raw=false)

and if pic is broken, you can see the following code paragraph:  
```
Jun 10 09:33:38.629  INFO near: Version: 1.0.0, Build: b30864b8, Latest Protocol: 22
Jun 10 09:33:39.023  INFO near: Opening store database at "/root/.near/betanet/data"
Jun 10 09:33:42.634  INFO stats: Server listening at ed25519:HRsGoJZtHNUfdnKGJ4EawS4zx96ajPmwaz68XjbArjvX@0.0.0.0:24567
Jun 10 09:33:43.447  WARN network: Peer stream error: Os { code: 104, kind: ConnectionReset, message: "Connection reset by peer" }
Jun 10 14:01:09.821  INFO stats: # 6963219 6qdETYwWth4RZgiUSaokar7sqPCuU9dnVao4vtcG4HGC V/86 31/31/40 peers ⬇ 237.3kiB/s ⬆ 250.8kiB/s 1.00 bps 0 gas/s CPU: 24%, Mem: 1.1 GiB
[/var/lib/buildkite-agent/.cargo/registry/src/github.com-1ecc6299db9ec823/wasmer-runtime-core-0.17.0/src/instance.rs:609] &error = User(
    Any,
)
Jun 10 14:01:19.824  INFO stats: # 6963229 GQ2WhAfJTkjM74Md7YBwmGxNxvovuWTLcJoVvARbw2KZ V/86 31/31/40 peers ⬇ 243.1kiB/s ⬆ 252.1kiB/s 1.00 bps 561.78 Ggas/s CPU: 23%, Mem: 1.1 GiB
```
So, there are:
* regular routine log, which starts with a date, a log level, and leverage color control char to embrace different parts, such as a state memo part, validator sign, p2p status, network traffic, bps and gps(g for gas), cpu mem usage;
* regular info log, which also starts with a date and a log level, but instead using color control char, it just put plain text followed the log level. those plain text is called information. They usually appear in the head of the log to show like version info and storage path and etc. Also, warning and errors which are expected are describe in that way.
* specail log, which NOT starts with a date, and often occupy multiple lines. those are unexpected warning or error emitted by runtime lib, such as a low level warning or rust panic.

#### Analyse
From perspective of self-monitor, the head and tail of the log is most important.

From the head, we can tell the up time, version, build, protocol id of the peer, as well as the storage path of the block data. It is easy to pick up those information, so we won't cover the detail in this blog.

From the tail, we can tell the latest running state of a peer. Following, we using python code to describe how to get and analyse those things.

First step, we want to fetch the latest N lines of the log, N maybe 10, which we think would be enough for us to discover latest running state:
```
import subprocess
def tail_logfile():
    ret = []
    command = 'tail -n 10 ~/.nearup/logs/betanet.log'
    try:
        output = subprocess.Popen(command, shell=True, stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
        ret = output.stdout.readlines()
    except Exception as e:
        print('%s, %s' % (type(e), e))
    return ret
```
Then, category each line, to keep at most the latest one record for each category.

To tell regular logs, we can see if the color control char is at the start of the line:
```
def is_regular(line):
    """
    return True if startwith \033[2m
    which means this line is a regular log line.
    """
    if line.startswith('\033[2m'):
        return True
    return False
```

Then, we use powful regular expression tool, split the regular log to detailed parts:
```
import re
PARTERNS = {
    'date': r'\033\[2m(.*?)\033\[0m',
    'level': r'\033\[32m(.*?)\033\[0m',
    'memo': r'\033\[1;33m(.*?)\033\[0m',
    'validator': r'\033\[1;37m(.*?)\033\[0m',
    'conn': r'\033\[1;36m(.*?) peers\033\[0m',
    'network': r'\033\[1;36m(\u2b07 [\d\.]+\w+\/s \u2b06 [\d\.]+\w+\/s)\033\[0m',
    'bpsgps': r'\033\[1;32m(.*?)\033\[0m',
    'cpumem': r'\033\[1;34m(.*?)\033\[0m'
}
def regular_parser(line):
    """
    if ret['content'] != '', then this line is a regular info log,
    otherwise, it is a regular state rutine log.
    """
    ret = {}
    ret['content'] = line.split('\033[0m')[-1].strip()
    for name, partern in PARTERNS.items():
        re_obj = re.search(partern, line)
        if re_obj:
            ret[name] = re_obj.group(1).strip()
    return ret
```
For example, to a regular routine log, we may get:
```
{
    "content": "",
    "date": "Jun 10 14:00:19.805",
    "level": "INFO",
    "memo": "# 6963175 BXH74vMw5Jn7UHK6sGVgSjzdtpC81x4JdJtDHAWFaQzF",
    "validator": "V/86",
    "conn": "31/30/40",
    "network": "⬇ 257.0kiB/s ⬆ 243.5kiB/s",
    "bpsgps": "0.90 bps 280.97 Ggas/s",
    "cpumem": "CPU: 21%, Mem: 1.1 GiB"
}
```
And for a regular info log, we may get:
```
{
    "content": "near: Version: 0.4.13, Build: 2d02b333",
    "date": "Jun 12 18:04:40.288",
    "level": "INFO"
}
```
So, obviously we can diff routine log from info log by an empty content field.。

Meanwhile, to a special log, we let the date of the most previous regular log to be his date, and keep muli-line, if any, to be one record. Like this:
```
{
    "date": "Jun 10 14:00:09.821",
    "data": "[/var/lib/buildkite-agent/.cargo/registry/src/github.com-1ecc6299db9ec823/wasmer-runtime-core-0.17.0/src/instance.rs:609] &error = User(
    Any,
)"
}
```

After that, we get:
* 0 or 1 regular routine log, which tells us the latest height, validator or not, p2p healthy, cpu and mem usage etc. We can emit self-monitor alarm through email or text msg, if any of them are abnormal.
* 0 or 1 regular info log, if the log level is warning or error, better to emit alarm too;
* 0 or 1 special log, generally it need to emit alarm immediately. Also, if a certain special log is trivial, we can make special filter to discard it.

Furthermore, the height we get form regular routine log, can be used to compare with those info from JSON RPC, which we will describe next.

### JSON RPC monitor
We can get detail info about JSON PRC of NEAR in https://docs.near.org/docs/interaction/rpc.  
Mostly, we use these two methods:
* status，which get a outline image of whole network;
* validators，which tells us detailed info about validator set.

The specific return field of those metheod can be found in the link above. Here, we just tell something about how to find abnormal from those values.

For example, we got status method's return info form local (which usually be http://127.0.0.1:3030) and remote (which usually be https://rpc.betanet.near.org) :
```
2020-06-13 19:50:01 scan status ....
local status:
betanet version 1.0.0 build b30864b8 height 7205724
remote status:
betanet version 1.0.0 build b30864b8 height 7205724
```
Comparing version and build, we know if local peer need to update;
Comparing height, if local's delay is obvious, better to emit alarm cause local peer may has stalled.

Then, comes to validators method. its return value is much more complicatied, so first reorganize and then compare. By the way, according to the return height of status method, we should prefer to use a channel which has higher height. 

After reorg, we cat this:
```
remote validator info:
CUR_SET> shards: [0], stake: 149570.036872, gen/total: 63/63
CUR_SEAT_PRICE: 120010.530474
NEXT_SET> shards: [0], stake: 149606.072015
NEXT_SEAT_PRICE: 121814.883050
PROPOSAL> NOT IN
TOP1_PROPOSAL: sl1sub stake 113953.634407
PREV_EPOCH_KICKOUT> NOT IN
EPOCH_START_HEIGHT: 7199487
```
Then we 
* see if we still in current validator set 'CUR_SET' and the situation of missing blocks, 'gen/total';
* see if we would be in next validator set 'NEXT_SET';
* compare our stake amount with next seat price 'NEXT_SEAT_PRICE', and stake more if necessary;

### Conclusion
From analysing log, comparing status and analysing validator info, we can tell a peer's health more pricisely. Then according to your alarm rules, to emit alarm through some way, such as email, auto phone or text msg. A basic self-monitor system sets up.

Furthermore, we can have those data visualized, leverage some dashboard tools like grafana. For example, we can draw a line chart of cpu and mem usage movement over time.