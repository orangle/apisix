[English](benchmark.md)

### 测试环境

我们使用阿里云ECS来测试，服务器规格为 `12 vCPU 12 GiB (I/O Optimization) ecs.ic5.3xlarge` , `Intel Xeon(Skylake) Platinum 8163 | 2.5GHz`

我们最多只使用 4 核去运行 APISIX, 剩下的 4 核用与系统和压力测试工具 [wrk](https://github.com/wg/wrk)。

### 测试反向代理: **一个 upstream, 不开启插件**

我们把 APISIX 当做反向代理来使用，不开启任何插件，响应体的大小为 1KB。

#### 一个 worker 进程

```
$ wrk -d 20 -c 32 http://127.0.0.1:9080/hello
Running 20s test @ http://127.0.0.1:9080/hello
  2 threads and 32 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.39ms  175.73us  13.71ms   93.15%
    Req/Sec    11.55k   184.73    12.16k    69.40%
  462008 requests in 20.10s, 1.79GB read
Requests/sec:  22985.71
Transfer/sec:     91.32MB

$ wrk -d 20 -c 32 http://127.0.0.1:9080/hello
Running 20s test @ http://127.0.0.1:9080/hello
  2 threads and 32 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.39ms  165.06us  10.83ms   92.81%
    Req/Sec    11.53k   183.77    12.02k    68.41%
  461403 requests in 20.10s, 1.79GB read
Requests/sec:  22955.47
Transfer/sec:     91.20MB
```

#### 四个 worker 进程

```
$ wrk -d 20 -c 64 http://127.0.0.1:9080/hello
Running 20s test @ http://127.0.0.1:9080/hello
  2 threads and 64 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     0.93ms  127.93us   4.45ms   83.13%
    Req/Sec    34.43k   424.13    35.46k    72.50%
  1370440 requests in 20.00s, 5.32GB read
Requests/sec:  68521.03
Transfer/sec:    272.23MB

$ wrk -d 20 -c 64 http://127.0.0.1:9080/hello
Running 20s test @ http://127.0.0.1:9080/hello
  2 threads and 64 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     0.93ms  124.69us   4.22ms   83.18%
    Req/Sec    34.36k   403.64    35.84k    71.00%
  1367054 requests in 20.00s, 5.30GB read
Requests/sec:  68350.62
Transfer/sec:    271.55MB
```

如果你想在自己的机器上做压力测试，应当在 80 端口再启动一个 Nginx 服务。
```shell
curl http://127.0.0.1:9080/apisix/admin/routes/1 -X PUT -d '
{
    "uri": "/hello",
    "plugins": {
    },
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "127.0.0.1:80": 1
        }
    }
}'
```

### 测试反向代理: **一个 upstream, 开启 2 个插件**

我们把 APISIX 当做反向代理来使用，开启限速和 prometheus 插件，响应体的大小为 1KB。

#### 一个 worker 进程

```
$ wrk -d 20 -c 32 http://127.0.0.1:9080/hello
Running 20s test @ http://127.0.0.1:9080/hello
  2 threads and 32 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.62ms  219.87us  14.83ms   92.96%
    Req/Sec     9.94k   198.08    10.40k    81.25%
  395594 requests in 20.00s, 1.56GB read
Requests/sec:  19779.35
Transfer/sec:     79.94MB

$ wrk -d 20 -c 32 http://127.0.0.1:9080/hello
Running 20s test @ http://127.0.0.1:9080/hello
  2 threads and 32 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.62ms  192.41us  12.99ms   92.52%
    Req/Sec     9.94k   249.30    10.31k    79.00%
  395617 requests in 20.00s, 1.56GB read
Requests/sec:  19780.46
Transfer/sec:     79.95MB
```

#### 四个 worker 进程

```
$ wrk -d 20 -c 32 http://127.0.0.1:9080/hello
Running 20s test @ http://127.0.0.1:9080/hello
  2 threads and 32 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   553.23us   98.07us   3.77ms   80.85%
    Req/Sec    28.89k   356.39    29.80k    72.39%
  1155592 requests in 20.10s, 4.56GB read
Requests/sec:  57493.30
Transfer/sec:    232.37MB

$ wrk -d 20 -c 64 http://127.0.0.1:9080/hello
Running 20s test @ http://127.0.0.1:9080/hello
  2 threads and 64 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.12ms  151.09us   6.33ms   83.88%
    Req/Sec    28.70k   343.48    30.18k    72.64%
  1147728 requests in 20.10s, 4.53GB read
Requests/sec:  57101.09
Transfer/sec:    230.78MB
```

如果你想在自己的机器上做压力测试，应当在 80 端口再启动一个 Nginx 服务。
```shell
curl http://127.0.0.1:9080/apisix/admin/routes/1 -X PUT -d '
{
    "uri": "/hello",
    "plugins": {
        "limit-count": {
            "count": 2000000000000,
            "time_window": 60,
            "rejected_code": 503,
            "key": "remote_addr"
        },
        "prometheus": {}
    },
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "127.0.0.1:80": 1
        }
    }
}'
```

### 和 Openresty 对比

单位: QPS

|用例 |	apisix|  1 worker|	empty 1 worker|	百分比 |
| -- |  --|   ---|  ---| --- | 
|1 upstream + 0 plugin|	23819.32|	28071.08|	84.8%
|1 upstream + 0 plugin|	23928.15|	28072.27|	85.2%
|1 upstream + 2 plugin|	19519.52|		
|1 upstream + 2 plugin|	19833.96|

对比测试使用的脚本:
https://gist.github.com/membphis/05064f2edc6fb4081c6af04fac43ba49

