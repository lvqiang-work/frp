# 重要声明，本开源基于https://github.com/fatedier/frp，在原本基础功能上添加认证相关功能
# 基于v0.44.0 进行修改，git url https://github.com/fatedier/frp/tree/v0.44.0

# [官方参考地址：https://gofrp.org/docs/](https://gofrp.org/docs/)
# [改造参考地址：https://gitee.com/baibaiclouds/frp](https://gitee.com/baibaiclouds/frp)

# build windows
```
cd frp根路径

go build -o ./release/frps_windows_amd64.exe ./cmd/frps

go build -o ./release/frpc_windows_amd64.exe ./cmd/frpc
```

# 扩展功能说明
# 1. 客户端token认证
**功能场景说明**
客户端预先配置一个token，在客户端启动连接到服务端时，服务端会验证客户端的token是否合法，如果不合法则无法连接成功，跟官方的token、OIDC认证一样，只不过这里是基于简单http认证，方便自家系统进行对接认证。

**客户端配置**
```
[common]
server_addr = gofrp.org
server_port = 50000
token = 123456789
user = test
client_token = 123456789
```

**服务端配置**
```
[common]
bind_port = 50000
vhost_http_port = 50003
subdomain_host = gofrp.org
token = 123456789
allow_ports=50006-60000
token_auth_url = http://127.0.0.1:8080/api/frpc/tokenAuth
port_check_url = http://127.0.0.1:8080/api/frpc/checkPort
domain_check_url = http://127.0.0.1:8080/api/frpc/checkDomain
```

**验证流程**
1. 客户端预先写入[client_token]值
2. 启动客户端，让客户端连接到服务端
3. 服务端收到客户端的连接，此时服务端会调用[token_auth_url]的url进行验证,是get请求，验证的完整url是http://127.0.0.1:8080/api/frpc/tokenAuth?clientToken=123456789
4. http://127.0.0.1:8080/api/frpc/tokenAuth?clientToken=123456789 的返回内容是字符串 yes 表示通过，非 yes 字符串都认证失败，认证失败客户端无法连接成功。

**认证方式优点**
token_auth_url 的url随意修改,方便接入自家系统.

# 2. 客户端远程端口映射验证
**功能场景说明**
不希望客户端随意指定远程端口，比如客户端配置了[remote_port = 50006]，frps服务器会开启50006端口监听。这个机制是比较危险的，如果有客户端恶意指定远程端口，frps根本无法控制，而且frps做不到指定客户端只能指定映射指定的远程端口。
比如：A客户端只能配置[remote_port = 50006]，配置[remote_port = 8888]无效。
这里frps就要就要验证A客户端有没有权限配置50006、8888等远程端口。

**客户端配置**
```
# 此名字很重要，后续会传到服务端，代表ssh代理映射了remote_port=50006的远程端口
[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 50006
```

**服务端配置**
```
[common]
bind_port = 50000
vhost_http_port = 50003
subdomain_host = gofrp.org
token = 123456789
allow_ports=50000-60000
token_auth_url = http://127.0.0.1:8080/api/frpc/tokenAuth
port_check_url = http://127.0.0.1:8080/api/frpc/checkPort
domain_check_url = http://127.0.0.1:8080/api/frpc/checkDomain
```

**再次改造**
未考虑到不同用户，可以使用相同的协议名称。所以在验证端口的时候将clientToken一并上传。

**验证流程**
1. 客户端预先写入[ssh]代理名称和[remote_port]远程端口信息
2. 启动客户端，让客户端连接到服务端
3. 服务端收到客户端的连接，此时服务端会调用[port_check_url]的url进行验证,是get请求，验证的完整url是http://127.0.0.1:8080/api/frpc/checkPort?proxyName=ssh&remotePort=50006
4. http://127.0.0.1:8080/api/frpc/checkPort?clientToken=123456789&proxyName=ssh&remotePort=50006 的返回内容是字符串 yes 表示通过，非 yes 字符串都认证失败，认证失败端口无法代理成功。

**认证方式优点**
1. port_check_url 的url随意修改,方便接入自家系统.
2. 客户端的代理名称可自行修改，比如会加一些认证信息，再配合调用http协议进行验证，很容易进行控制指定的客户端只能指定对应的远程端口。

# 3. 客户端远程域名映射验证
**功能场景说明**
这里frps就要就要验证A客户端有没有权限配置远程域名。

**客户端配置**
```
# 此名字很重要，后续会传到服务端
[web]
type = http
local_ip = 127.0.0.1
local_port = 80
subdomain = test.web
```

**服务端配置**
```
[common]
bind_port = 8100
token_auth_url = http://127.0.0.1:8080/api/frpc/tokenAuth

# 此url用来验证客户端[ssh]的代理信息是否合法
domain_check_url = http://127.0.0.1:8080/api/frpc/checkDomain
```

**新增改造**
未考虑到使用http和https的时候，自己想使用的域名或以前使用的域名可能被别人抢占。所以在验证端口的时候将clientToken一并上传。
subdomain在原版的frp中并不允许使用*或者.  由于subdomain在管理系统中是自动生成的，生成规则是user.客户端的代理名称,所以注释掉原代码

**验证流程**
1. 客户端预先写入[ssh]代理名称和[subdomain]远程端口信息
2. 启动客户端，让客户端连接到服务端
3. 服务端收到客户端的连接，此时服务端会调用[domain_check_url]的url进行验证,是get请求，验证的完整url是http://127.0.0.1:8080/api/frpc/checkDomain?proxyName=ssh&subdomain=test.web
4. http://127.0.0.1:8080/api/frpc/checkDomain?proxyName=ssh&subdomain=test.web 的返回内容是字符串 yes 表示通过，非 yes 字符串都认证失败，认证失败端口无法代理成功。

**认证方式优点**
1. domain_check_url 的url随意修改,方便接入自家系统.
2. 客户端的代理名称可自行修改，比如会加一些认证信息，再配合调用http协议进行验证，很容易进行控制指定的客户端只能指定对应的远程端口。

# frp的详细配置请参照官方 README_zh.md
