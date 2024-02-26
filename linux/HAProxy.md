# HAProxy

### HAProxy简介

HAProxy是一款开源的高性能负载均衡器（Load Balancer）和代理服务器，广泛用于提高网站的可用性和性能，通过分发流量到多个服务器来处理。HAProxy支持TCP和HTTP应用，能够处理高并发连接，常用于保护Web服务器，实现数据库读写分离，以及各种高可用性（HA）配置中。

### 核心特性

- **高性能和稳定性**：能够处理数十万并发连接，广泛应用于高流量网站。
- **灵活的路由选择**：支持基于内容的路由决策，如HTTP头、SSL/TLS会话ID等。
- **健康检查**：定期检查后端服务器的健康状态，自动剔除不健康的服务器。
- **安全特性**：提供SSL/TLS终端解密、请求限速、黑名单等安全特性。
- **详细的统计信息**：提供丰富的实时监控和统计数据，方便分析和故障排查。

### 1. 安装HAProxy

在CentOS上，你可以通过`yum`包管理器安装HAProxy。首先，打开终端或通过SSH连接到你的服务器，然后执行以下命令来安装HAProxy：

```bash
sudo yum install haproxy -y
```

### 2. 配置HAProxy

安装完成后，下一步是配置HAProxy以满足你的负载均衡需求。HAProxy的配置文件通常位于`/etc/haproxy/haproxy.cfg`。使用文本编辑器编辑此文件，例如，使用`nano`或`vi`：

```bash
sudo vi /etc/haproxy/haproxy.cfg
```

以下是一个基本的配置示例，它将HAProxy配置为HTTP负载均衡器，将流量分发到两个后端Web服务器：

```haproxy
global
    log /dev/log local0
    maxconn 4000
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000ms
    timeout client  50000ms
    timeout server  50000ms

frontend http_front
    bind *:80
    stats uri /haproxy?stats
    default_backend web_servers

backend web_servers
    balance roundrobin
    server web1 192.168.1.101:80 check
    server web2 192.168.1.102:80 check
```

> ### HAProxy配置文件结构
>
> HAProxy的配置文件通常位于`/etc/haproxy/haproxy.cfg`，分为几个主要部分：
>
> - **global**：定义全局设置，影响HAProxy的整体运行环境。
> - **defaults**：为其他部分提供默认配置，可以被特定配置覆盖。
> - **frontend**：定义监听的端口和IP，以及将请求转发到哪个backend。
> - **backend**：包含处理实际请求的服务器（即真实服务器）的列表和负载均衡策略。
> - **listen**：`frontend`和`backend`的组合，用于特定应用的简化配置。
>
> ### 关键配置指令详解
>
> #### Global 部分
>
> - `log`：指定日志输出。通常配置为系统日志地址。
> - `maxconn`：定义HAProxy可以同时处理的最大连接数。
> - `user`和`group`：指定HAProxy进程运行的用户和组。
> - `daemon`：以守护进程模式运行HAProxy。
>
> #### Defaults 部分
>
> - `log`：定义默认的日志输出，通常会被frontend或backend的配置覆盖。
> - `mode`：设置工作模式，`http`或`tcp`，影响某些选项和检查的可用性。
> - `timeout connect`：设置连接到服务器的超时时间。
> - `timeout client`：客户端空闲连接超时时间。
> - `timeout server`：服务器空闲连接超时时间。
>
> #### Frontend 部分
>
> - `bind`：指定IP地址和端口号，HAProxy在此监听入站连接。
> - `default_backend`：如果没有其他规则匹配，则将流量发送到指定的backend。
> - `acl`：定义访问控制列表规则，用于基于请求内容做决策。
> - `use_backend`：根据`acl`规则选择使用的backend。
>
> #### Backend 部分
>
> - `balance`：负载均衡算法，如`roundrobin`、`leastconn`等。
> - `server`：定义后端服务器，包括IP、端口和选项（如`check`启用健康检查）。
> - `http-check expect`：定义健康检查成功的条件。
>
> #### 监听（Listen）部分
>
> `listen`部分是`frontend`和`backend`的简化组合，适用于需要在同一配置块中定义监听地址和后端服务器的场景。

### 3. 启动HAProxy

配置完成后，启动HAProxy服务：

```bash
sudo systemctl start haproxy
```

确保HAProxy随系统启动：

```bash
sudo systemctl enable haproxy
```

### 4. 验证配置

你可以通过访问HAProxy服务器的IP或域名（如果已配置）在Web浏览器中查看效果。如果一切配置正确，你应该能够看到流量被分发到后端服务器。

此外，你可以通过访问设置的统计页面（在本例中为`http://your_server_ip/haproxy?stats`）查看负载均衡器的状态和统计信息。

### 5. 调试和日志

如果遇到问题，可以查看HAProxy的日志以获取更多信息。根据`global`部分的配置，日志通常会被发送到`/var/log/haproxy.log`或系统日志。

### 6.ACL访问控制列表

ACL（访问控制列表）在HAProxy配置中是一种非常强大的功能，它允许你基于客户端请求的不同特征来做出决策，例如重定向请求、选择不同的后端服务器、拒绝请求等。ACL可以基于多种类型的匹配条件，包括请求的URL、头信息、源IP地址、请求方法等。

**基本概念**

- **ACL语句**：在`frontend`或`listen`部分定义，用于匹配特定的请求属性。
- **条件**：定义匹配请求所需满足的条件，如路径、头信息值等。
- **动作**：当请求满足ACL条件时执行的操作，如使用特定的`backend`、拒绝请求等。

**定义ACL**

在`frontend`或`listen`部分定义ACL。语法如下：

```haproxy
acl <acl_name> <criterion> <value> [<value>...]
```

- `<acl_name>`：ACL的名称，自定义，用于引用此ACL。
- `<criterion>`：匹配标准，如`path_beg`（路径开始）、`hdr(host)`（主机头）等。
- `<value>`：与标准匹配的值。

**根据路径转发请求**

假设你希望根据请求的URL路径将请求分发到不同的后端服务器。

```haproxy
frontend http_front
    bind *:80
    acl url_blog path_beg /blog
    acl url_store path_beg /store
    use_backend blog_servers if url_blog
    use_backend store_servers if url_store
    default_backend web_servers
```

这个例子中，如果路径以`/blog`开头，请求将被发送到`blog_servers`后端；如果路径以`/store`开头，则请求被发送到`store_servers`后端。

**基于源IP拒绝请求**

如果你想拒绝来自特定IP地址的请求：

```haproxy
frontend http_front
    bind *:80
    acl blocked_ip src 192.168.1.100
    http-request deny if blocked_ip
```

此配置会拒绝所有来自`192.168.1.100`的请求。

**根据HTTP头信息做出决策**

基于请求的头信息，例如`User-Agent`或`Referer`，来决定请求的路由。

```haproxy
frontend http_front
    bind *:80
    acl old_browser hdr(User-Agent) -i oldie
    use_backend old_site_backend if old_browser
```

此例中，如果请求的`User-Agent`头信息包含“oldie”，则请求会被路由到`old_site_backend`。

**注意事项**

- **性能考虑**：虽然ACL非常强大，但大量复杂的ACL匹配可能影响性能，特别是在高流量的情况下。合理设计ACL规则，避免不必要的复杂性。
- **测试**：更改或新增ACL规则后，务必进行充分测试，以确保规则按预期工作，不会意外地阻断合法请求或允许不应通过的请求。
- **日志记录**：在调试ACL规则时，合理配置日志记录对于识别问题非常有帮助。

ACL是HAProxy配置中非常灵活且强大的部分，通过精心设计，可以实现高度定制的请求路由策略，有效提升应用的安全性、可用性和性能。

### 7.健康检查

HAProxy的高级健康检查功能提供了更灵活和详细的控制，以适应不同应用场景的需求。这些高级功能允许基于多样化的条件进行健康检查，从而更准确地评估后端服务器的健康状态。以下是一些高级健康检查的配置示例和说明：

**响应状态码范围**

你可以配置HAProxy以接受一系列的HTTP状态码作为健康检查通过的条件。这在某些情况下非常有用，比如当后端服务返回多个不同的成功状态码时。

```haproxy
backend web_backend
    mode http
    option httpchk GET /health
    http-check expect rstatus ^2[0-9][0-9]$
    server web1 192.168.0.3:80 check
```

这里的`http-check expect rstatus ^2[0-9][0-9]$`表示任何200-299之间的状态码都被认为是健康的。

**检查响应内容**

HAProxy可以根据HTTP健康检查的响应体内容来判断服务是否健康。这对于确保后端服务不仅能够响应请求，而且返回预期内容非常有用。

```haproxy
backend api_backend
    mode http
    option httpchk GET /api/health
    http-check expect string healthy
    server api1 192.168.0.5:80 check
```

在这个配置中，只有当`/api/health`路径返回的响应体中包含`healthy`字符串时，`api1`服务器才被认为是健康的。

**自定义健康检查条件**

HAProxy还允许定义更复杂的健康检查条件，例如结合使用多个`http-check expect`条件。

```haproxy
backend complex_backend
    mode http
    option httpchk GET /complex/health
    http-check expect rstatus ^2[0-9][0-9]$
    http-check expect ! string error
    server complex1 192.168.0.6:80 check
```

这里，后端服务必须返回一个200-299之间的状态码，并且响应体中不包含`error`字符串，服务才被视为健康。

**使用Lua脚本进行健康检查**

对于需要高度自定义逻辑的健康检查，HAProxy支持使用Lua脚本。这使得你可以编写复杂的检查逻辑，例如基于响应内容的特定模式或动态变化的条件。

要使用Lua脚本进行健康检查，你需要在`global`部分引入Lua脚本文件，并在`backend`配置中引用具体的Lua函数作为健康检查逻辑。

```haproxy
global
    lua-load /etc/haproxy/healthcheck.lua

backend lua_backend
    mode http
    option httpchk GET /lua/health
    http-check expect lua.is_healthy
    server lua1 192.168.0.7:80 check
```

这个示例中，`healthcheck.lua`是一个Lua脚本文件，其中包含名为`is_healthy`的函数，HAProxy会调用这个函数来执行健康检查。

**注意事项**

- 使用高级健康检查特性时，确保检查逻辑与后端服务的实际健康状况和预期行为相匹配。
- 复杂的健康检查逻辑可能增加HAProxy的处理负担，特别是在高流量场景下。合理设计健康检查，避免不必要的性能开销。
- 在生产环境中部署新的健康检查配置之前，应该进行充分的测试，确保配置按预期工作，不会误判服务健康状态。

通过使用HAProxy的高级健康检查功能，你可以更精细地控制流量分发逻辑，确保用户请求仅被路由到真正健康的后端服务，从而提高应用的可用性和用户体验。

### 结语

这是一个HAProxy入门级的使用示例，涵盖了安装、基本配置和启动服务的步骤。随着你对HAProxy的熟悉，你可以探索更高级的功能和配置选项，如SSL/TLS终端、更复杂的负载均衡规则、持久化连接等，来满足特定的负载均衡需求。