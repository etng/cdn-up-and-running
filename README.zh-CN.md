# CDN 从零开始实践

这个仓库的目标是：通过“从零”编码实现一个 CDN，来构建关于 CDN 如何工作的知识体系。我们要设计的 CDN 会使用：nginx、lua、docker、docker-compose、Prometheus、grafana 和 wrk。

我们会先创建一个单一后端服务，然后逐步扩展为一个多节点、可模拟延迟、可观测、可测试的 CDN。在每个章节中，都会讨论构建、管理和运行 CDN 时面临的挑战与权衡。

![grafana screenshot](img/4.0.1_metrics.webp "grafana screenshot")

## 什么是 CDN？

内容分发网络（Content Delivery Network，CDN）是一组在空间上分布的计算机，其目的是为那些将工作负载缓存到该网络中的系统提供高可用性和更好的性能。

## 为什么你需要 CDN？

CDN 可以帮助改进：
* 加载时间更短（更流畅的流媒体、即时可购买页面、更快的好友动态等）
* 承载流量尖峰（黑色星期五、热门流媒体首发、突发新闻等）
* 降低成本（流量卸载）
* 面向数百万用户的可扩展性

## CDN 是如何工作的？

CDN 通过把内容（媒体文件、页面、游戏、JavaScript、JSON 响应等）放到离用户更近的位置，来让服务变得更快。

当用户想要访问某个服务时，CDN 的路由系统会把请求送到“最佳”的节点上，这个节点很可能已经缓存了内容，并且离客户端更近。这里先不用太纠结“最佳”这个词的宽泛含义。希望在后续阅读过程中，你会逐渐理解什么才算最佳节点。

## CDN 技术栈

我们要构建的 CDN 依赖于：
* [`Linux/GNU/Kernel`](https://www.linux.org/) - 一个在网络能力和 IO 表现上都非常出色的内核 / 操作系统。
* [`Nginx`](http://nginx.org/) - 一个优秀的 Web 服务器，也可以作为具备缓存能力的反向代理。
* [`Lua(jit)`](https://luajit.org/) - 一门简单而强大的语言，用来给 nginx 增加功能。
* [`Prometheus`](https://prometheus.io/) - 一个具备维度化数据模型、灵活查询语言和高效时序数据库的系统。
* [`Grafana`](https://github.com/grafana/grafana) - 一个开源的分析与监控工具，可接入包括 Prometheus 在内的多种数据源。
* [`Containers`](https://www.docker.com/) - 用于打包、部署和隔离应用的技术，我们会使用 docker 和 docker compose。

# Origin：后端服务

Origin 是内容被创建出来的系统，或者至少是 CDN 内容的来源。我们要构建的示例服务会是一个简单直接的 JSON API。这个后端服务也可以返回图片、视频、JavaScript、HTML 页面、游戏资源，或者任何你想交付给客户端的内容。

我们会使用 Nginx 和 Lua 来设计这个后端服务。这也是一个很好的机会来介绍 Nginx 和 Lua，因为它们在后续内容中会频繁出现。

> **注意**：后端服务可以用任何你喜欢的语言来编写。

## Nginx 快速介绍

Nginx 是一个会遵循其[配置文件](http://nginx.org/en/docs/beginners_guide.html#conf_structure)运行的 Web 服务器。配置文件以[指令](http://nginx.org/en/docs/dirindex.html)为核心。指令是一种用于设置 nginx 属性的简单结构。指令有两种类型：**简单指令**和**块指令（上下文）**。

**简单指令**由指令名加参数组成，并以分号结尾。

```nginx
# 语法：<name> <parameters>;
# 示例
add_header X-Header AnyValue;
```

**块指令**遵循相同的模式，但不是以分号结尾，而是用大括号包裹。块指令内部还可以继续包含其他指令。这个块也被称为上下文（context）。

```nginx
# 语法：<name> <parameters> <block>
location / {
  add_header X-Header AnyValue;
}
```

Nginx 使用 worker（进程）来处理请求。[nginx 的架构](https://www.aosabook.org/en/nginx.html)对其性能起着关键作用。

![simplified workers nginx architecture](img/simplified_workers_nginx_architecture.webp "simplified workers nginx architecture")

> **注意**：虽然多个 worker 共享一个 accept 队列是常见模式，但也存在其他用于[负载均衡传入请求](https://blog.cloudflare.com/the-sad-state-of-linux-socket-balancing/)的模型。

## 后端服务配置

让我们来看一下后端 JSON API 的 nginx 配置。直接看它如何工作，会更容易理解。

```nginx
events {
  worker_connections 1024;
}
error_log stderr;

http {
  access_log /dev/stdout;

  server {
    listen 8080;

    location / {
      content_by_lua_block {
        ngx.header['Content-Type'] = 'application/json'
        ngx.say('{"service": "api", "value": 42}')
      }
    }
  }
}
```

你能看懂这个配置在做什么吗？无论如何，我们还是逐条拆开来解释。

[`events`](http://nginx.org/en/docs/ngx_core_module.html#events) 提供了[连接处理配置](http://nginx.org/en/docs/events.html)的上下文，而 [`worker_connections`](http://nginx.org/en/docs/ngx_core_module.html#worker_connections) 定义了单个 worker 进程能够打开的最大并发连接数。

```nginx
events {
  worker_connections 1024;
}
```

[`error_log`](http://nginx.org/en/docs/ngx_core_module.html#error_log) 用于配置错误日志。这里我们只是把所有错误输出到 stdout（准确说是错误输出）。

```nginx
error_log stderr;
```

[`http`](http://nginx.org/en/docs/http/ngx_http_core_module.html#http) 提供了一个根上下文，用于设置所有的 HTTP/HTTPS 服务器。

```nginx
http {}
```

[`access_log`](http://nginx.org/en/docs/http/ngx_http_log_module.html#access_log) 用于配置访问日志的路径（以及可选的格式等）。

```nginx
access_log /dev/stdout;
```

[`server`](http://nginx.org/en/docs/http/ngx_http_core_module.html#server) 用于设置一个服务器的根配置，也就是我们将在其中定义该服务器的具体行为。一个 `http` 上下文中可以包含多个 `server` 块。

```nginx
server {}
```

在 `server` 内部，我们可以通过 [`listen`](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen) 指令来控制[服务器监听请求](http://nginx.org/en/docs/http/request_processing.html)的地址和 / 或端口。

```nginx
listen 8080;
````

在 `server` 配置中，我们可以使用 [`location`](http://nginx.org/en/docs/http/ngx_http_core_module.html#location) 指令来指定路由。它会为匹配到的请求路径提供特定配置。

```nginx
location / {}
```

在这个 location 中（顺带一提，`/` 会处理所有请求），我们将使用 Lua 来生成响应。有一个叫做 [`content_by_lua_block`](https://github.com/openresty/lua-nginx-module#content_by_lua_block) 的指令，它提供了一个运行 Lua 代码的上下文。

```nginx
content_by_lua_block {}
```

最后，我们会使用 Lua 和基础的 [Nginx Lua API](https://github.com/openresty/lua-nginx-module#nginx-api-for-lua) 来定义目标行为。

```lua
-- ngx.header 用于设置当前响应头
ngx.header['Content-Type'] = 'application/json'
-- ngx.say 会写入响应体
ngx.say('{"service": "api", "value": 42}')
```

请注意，大多数指令都有自己的作用域。例如，`location` 只在 `location`（递归地）和 `server` 上下文中适用。

![directive restriction](img/nginx_directive_restriction.webp "directive restriction")

> **注意**：从现在开始，我们不会再逐条解释每一个新增指令，只会描述与当前章节最相关的部分。

## CDN 1.0.0 演示时间

让我们看看目前做了什么。

```bash
git checkout 1.0.0 # 切回这个特定配置
docker-compose run --rm --service-ports backend # 运行容器并暴露服务端口
http http://localhost:8080/path/to/my/content.ext # 访问服务，我用的是 httpie，你也可以用 curl 或其他工具

# 你应该会看到 JSON 响应 :)
```

## 添加缓存能力

为了让后端服务可以被缓存，我们需要设置缓存策略。我们将使用 HTTP 头 [Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control) 来定义希望的缓存行为。

```Lua
-- 我们希望内容被缓存 10 秒，或者使用传入的 max_age（例如 /path/to/service?max_age=40 表示缓存 40 秒）
ngx.header['Cache-Control'] = 'public, max-age=' .. (ngx.var.arg_max_age or 10)
```

如果你愿意，也可以检查一下返回响应头中的 `Cache-Control`。

```bash
git checkout 1.0.1 # 切回这个特定配置
docker-compose run --rm --service-ports backend
http "http://localhost:8080/path/to/my/content.ext?max_age=30"
```

## 添加指标

查看日志很适合调试。但当流量增大之后，几乎不可能仅靠日志来理解服务的运行状况。为了解决这个问题，我们会使用 [VTS](https://github.com/vozlt/nginx-module-vts)，这是一个为 nginx 增加指标采集能力的模块。

```nginx
vhost_traffic_status_zone shared:vhost_traffic_status:12m;
vhost_traffic_status_filter_by_set_key $status status::*;
vhost_traffic_status_histogram_buckets 0.005 0.01 0.05 0.1 0.5 1 5 10; # bucket 单位是秒
```

[`vhost_traffic_status_zone`](https://github.com/vozlt/nginx-module-vts#vhost_traffic_status_zone) 用于设置指标所需的内存空间。[`vhost_traffic_status_filter_by_set_key`](https://github.com/vozlt/nginx-module-vts#vhost_traffic_status_filter_by_set_key) 用于按某个变量对指标进行分组（例如这里我们决定按 `status` 分组）；最后，[`vhost_traffic_status_histogram_buckets`](https://github.com/vozlt/nginx-module-vts#vhost_traffic_status_histogram_buckets) 提供了以秒为单位划分 bucket 的方式。我们决定创建从 `0.005` 到 `10` 秒的 bucket，因为这有助于我们构建百分位指标（`p99`、`p50` 等）。

```nginx
location /status {
  vhost_traffic_status_display;
  vhost_traffic_status_display_format html;
}
```

我们还必须通过一个 location 暴露这些指标。这里会使用 `/status`。

```bash
git checkout 1.1.0
docker-compose run --rm --service-ports backend
# 打开 http://localhost:8080/status/format/html 可以看到 8080 服务的状态信息
# 注意 VTS 还支持其他格式，例如 status/format/prometheus，这对我们后面会很有帮助
```

![nginx vts status page](img/metrics_status.webp "nginx vts status page")

有了指标之后，我们就能运行压测，并观察配置改动到底有没有带来更好的性能。

> **注意**：你可以[把指标归入自定义命名空间](https://github.com/leandromoreira/cdn-up-and-running/commit/105f54a27d1b58b88659789ae024d70c89d4a478)。当一个 location 会因上下文不同而表现不同的时候，这会很有用。

## 重构 nginx 配置

随着配置文件变大，它也会越来越难理解。Nginx 提供了一个很好用的 [`include`](http://nginx.org/en/docs/ngx_core_module.html#include) 指令，可以让我们拆分出局部配置文件，并在根配置文件中引入它们。

```diff
-    location /status {
-      vhost_traffic_status_display;
-      vhost_traffic_status_display_format html;
-    }
+    include basic_vts_location.conf;

```

我们可以把 location 抽出来，按相似性组织配置，或者做任何有意义的拆分。[Lua 代码也可以做类似的事情](https://github.com/openresty/lua-nginx-module#lua_package_path)。

```diff
       content_by_lua_block {
-        ngx.header['Content-Type'] = 'application/json'
-        ngx.header['Cache-Control'] = 'public, max-age=' .. (ngx.var.arg_max_age or 10)
-
-        ngx.say('{"service": "api", "value": 42, "request": "' .. ngx.var.uri .. '"}')
+        local backend = require "backend"
+        backend.generate_content()
       }
```

所有这些改动都是为了提高可读性，同时也促进复用。


# CDN：位于后端之前的一层

## 代理转发

到目前为止，我们做的事情其实都还不算 CDN。现在是时候真正开始构建 CDN 了。为此，我们会再创建一个 nginx 节点，只需增加少量新指令，把 `edge`（CDN）节点连接到 `backend` 节点。

![backend edge architecture](img/edge_backend.webp "backend edge architecture")

这里其实没什么复杂的，只是定义了一个 [`upstream`](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#upstream) 块，其中有一个指向 `backend` 端点的 server。在 location 中，我们不再直接返回内容，而是通过刚创建好的 [`proxy_pass`](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass) 指向 upstream。

```nginx
upstream backend {
  server backend:8080;
  keepalive 10;  # 连接池，便于复用
}

server {
  listen 8080;

  location / {
    proxy_pass http://backend;
    add_header X-Cache-Status $upstream_cache_status;
  }
}
````

我们还增加了一个新响应头（`X-Cache-Status`），用来表明[缓存是否被命中](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#variables)。
* **HIT**：当内容已经在 CDN 中时，`X-Cache-Status` 应该返回 hit。
* **MISS**：当内容还不在 CDN 中时，`X-Cache-Status` 应该返回 miss。

```bash
git checkout 2.0.0
docker-compose up
# 我们仍然可以直接从 backend 获取内容
http "http://localhost:8080/path/to/my/content.ext"
# 但我们真正想做的是通过 edge（CDN）访问内容
http "http://localhost:8081/path/to/my/content.ext"
```

## 缓存

当我们尝试获取内容时，会发现 `X-Cache-Status` 响应头并没有出现。看起来 edge 节点始终都会去请求 backend。这显然不是 CDN 应有的工作方式，对吧？

```log
backend_1     | 172.22.0.4 - - [05/Jan/2022:17:24:48 +0000] "GET /path/to/my/content.ext HTTP/1.0" 200 70 "-" "HTTPie/2.6.0"
edge_1        | 172.22.0.1 - - [05/Jan/2022:17:24:48 +0000] "GET /path/to/my/content.ext HTTP/1.1" 200 70 "-" "HTTPie/2.6.0"
````

当前的 edge 只是把客户端请求简单代理到 backend。那我们缺了什么？单纯的“简单代理”到底有没有用？其实也有，比如你可能希望做限流、认证、授权、TLS 终止，或者作为多个服务的网关，但这不是我们这里的目标。

我们需要使用 [`proxy_cache_path`](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_path) 指令，在 nginx 中创建一个缓存区域。它会设置缓存内容存放的路径、共享内存 `key_zone`，以及 `inactive`、`max_size` 等策略，从而控制缓存的行为方式。

```nginx
proxy_cache_path /cache/ levels=2:2 keys_zone=zone_1:10m max_size=10m inactive=10m use_temp_path=off;
```

配置好缓存之后，我们还必须通过 [`proxy_cache`](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache) 指向正确的 zone（通过 `proxy_cache_path keys_zone=<name>:size`），并通过 [`proxy_pass`](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass) 连接到前面创建的 upstream。

```nginx
location / {
    # ...
    proxy_pass http://backend;
    proxy_cache zone_1;
}
```

缓存还有另一个重要方面，由 [`proxy_cache_key`](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_key) 指令控制。
当客户端向 nginx 请求内容时，它会（高度简化地）：

* 接收请求（假设是：`GET /path/to/something.txt`）
* 对缓存键的值应用一次 md5 哈希（假设缓存键是 `uri`）
  * md5("/path/to/something.txt") => `b3c4c5e7dc10b13dc2e3f852e52afcf3`
    * 你可以在终端里验证：`echo -n "/path/to/something.txt" | md5`
* 检查对应内容（哈希 `b3c4..`）是否已经缓存
* 如果已缓存，就直接返回对象；否则就从 backend 抓取内容
  * 同时还会把内容保存在本地（内存和磁盘）中，以便后续请求复用

让我们通过 lua 指令 [`set_by_lua_block`](https://github.com/openresty/lua-nginx-module#set_by_lua_block) 创建一个名为 `cache_key` 的变量。它会为每个进入的请求，把 `cache_key` 填充为 `uri` 的值。除此之外，我们还需要更新 [`proxy_cache_key`](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_key)。

```nginx
location / {
    set_by_lua_block $cache_key {
      return ngx.var.uri
    }
    # ...
    proxy_cache_key $cache_key;
}
```

> **注意**：如果用 `uri` 作为缓存键，那么 `http://example.com/path/to/content.ext` 和 `http://example.edu/path/to/content.ext` 这两个请求（假设共用同一个缓存代理）会被视为同一个对象。如果你不显式提供缓存键，nginx 会使用一个合理的**默认值**：`$scheme$proxy_host$request_uri`。

现在我们终于可以看到缓存正常工作了。

```bash
git checkout 2.1.0
docker-compose up
http "http://localhost:8081/path/to/my/content.ext"
# 第二次请求应该会直接从 CDN 返回内容，不再访问 backend
http "http://localhost:8081/path/to/my/content.ext"
```

![cache hit header](img/cache_hit.webp "cache hit header")

## 监控工具

通过命令行查看缓存是否生效并不高效。更好的方式是用专门的工具。我们会用 **Prometheus** 来抓取所有服务器上的指标，再用 **Grafana** 把 Prometheus 收集到的数据展示成图表。

![instrumentalization architecture](img/metrics_architecture.webp "instrumentalization architecture")

Prometheus 的配置大致如下。

```yaml
global:
  scrape_interval:     10s # Prometheus 每 10 秒抓取一次目标
  evaluation_interval: 10s
  scrape_timeout: 2s

  external_labels:
      monitor: 'CDN'

scrape_configs:
  - job_name: 'prometheus'
    metrics_path: '/status/format/prometheus'
    static_configs:
      - targets: ['edge:8080', 'backend:8080'] # 将通过 scrap_path 抓取的服务列表
```

然后，我们还需要为 Grafana 添加一个 Prometheus 数据源。

![grafana source](img/add_source.webp "grafana source")

并设置正确的 Prometheus 服务器地址。

![grafana source set](img/set_source.webp "grafana source set")

## 模拟工作负载（延迟）

后端服务器现在是人工生成响应的。我们会使用 lua 增加模拟延迟，使它更接近真实世界场景。这里我们将用[百分位数](https://www.mathsisfun.com/data/percentiles.html)来建模延迟。

```lua
percentile_config={
    {p=50, min=1, max=20,}, {p=90, min=21, max=50,}, {p=95, min=51, max=150,}, {p=99, min=151, max=500,},
}
```

我们先随机选一个 1 到 100 之间的数字，再根据对应的 `percentile profile` 在 `min` 到 `max` 的范围内再做一次随机。最后，对这个时长执行 [`sleep`](https://github.com/openresty/lua-nginx-module#ngxsleep)。

```lua
local current_percentage = random(1, 100) -- 决定这次请求落在哪个百分位
-- 假设我们抽到了 94
-- 那么就会使用 p90 这个 percentile_config
local sleep_duration = random(p90.min, p90.max)
sleep(sleep_seconds)
```

这个模型让我们可以自由尝试去模拟更接近[真实世界中观测到的延迟](https://research.google/pubs/pub40801/)。

## 负载测试

我们会运行一些压测，来更好地理解正在构建的这个方案。Wrk 是一个 HTTP 基准测试工具，并且可以通过 Lua 动态配置。这里我们随机选一个 1 到 100 之间的数字，并请求对应资源。

```lua
request = function()
  local item = "item_" .. random(1, 100)

  return wrk.format(nil, "/" .. item .. ".ext")
end
```

下面这条命令会用两个线程、10 个连接，对系统持续测试 10 分钟（600 秒）。

```bash
wrk -c10 -t2 -d600s -s ./src/load_tests.lua --latency http://localhost:8081
```

当然，你也可以在自己机器上运行：

```bash
git checkout 2.2.0
docker-compose up

# 运行测试
./load_test.sh

# 去 Grafana 看看系统表现如何
http://localhost:9091
```

`wrk` 输出如下。总共发出了 **37k** 个请求，其中 **674** 个失败。

```bash
Running 10m test @ http://localhost:8081
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   218.31ms  236.55ms   1.99s    84.32%
    Req/Sec    35.14     29.02   202.00     79.15%
  Latency Distribution
     50%  162.73ms
     75%  350.33ms
     90%  519.56ms
     99%    1.02s
  37689 requests in 10.00m, 15.50MB read
  Non-2xx or 3xx responses: 674
Requests/sec:     62.80
Transfer/sec:     26.44KB
```

Grafana 显示，在某个时间点，`edge` 共响应了 **68** 个请求，其中 **16** 个请求继续访问了 `backend`。此时的[缓存命中效率](https://www.cloudflare.com/learning/cdn/what-is-a-cache-hit-ratio/)是 **76%**，有 1% 的请求延迟超过 **3.6s**，5% 超过 **786ms**，中位数大约是 **73ms**。


> 请注意：延迟指标会高度依赖 bucket 的大小，而且[使用 histogram 来衡量性能](https://medium.com/mercari-engineering/have-you-been-using-histogram-metrics-correctly-730c9547a7a9)不一定总是合适。

![grafana result for 2.2.0](img/2.2.0_metrics.webp "grafana result for 2.2.0")

## 通过测试学习：修改缓存 TTL（max age）

这个项目应该鼓励你去做实验：调整参数、运行压测、观察结果。我认为这种循环是很好的学习方式。接下来我们试试看修改缓存行为会发生什么。

### 1 秒

把缓存有效期设为 1 秒。

```lua
request = function()
  local item = "item_" .. random(1, 100)

  return wrk.format(nil, "/" .. item .. ".ext?max_age=1")
end
```

运行测试后，结果是：只有 16k 个请求，并且出现了 773 个错误。

```
Running 10m test @ http://localhost:8081
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   378.72ms  254.21ms   1.46s    68.40%
    Req/Sec    15.11      9.98    90.00     74.18%
  Latency Distribution
     50%  396.15ms
     75%  507.22ms
     90%  664.18ms
     99%    1.05s
  16643 requests in 10.00m, 6.83MB read
  Non-2xx or 3xx responses: 773
Requests/sec:     27.74
Transfer/sec:     11.66KB
```

我们还注意到，缓存命中率显著下降到了 **23%**，并且更多请求泄漏到了 backend。

![grafana result for 2.2.1 1 second](img/2.2.1_metrics_1s.webp "grafana result for 2.2.1 1 second")

### 60 秒

如果反过来，把缓存过期时间提高到整整 1 分钟，会怎样？

```lua
request = function()
  local item = "item_" .. random(1, 100)

  return wrk.format(nil, "/" .. item .. ".ext?max_age=60")
end
```

运行测试后，这次的结果是：45k 个请求，551 个错误。

```bash
Running 10m test @ http://localhost:8081
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   196.27ms  223.43ms   1.79s    84.74%
    Req/Sec    42.31     34.80   242.00     78.01%
  Latency Distribution
     50%   79.67ms
     75%  321.06ms
     90%  494.41ms
     99%    1.01s
  45695 requests in 10.00m, 18.79MB read
  Non-2xx or 3xx responses: 551
Requests/sec:     76.15
Transfer/sec:     32.06KB
```

我们可以看到更好的**缓存效率（80% 对比 23%）**和**吞吐量（45k 对比 16k 请求）**。

![grafana result for 2.2.1 60 seconds](img/2.2.1_metrics_60s.webp "grafana result for 2.2.1 60 seconds")

> **注意**：缓存时间更长有助于提升性能，但代价是内容可能更陈旧。

## 精调：缓存锁、陈旧内容、超时、网络

对许多小规模负载来说，使用 Nginx、Linux 等的默认配置通常已经足够。但如果你的目标更大，就不可避免地需要根据自身需求来精调 CDN。

Web 服务器的调优过程非常庞杂。从 [`nginx/Linux 进程 socket`](https://blog.cloudflare.com/the-sad-state-of-linux-socket-balancing/) 的管理，到 [`linux 网络队列`](https://github.com/leandromoreira/linux-network-performance-parameters)，再到 [`io`](https://serverfault.com/questions/796665/what-are-the-performance-implications-for-millions-of-files-in-a-modern-file-sys) 如何影响性能，都会牵涉其中。[应用与操作系统](https://nginx.org/en/docs/http/ngx_http_core_module.html#sendfile)之间存在大量共生关系，并且会直接影响性能，例如[通过 ktls 减少用户态上下文切换](https://docs.kernel.org/networking/tls-offload.html)。

你会读很多 man page，大多是在调 timeout 和 buffer。测试循环可以帮助你建立对自己想法的信心，流程如下：

* 先提出一个假设，或者观察到某个异常，想测试某个参数值
  * 每次只聚焦于一组相关参数
* 设置新值
* 运行测试
* 将结果与旧参数下的同一台服务器进行对比

> **注意**：在本地做测试适合学习，但大多数时候你最终只会信任生产环境结果。做好部分发布的准备，并对比旧系统 / 旧配置与新测试参数的差异。

你有没有注意到，这些错误都和超时有关？看起来 `backend` 的响应时间比 `edge` 愿意等待的时间还要长。

```log
edge_1        | 2021/12/29 11:52:45 [error] 8#8: *3 upstream timed out (110: Operation timed out) while reading response header from upstream, client: 172.25.0.1, server: , request: "GET /item_34.ext HTTP/1.1", upstream: "http://172.25.0.3:8080/item_34.ext", host: "localhost:8081"
```

为了解决这个问题，我们可以尝试增大代理超时时间。这里还用到了一个很好用的指令 [`proxy_cache_use_stale`](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_use_stale)，它允许 nginx 在遇到 `error`、`timeout`，甚至在更新缓存期间，返回 `stale content`（陈旧内容）。

```nginx
proxy_cache_lock_timeout 2s;
proxy_read_timeout 2s;
proxy_send_timeout 2s;
proxy_cache_use_stale error timeout updating;
```

当我们继续研究代理缓存时，又注意到了一个指令：[`proxy_cache_lock`](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_lock)。它会把多个用户对同一内容的请求合并成一个上游请求，一次只让一个请求去 `upstream` 拉取内容。这个机制通常被称为 [coalescing](https://cloud.google.com/cdn/docs/caching#request-coalescing)。

```nginx
proxy_cache_lock on
```

![caching lock](img/cache_lock.webp "caching lock")

运行测试后我们观察到，超时错误减少了，但吞吐量也下降了。为什么？也许是锁竞争导致的。这个特性的最大收益，是避免 backend 出现[惊群效应](https://alexpareto.com/2020/06/15/thundering-herds.html)。流量从 **6k** 降到了 **3k**，请求数也从 **16** 降到了 **8**。

![grafana result for test 3.0.0](img/3.0.0_metrics.webp "grafana result for test 3.0.0")

## 从正态分布到长尾分布

我们一直以来做压测时都假设请求服从[正态分布](https://en.wikipedia.org/wiki/Normal_distribution)，但现实并非如此。在生产环境中，更常见的是[大部分请求集中在少数几个热门对象上](https://en.wikipedia.org/wiki/Long_tail)。为了更贴近这种情况，我们会调整代码：随机选取 1 到 100 之间的数字，然后决定它是否对应热门内容。

```lua
local popular_percentage = 96 -- 96% 的用户请求前 5 个热门内容
local popular_items_quantity = 5 -- 热门内容数量
local max_total_items = 200 -- 客户端可能请求的总内容数

request = function()
  local is_popular = random(1, 100) <= popular_percentage
  local item = ""

  if is_popular then -- 如果是热门请求，就从热门内容中选一个
    item = "item-" .. random(1, popular_items_quantity)
  else -- 否则就从其余内容中任选一个
    item = "item-" .. random(popular_items_quantity + 1, popular_items_quantity + max_total_items)
  end

  return wrk.format(nil, "/path/" .. item .. ".ext")
end
```

> **注意**：我们本来也可以用[某个公式](https://firstmonday.org/ojs/index.php/fm/article/view/1832/1716)来建模长尾分布，但对于这个仓库的目的来说，这样的近似已经足够了。

现在，分别在 `proxy_cache_lock` 为 `off` 和 `on` 的情况下再测试一次。

### 长尾分布下 `proxy_cache_lock` 关闭
![grafana result for test 3.1.0](img/3.1.0_metrics.webp "grafana result for test 3.1.0")
### 长尾分布下 `proxy_cache_lock` 开启
![grafana result for test 3.1.1](img/3.1.1_metrics.webp "grafana result for test 3.1.1")

两者结果非常接近，虽然 `lock off` 仍然略好一些。这个特性是否值得上线，可能还需要通过生产环境验证。

> **注意**：`proxy_cache_lock_timeout` 很危险，但又是必须的；一旦超过配置时间，所有请求都会打到 backend。

## 路由挑战

到目前为止，我们测试的只有一个 edge，但现实中会有数百个节点。更多 edge 节点对于扩展性、弹性，以及让响应更接近用户都很有必要。而多节点的引入也带来了另一个挑战：客户端需要想办法知道该去哪个节点获取内容。

我们可以通过很多方式来解决这个问题，下面会尝试探索其中一部分。

### 负载均衡

负载均衡器会把客户端请求分发到所有 edge 节点上。

#### 轮询（Round-robin）

轮询是一种负载均衡策略：它维护一个有序的 edge 列表，每次请求都选择下一个服务器，到了列表末尾再回到开头。

```nginx
# 在 nginx 中，如果不显式指定策略，默认就是 weighted round-robin
# http://nginx.org/en/docs/http/ngx_http_upstream_module.html#upstream
upstream backend {
  server edge:8080;
  server edge1:8080;
  server edge2:8080;
}

server {
  listen 8080;

  location / {
    proxy_pass http://backend;
    add_header X-Edge LoadBalancer;
  }
}
```

`round-robin` 的优点是什么？请求会被比较均匀地分散到所有服务器上。某些服务器可能更慢，或者某些响应会堆积很多请求；这时还有 [`least_conn`](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#least_conn) 这样的策略，它会一并考虑连接数。

它的缺点是什么？它不感知缓存，也就是说多个客户端可能会因为命中了未缓存的服务器而遭遇更高延迟。

> [更多关于何时使用或避免 `rr` 策略，可参考这里。](https://github.com/leandromoreira/cdn-up-and-running/issues/10)

```bash
# 演示时间
git checkout 4.0.0
docker-compose up
./load_test.sh
```

![round-robin grafana](img/4.0.0_metrics.webp "round-robin grafana")

> **注意**：这里的负载均衡器本身就是单点故障。[Facebook 有一场很好的演讲](https://www.youtube.com/watch?v=bxhYNfFeVF4)，解释了他们如何构建一个具备弹性、可维护、可扩展的负载均衡器。

#### 一致性哈希（Consistent Hashing）

既然缓存感知对于 CDN 很重要，那么直接使用 round-robin 就比较困难。有一种叫做 [`consistent hashing`](https://en.wikipedia.org/wiki/Consistent_hashing) 的负载均衡方法，可以通过选择某个信号（例如 `uri`）并将其映射到哈希表中，来稳定地把所有请求发送到同一台服务器，从而尽量解决这个问题。

Nginx 中也有对应的指令，叫做 [`hash`](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#hash)。

```nginx
upstream backend {
  hash $request_uri consistent;
  server edge:8080;
  server edge1:8080;
  server edge2:8080;
}

server {
  listen 8080;

  location / {
    proxy_pass http://backend;
    add_header X-Edge LoadBalancer;
  }
}
```

`consistent hashing` 的优点是什么？它通过策略提高了缓存命中的概率。

它的缺点是什么？想象一下，某个单独的内容（视频、游戏）突然爆红，此时就可能只有少数几台服务器承担绝大部分用户请求。

> **注意**：[Consistent Hashing With Bounded Load](https://medium.com/vimeo-engineering-blog/improving-load-balancing-with-a-new-consistent-hashing-algorithm-9f1bd75709ed) 就是为了解决这个问题而诞生的。

```bash
# 演示时间
git checkout 4.0.1
docker-compose up
./load_test.sh
```

![consistent hashing grafana](img/4.0.1_metrics.webp "consistent hashing grafana")

> **注意**：最开始我用过一个 Lua 库，因为我当时以为一致性哈希只在商业版 nginx 中可用。

#### 负载均衡器瓶颈

负载均衡器至少有两个问题（除了它还是一个 [SPoF](https://en.wikipedia.org/wiki/Single_point_of_failure)）：

* 网络出口带宽 - 负载均衡器的输入 / 输出带宽能力，至少要大于等于其后所有服务器带宽之和。
  * 一种做法是使用 [DSR](https://www.loadbalancer.org/blog/yahoos-l3-direct-server-return-an-alternative-to-lvs-tun-explored/) 或 [307](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/307)。
* 分布式 edge - 节点可能在地理上分散得很远，这会让负载均衡器难以有效工作。

### 网络可达性

在前面的负载均衡章节中，我们看到的很多问题其实都与网络可达性有关。这里我们会讨论一些解决方式，以及它们各自的优缺点。

#### API

我们可以引入一个 `API（cdn routing）`，客户端只有先查询这个 API，才能知道去哪里获取内容（也就是某个具体的 edge 节点）。客户端可能还需要自己处理故障切换。

> **注意**：如果在软件层解决这个问题，可以结合多种方案的优点：先用 `consistent hashing` 做基础分发，当某个内容变得热门之后，再切换成[更自然的分布方式](https://brooker.co.za/blog/2012/01/17/two-random.html)。

#### DNS

我们也可以使用 DNS。它和 API 很相似，但这里我们依赖的是 DNS 缓存 TTL。在这种情况下，故障切换会更难处理。

#### Anycast

我们还可以使用单一的 [域名 / IP，并在所有节点所在位置广播这个 IP](https://en.wikipedia.org/wiki/Anycast)，把“为某个用户找到最近节点”这件事交给[网络路由协议](https://www.youtube.com/watch?v=O6tCoD5c_U0)来完成。

## 其他话题

我们还没有讨论 CDN 中很多重要的方面，例如：

* [Peering](https://www.peeringdb.com/) - CDN 会把节点 / 内容部署在 ISP、公共对等互联点和私有对等互联点上。
* 安全 - CDN 经常遭受攻击，例如 DDoS、[缓存投毒](https://youst.in/posts/cache-poisoning-at-scale/)等。
* [缓存策略](https://netflixtechblog.com/netflix-and-fill-c43a32b490c0) - 有些情况下，不是 edge 从 backend 拉内容，而是 backend 主动把内容推送到 edge。
* [租户](https://en.wikipedia.org/wiki/Multitenancy) / 隔离 - CDN 会在同一批节点上承载多个客户，因此隔离是必须的。
  * 指标、缓存区域、配置（缓存策略、backend）等等都需要隔离。
* Token - CDN 通常会提供某种形式的[令牌保护](https://en.wikipedia.org/wiki/JSON_Web_Token)，防止未授权客户端访问内容。
* [健康检查（故障检测）](https://youtu.be/1TIzPL4878Q?t=782) - 用来判断某个节点是否正常工作。
* HTTP Headers - 客户端往往希望添加一些头（例如 [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)），有时甚至需要动态添加。
* [地理封锁](https://github.com/leev/ngx_http_geoip2_module#example-usage) - 为了省钱或满足合同限制，你的 CDN 可能会根据用户所在区域执行某些策略。
* Purging - 能够[从缓存中清除内容](https://docs.nginx.com/nginx/admin-guide/content-cache/content-caching/#purging-content-from-the-cache)。
* [Throttling](https://github.com/leandromoreira/nginx-lua-redis-rate-measuring#use-case-distributed-throttling) - 限制并发请求数。
* [Edge computing](https://leandromoreira.com/2020/04/19/building-an-edge-computing-platform/) - 能够在边缘节点上以过滤器形式运行代码来处理托管内容。
* 等等等等……

## 结论

希望你已经对 CDN 的工作方式有了一些了解。它是一个复杂的工程，高度依赖于节点离客户端有多近，以及你能否在考虑缓存因素的前提下合理分配负载，从而同时应对流量高峰与低谷。
