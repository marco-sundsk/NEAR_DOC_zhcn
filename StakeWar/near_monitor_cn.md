## NEAR 节点自监控
### 导言
对NEAR节点进程的监控，可以从以下几个方面入手：
* 节点进程的日志
* 本地JSON RPC接口
* 远端JSON RPC接口

通过对这三方面信息的综合分析，可以比较准确的得到节点的运行状态和资源占用情况。

### 日志分析
####  日志的位置
NEAR使用nearup工具启动和持续管理节点进程。nearup是个python3写的脚本工具项目。地址在：https://github.com/near/nearup 

通过分析该工具源码，可知节点进程有两种启动方式：docker和非docker方式。

在docker方式下，通过docker的方式查看日志，注意节点进程的容器名称是`nearcore`：  
`docker logs --follow nearcore --tail 100`

在非docker方式下，从nearuplib/nodelib.py文件的show_logs方法中可以看到日志文件位置：  
`command += [os.path.expanduser(f'~/.nearup/logs/{net}.log')]`

#### 日志的格式
NEAR节点日志按格式主要可分为三种，  
![pic](https://github.com/marco-sundsk/NEAR_DOC_zhcn/blob/master/static/images/StakeWar/blog1.png?raw=false)

如果图片看不到的，请看下面的文字示例：
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
* 规范例行日志 以日期开头，INFO级别，有stats描述字，且日志内容由多种不同颜色区分成信息块。一般包含状态主体、验证人标识、p2p连接情况、网络吞吐、每秒出块和gas的统计、cpu内存占用等；
* 规范信息日志 以日期开头，在日志级别之后，以无颜色文本描述信息。这种日志最有用的是开头的版本信息，以及存储路径信息。还有各种异常，一般以高于INFO的日志级别显示。
* 特殊日志 不以日期开头，可能占用多行，一般是程序异常引起的警告甚至panic信息。

#### 分析方式
从自监控的角度，一般仅需关注日志的头部和尾部。

从头部，可以获得节点进程的启动时间，版本号，build号和协议号等。还可以获得区块存储路径，可用于存储监控。头部信息比较简单，这里不再赘述。

从尾部，可获得最新的节点运行信息。这里以python代码为例，详细介绍这部分信息的获取和分析。

首先，从尾部获取比如10行日志数据，我们希望这些内容足够揭示节点的运行情况：
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
然后，我们对每行日志进行分类，判断出是哪一类日志，对每一类日志，都保留最后一次记录。
是否规范日志，可通过是否颜色控制字开头来判断：
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
对于规范日志，祭出强大的正则表达式，对各个部分进行拆分：
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
比如对一条规范例行日志，经过拆分，我们得到：
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
对一条规范信息日志，经过拆分，我们得到：
```
{
    "content": "near: Version: 0.4.13, Build: 2d02b333",
    "date": "Jun 12 18:04:40.288",
    "level": "INFO"
}
```
因此，可以通过content的内容是否为空串，区分信息日志与例行日志。

而对于特殊日志，我们将其前面一条规范日志的时间作为其时间，将其多行内容完整记录下来。
```
{
    "date": "Jun 10 14:00:09.821",
    "data": "[/var/lib/buildkite-agent/.cargo/registry/src/github.com-1ecc6299db9ec823/wasmer-runtime-core-0.17.0/src/instance.rs:609] &error = User(
    Any,
)"
}
```

这样一来，每次扫描日志后，我们都得到：
* 0或1条规范例行日志，从中可得到最新的height，是否验证人，p2p健康状况，cpu和内存占用情况，酌情进行告警。
* 0或1条规范信息日志，如果级别高于INFO，一般需要直接告警出来；
* 0或1条特殊日志，一般都需要直接告警出来，如果确认该条特殊日志无害，在通过配置专门的filter过滤处理。

而规范例行日志中的高度height信息，可以与rpc信息相互验证。

### JSON RPC监控
NEAR的JSON PRC详情可参见 https://docs.near.org/docs/interaction/rpc 
我们用到的主要是两个method：
* status，获取网络整体情况
* validators，获取验证人集合情况

这些方法的具体返回字段请见上面的链接。这里主要介绍如何通过对本地和远程调用的结果对比，发现异常。

比如对本地和远程（一般选用团队提供的rpc服务 https://rpc.betanet.near.org）的status方法返回：
```
2020-06-13 19:50:01 scan status ....
local status:
betanet version 1.0.0 build b30864b8 height 7205724
remote status:
betanet version 1.0.0 build b30864b8 height 7205724
```
我们需要对比版本和build号，看本地节点是否需要更新；
我们需要对比最新高度，如果本地落后远程过多，则本地节点大概率出了问题，需要立刻告警出来。

而，对validators方法的返回信息，需要检查和对比的项目就要复杂一些。优先选择remote的返回信息进行分析。当remote不可达，且日志分析未发现本地节点异常时，亦可采信本地rpc的返回内容：
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
* 看是否还在当前验证人集合CUR_SET中，以及漏块情况gen/total;
* 看是否在下一届验证人结合NEXT_SET中；
* 看stake量与下一届的席位价格差距，如果差距过小，需要提醒追加stake；

### 结语
通过日志分析，本地和远程rpc的对比分析，基本能精确定位本地节点是否有问题，然后根据情况进行邮件或短信告警，即可实现对NEAR 节点的基本自监控功能。

更进一步，我们还可将得到的数据，通过grafana等dashboard工具进行可视化展现。比如将cpu和内存占用以折线图的形式呈现等。