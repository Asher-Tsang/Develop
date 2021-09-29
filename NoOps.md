# NoOps 常用命令指南

## proxy

* 创建proxy
  * `-user`指定 user-id（目前客服还未提供并行账号，故而所有 user-id 均指定为`1`放在管理员下）。同时还支持指定`-email`或`-phone`
  * `-internal` 指定集群登录节点服务 URL 地址，约定格式为: *`http://{node_name}.{cluster_code}:{port}`*
  * `-prefix-name` 指定生成的 public 地址前缀，约定格式为: `<app_name>-<cluster_code>-<account_name>-`, 然后系统最终生成 public 地址格式为 *`http://{prefix-name}-{random_string}.proxy.paratera.com`*

```shell
$ noops proxy create -user 1 -internal http://ln71.BSCC-T:10057 -prefix-name ms-bscc-t-sc80789-
USER   NAME     PUBLIC                                                               INTERNAL
1      zkzxgn   http://ms-bscc-t-sc80789-zkzxgn.proxy.paratera.com   http://ln71.bscc-t:10057
```

* 软删除 proxy

```shell
$ noops proxy delete -user 1 -name zkzxgn
Deleted: users/1/proxies/zkzxgn
```

## ssh

* 生成 sshproxy 账号信息

```shell
# 使用系统随机生成的密码为并行账号`zengzhuo@paratera.com`生成`sshproxy`账号信息
$ noops ssh create -email zengzhuo@paratera.com

# 使用已存在公钥为并行账号`zengzhuo@paratera.com`生成`sshproxy`账号信息
$ noops ssh create -email zengzhuo@paratera.com -pubkey @/tmp/id_rsa.pub
```

## acct

* 复制账号

```shell
$ noops acct cp -email zengzhuo@paratera.com -cluster BSCC-A -name sc50933
USER                                               CLUSTER   NAME      LOGIN_NAME
SELF-WPnpLlObgfvFo-VqtQ0qg3rAUx_cTGmyi-mdKjUpUws   BSCC-A    sc50933   sc50933@BSCC-A
```

> 缺省系统随机选择一个源并行账号下的集群账号复制到指定并行账号下，可以通过指定 `-src-email`, `-src-phone`, 或者 `-src-user` 来显示指定源并行账号。

## HTTP/TCP 反向代理

关于反向代理及Envoy

## Tips

* 正向代理

*正向代理（forward proxy）：是一个位于客户端和目标服务器之间的服务器(代理服务器)，为了从目标服务器取得内容，客户端向代理服务器发送一个请求并指定目标，然后代理服务器向目标服务器转交请求并将获得的内容返回给客户端。*

* 反向代理

*反向代理（reverse proxy）：是指以代理服务器来接受internet上的连接请求，然后将请求转发给内部网络上的服务器，并将从服务器上得到的结果返回给internet上请求连接的客户端，此时代理服务器对外就表现为一个反向代理服务器。*

## Design

* noops启动noopsd服务，该服务与envoy通信，动态更新envoy配置
* user向envoy对外暴露的端口，envoy根据配置将请求转发给server

## Improved development guide

* docker启动方式为-p映射端口时的使用方式

## Support proxy public customization

* internal 上游服务URL
* public 下游请求URL
* suffix (.proxy.paratera.com) 可在配置文件中自定义

### e.g

```shell
$ noops proxy create -user 1024 -internal http://example.com 
USER        NAME        PUBLIC                              INTERNAL            LABELS
1024        d1yxj2      http://d1yxj2.proxy.paratera.com    http://example.com  
```

* 未指定name时，name为随机生成固定长度字符串
* public: *<http://{random_string}.proxy.paratera.com>*

---

```shell
$ noops proxy create -user 1024 -internal http://example.com -prefix-name ms-bscc-a-sc50482
USER        NAME                           PUBLIC                                                 INTERNAL            LABELS
1024        ms-bscc-a-sc50482-q325sj       http://ms-bscc-a-sc50482-q325sj.proxy.paratera.com     http://example.com  
```

* 指定prefix-name时，public url为prefix-name+“-”+随机固定长度字符串+suffix拼接而成
* public: *<http://ms-{random_string}.proxy.paratera.com>*

---

```shell
$ noops proxy create -user 1024 -internal http://example.com -name proxy123
USER        NAME        PUBLIC                              INTERNAL            LABELS
1024        proxy123    http://proxy123.proxy.paratera.com  http://example.com  
```

* 指定name时，返回的public url为name和suffix拼接
* 同时指定name和prefix-name时，name覆盖prefix-name
* public: *<http://proxyid.proxy.paratera.com>*

---

```shell
$ noops proxy create -user 1024 -internal http://example.com -public http://returntest.com
USER        NAME        PUBLIC                  INTERNAL            LABELS
1024        8hshwe      http://returntest.com   http://example.com  
```

* 指定public时，返回的public url为public值
* public会覆盖prefix-name和name
* public: *<http://returntest.com>*

## Automatically delete data

* 最大保留日期以配置文件方式支持自定义，单位为天
* 每日凌晨3点执行goroutine,扫描表中update_time与当前日期相差大于最大保留日期且is_deleted值为1的记录
* 删除记录

## Improve labels

当前 labels 存储实现使用一个独立字段 labels 存储格式为: `labels:k1=v1,k2=kv2`。这种实现造成多个 label 组合查询不易于实现，另外没有数据库的约束容易造成存储时格式混乱。

故而考虑采用一个独立的表 `t_label` 存放所有 labels, 包含以下字段：

1. id: bigint 主键
2. name: varchar 索引，存放业务的 name， 比如 `users/1024/proxies/my-proxy` （考虑多个label）
3. label_key: varchar 索引 （key,value 为mysql保留字）
4. label_value: varchar 索引

此 `t_label` 使用 `name` 和其他业务表关联关系，由于 `name` 格式没有强约束，此 `t_label` 也还可以用于其他业务表，比如 `users/1024/sshSecrets/my-secret`

## Support proxy update

* 支持修改internal
* 支持修改public
* 支持修改labels

## Support TCP proxy

* 使用场景
  * 每个用户使用独立IP,PORT
  * 复用HTTP CRUD接口

* 创建

```shell
$ noops proxy create -internal tcp://172.18.12.171:9090 -public tcp://172.18.12.171:8081 -user 1024
USER   NAME     PUBLIC                     INTERNAL
1024   4mkjhn   tcp://172.18.12.171:8081   tcp://172.18.12.171:9090
```

* 删除

```shell
$ noops proxy delete -user 1024 -name 4mkjhn
Deleted: users/1024/proxies/4mkjhn
```

### Questions

* 启动Envoy的宿主机的监听端口不能和Envoy重复
* 目前测试internal和public只能是IP地址

## Deploy address desigin

* Create

  ```shell
  $ noops addr create -user 1024 -host ln124.BSCC-A3
  USER   HOST            NAME     INTERFACE   IP
  1024   ln124.BSCC-A3   5tzgl7

  ```

* List

  ```shell
  $ noops addr list
  USER   HOST            NAME          INTERFACE   IP
  1024   ln124.BSCC-A3   5tzgl7        
  1024   ln131.BSCC-A    addr-lkjfs3   lo1         172.18.12.182
  1024   ln132.BSCC-M    addr-sdf2sk   lo2         192.168.12.14
  ```

* Update

  ```shell
  $ noops addr update -user 1024 -host ln124.BSCC-A3 -name cr6khh -interface lo4 -ip 172.168.8.1/32
  USER   HOST            NAME     INTERFACE   IP
  1024   ln124.BSCC-A3   cr6khh   lo4         172.168.8.1
  ```

* List

  ```shell
  $ noops addr list
  USER   HOST            NAME          INTERFACE   IP
  1024   ln124.BSCC-A3   5tzgl7        lo4         172.168.8.1
  1024   ln131.BSCC-A    addr-lkjfs3   lo1         172.18.12.182
  1024   ln132.BSCC-M    addr-sdf2sk   lo2         192.168.12.14
  ```

* Delete
  
  ```shell
  $ noops addr delete -user 1024 -host ln124.BSCC-A3 -name 5tzgl7 
  Deleted: users/1024/hosts/ln124.BSCC-A3/addresses/5tzgl7
  ```

* List

  ```shell
  $ noops addr list -show-deleted
  USER   HOST            NAME     INTERFACE   IP
  1024   ln124.BSCC-A3   5tzgl7   lo4         172.168.8.1
  ```

* Undelete
  
  ```shell
  $ noops addr undelete -user 1024 -host ln124.BSCC-A3 -name 5tzgl7
  USER   HOST            NAME     INTERFACE   IP
  1024   ln124.BSCC-A3   5tzgl7   lo4         172.168.8.1
  ```

## Proxy HTTP API Design

* Create
  
  ```http
  POST: "/api/v1beta1/users/1024/regions/zw/proxies?"
  body: "proxy"
  ```

* List
  
  ```http
  GET: "/api/v1beta1/users/-/regions/-/proxies?"
  ```

* Delete
  
  ```http
  DELETE: "/api/v1beta1/users/1024/regions/zw/proxies/sinh2i"
  ```

* Undelete

  ```http
  POST: "/api/v1beta1/users/1024/regions/zw/proxies/sinh2i:undelete"
  body: "proxy"
  ```

## NoOps user update 更改方案

### 方案一

使用`-email` 和`-cemail`区别

> `-email` 指 `选择email`
>
> `-cemail` 指 `change email`

```shell
$noops user update -h
USAGE
  noops [flags] user update [flags] 

FLAGS
  -display-name ...    Change user display name
  -c-email ...         Change user email
  -c-phone ...         Change user phone
  -user ...            Select user id
  -email ...           Select user email
  -phone ...           Select user phone
  -version false       Show this program version
```

### 方案二

```shell
$noops user update -h
USAGE
  noops [flags] user update [flags] 

FLAGS
  -user ...          Select user id
  -email ...         Select user email
  -phone ...         Select user phone
  -version false     Show this program version
  
 $noops user update -email zengzhuo@paratera.com
 Select one of follow number to change:
 1.) display-name ...  Change user display name
 2.) email ...         Change user email
 3.) phone ...         Change user phone
 
```

### 方案三

```shell
$noops user update -h
USAGE
  noops [flags] user update <flags> [email:]<email_value> [phone:]<phone_value> [display-name:]<display-name_value>

FLAGS
  -user ...          Select user id
  -email ...         Select user email
  -phone ...         Select user phone
  -version false     Show this program version
```

## NoOps 新增导出超算账号绑定关系

  ```shell
  $noops acct desc -h
  USAGE
    noops acct desc [flags]

  FLAGS
    -cluster ...      Filter by cluster code
    -name ...         Filter by SSH account name
    -out ...          File path output
    -internal false   Show internal user
    -version false    Show this program version
    
  $noops acct desc -name deploy -cluster BSCC-A [-internal [true]]
  NAME: deploy
  CLUSTER: BSCC-A
  BIND_RELATIONSHIP:
  ID     EMAIL                          PHONE         STATE           DISPLAY_NAME
  1      admin@paratera.com                           Active          系统管理员
  10     jt.meng@siat.ac.cn                           Active          孟金涛
  1024   wutz@paratera.com              13391578256   Active          吴泰增
  ```

### Option One

  新增接口

  ```go
  type AccountService interface {
      // DescAccount list all aboout an account
      DescAccount(ctx context.Context, clusterCode string, accountID string, opts DescAccountsOptions) (*ListUsersPage, error)
  }
  ```

  > 优点: 无需发送多次请求
  >
  > 缺点: 需要新增接口,改动较大

### Option Two

  在noops已有命令行基础上实现(noops user list, noops acct list)

  >优点: 实现简单
  >
  >缺点: 产生多次网络I/O
