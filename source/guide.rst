
.. _guide:


用户指南 (User Guide)
^^^^^^^^^^^^^^^^^^^^^^

.. _authentication:


MQTT 认证/访问控制
------------------

**EMQ X** 消息服务器 *连接认证* 和 *访问控制* 由一系列的认证插件(Plugins)提供，他们的命名都符合 ``emqx_auth_<name>`` 的规则。

在 EMQ X 中，这俩个功能分别是指：

1. **连接认证**: *EMQ X* 会校验每个连接上的客户端是否具有接入系统的权限，若没有则会断开该连接
2. **访问控制**: *EMQ X* 会校验客户端每个 *发布/订阅(PUBLISH/SUBSCRIBE)* 的权限，以 *拒绝/允许* 此处操作


认证(Authentication)
>>>>>>>>>>>>>>>>>>>>>

*EMQ X* 消息服务器认证由一系列认证插件(Plugin)提供，系统支持按用户名密码、ClientID 或匿名认证。


系统默认开启匿名认证(anonymous)，通过加载认证插件可开启的多个认证模块组成认证链::

               ----------------           ----------------           ------------
    Client --> | Username认证 | -ignore-> | ClientID认证 | -ignore-> | 匿名认证 |
               ----------------           ----------------           ------------
                      |                         |                         |
                     \|/                       \|/                       \|/
                allow | deny              allow | deny              allow | deny


**开启匿名认证**

etc/emqx.conf 配置启用匿名认证:

.. code:: properties

    ## Allow anonymous authentication by default if no auth plugins loaded.
    ## Notice: Disable the option in production deployment!
    ##
    ## Value: true | false
    allow_anonymous = true


.. _acl:

访问控制(ACL)
>>>>>>>>>>>>>

*EMQ X* 消息服务器通过 ACL(Access Control List) 实现 MQTT 客户端访问控制。

ACL 访问控制规则定义::

    允许(Allow)|拒绝(Deny) 谁(Who) 订阅(Subscribe)|发布(Publish) 主题列表(Topics)

MQTT 客户端发起订阅/发布请求时，EMQ X 消息服务器的访问控制模块，会逐条匹配 ACL 规则，直到匹配成功为止::

              ---------              ---------              ---------
    Client -> | Rule1 | --nomatch--> | Rule2 | --nomatch--> | Rule3 | --> Default
              ---------              ---------              ---------
                  |                      |                      |
                match                  match                  match
                 \|/                    \|/                    \|/
            allow | deny           allow | deny           allow | deny


**默认访问控制设置**


*EMQ X* 消息服务器默认访问控制，在 etc/emqx.conf 中设置:

.. code:: properties

    ## Allow or deny if no ACL rules matched.
    ##
    ## Value: allow | deny
    acl_nomatch = allow

    ## Default ACL File.
    ##
    ## Value: File Name
    acl_file = etc/acl.conf

ACL 规则定义在 etc/acl.conf，EMQ X 启动时加载到内存:

.. code:: erlang

    %% Allow 'dashboard' to subscribe '$SYS/#'
    {allow, {user, "dashboard"}, subscribe, ["$SYS/#"]}.

    %% Allow clients from localhost to subscribe any topics
    {allow, {ipaddr, "127.0.0.1"}, pubsub, ["$SYS/#", "#"]}.

    %% Deny clients to subscribe '$SYS#' and '#'
    {deny, all, subscribe, ["$SYS/#", {eq, "#"}]}.

    %% Allow all by default
    {allow, all}.


EMQ X 3.1 版本提供的认证插件包括:

+----------------------------+---------------------------+
| 插件                       | 说明                      |
+============================+===========================+
| `emqx_auth_clientid`_      | ClientId 认证/鉴权插件    |
+----------------------------+---------------------------+
| `emqx_auth_username`_      | 用户名密码认证/鉴权插件   |
+----------------------------+---------------------------+
| `emqx_auth_jwt`_           | JWT 认证/鉴权插件         |
+----------------------------+---------------------------+
| `emqx_auth_ldap`_          | LDAP 认证/鉴权插件        |
+----------------------------+---------------------------+
| `emqx_auth_http`_          | HTTP 认证/鉴权插件        |
+----------------------------+---------------------------+
| `emqx_auth_mysql`_         | MySQ L认证/鉴权插件       |
+----------------------------+---------------------------+
| `emqx_auth_pgsql`_         | Postgre 认证/鉴权插件     |
+----------------------------+---------------------------+
| `emqx_auth_redis`_         | Redis 认证/鉴权插件       |
+----------------------------+---------------------------+
| `emqx_auth_mongo`_         | MongoDB 认证/鉴权插件     |
+----------------------------+---------------------------+

其中，关于每个认证插件的配置及用法，可参考 `扩展插件 (Plugins) <https://developer.emqx.io/docs/emq/v3/cn/plugins.html>`_ 关于认证部分。


.. note:: auth 插件可以同时启动多个。每次检查的时候，按照优先级从高到低依次检查，同一优先级的，先启动的插件先检查。(内置默认的 acl.conf 优先级为-1，各个插件默认为0)

此外 *EMQ X* 还支持使用 **PSK (Pre-shared Key)** 的方式来控制客户端的接入，但它并不是使用的上述的 *连接认证* 链的方式，而是在 SSL 握手期间进行验证。详情参考 `Pre-shared Key <https://en.wikipedia.org/wiki/Pre-shared_key>`_ 和 `emqx_psk_file`_


MQTT 发布订阅
-------------

MQTT 是为移动互联网、物联网设计的轻量发布订阅模式的消息服务器，目前支持 MQTT `v3.1.1 <http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/mqtt-v3.1.1.html>`_ 和 `v5.0 <http://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html>`_:

.. image:: ./_static/images/pubsub_concept.png

*EMQ X* 启动成功后，任何设备或终端的 MQTT 客户端，可通过 MQTT 协议连接到服务器，通过 PUBLISH/SUBSCRIBE 进行交换消息。

MQTT 协议客户端库: https://github.com/mqtt/mqtt.github.io/wiki/libraries

例如，mosquitto_sub/pub 命令行发布订阅消息::

    mosquitto_sub -t topic -q 2
    mosquitto_pub -t topic -q 1 -m "Hello, MQTT!"

*EMQ X* 对于 MQTT 协议服务所监听的端口等配置，都可在 etc/emqx.conf 文件中设置:

.. code:: properties

    ## TCP Listener: 1883, 127.0.0.1:1883, ::1:1883
    listener.tcp.external = 0.0.0.0:1883

    ## Size of acceptor pool
    listener.tcp.external.acceptors = 8

    ## Maximum number of concurrent clients
    listener.tcp.external.max_connections = 1024000

    ## Maximum external connections per second.
    ##
    ## Value: Number
    listener.tcp.external.max_conn_rate = 1000

MQTT/SSL 监听器，缺省端口8883:

.. code:: properties

    ## SSL Listener: 8883, 127.0.0.1:8883, ::1:8883
    listener.ssl.external = 8883

    ## Size of acceptor pool
    listener.ssl.external.acceptors = 16

    ## Maximum number of concurrent clients
    listener.ssl.external.max_connections = 102400

    ## Maximum MQTT/SSL connections per second.
    ##
    ## Value: Number
    listener.ssl.external.max_conn_rate = 500

.. _http_publish:


HTTP 发布接口
-------------

*EMQ X* 消息服务器提供了一个 HTTP 发布接口，应用服务器或 Web 服务器可通过该接口发布 MQTT 消息::

    HTTP POST http://host:8080/api/v3/mqtt/publish

Web 服务器例如 PHP/Java/Python/NodeJS 或 Ruby on Rails，可通过 HTTP POST 请求发布 MQTT 消息:

.. code:: bash

    curl -v --basic -u user:passwd -H "Content-Type: application/json" -d '{"qos":1, "retain": false, "topic":"world", "payload":"test" , "client_id": "C_1492145414740"}'  -k http://localhost:8080/api/v3/mqtt/publish

HTTP 接口参数:

+----------+----------------------+
| 参数     | 说明                 |
+==========+======================+
| client_id| MQTT 客户端 ID       |
+----------+----------------------+
| qos      | QoS: 0 | 1 | 2       |
+----------+----------------------+
| retain   | Retain: true | false |
+----------+----------------------+
| topic    | 主题(Topic)          |
+----------+----------------------+
| payload  | 消息载荷             |
+----------+----------------------+

.. NOTE::

    HTTP 发布接口采用 `Basic <https://en.wikipedia.org/wiki/Basic_access_authentication>`_ 认证。上例中的 ``user`` 和 ``passwd`` 是来自于 Dashboard 下的 Applications 内的 AppId 和 其密码


MQTT WebSocket 连接
-------------------

*EMQ X* 还支持 WebSocket 连接，Web 浏览器可直接通过 WebSocket 连接至服务器:

+-------------------------+----------------------------+
| WebSocket URI:          | ws(s)://host:8083/mqtt     |
+-------------------------+----------------------------+
| Sec-WebSocket-Protocol: | 'mqttv3.1' or 'mqttv3.1.1' |
+-------------------------+----------------------------+

Dashboard 插件提供了一个 MQTT WebSocket 连接的测试页面::

    http://127.0.0.1:18083/#/websocket

*EMQ X* 通过内嵌的 HTTP 服务器，实现 MQTT/WebSocket，etc/emqx.conf 设置:

.. code:: properties

    ## MQTT/WebSocket Listener
    listener.ws.external = 8083
    listener.ws.external.acceptors = 4
    ## Maximum number of concurrent MQTT/WebSocket connections.
    ##
    ## Value: Number
    listener.ws.external.max_connections = 102400

    ## Maximum MQTT/WebSocket connections per second.
    ##
    ## Value: Number
    listener.ws.external.max_conn_rate = 1000



.. _shared_sub:

共享订阅 (Shared Subscription)
-------------------------------

*EMQ X* R3.1 版本支持集群级别的共享订阅功能。 共享订阅(Shared Subscription)支持在多订阅者间采用多种常用的负载策略派发消息::

                                ---------
                                |       | --Msg1--> Subscriber1
    Publisher--Msg1,Msg2,Msg3-->| EMQ X | --Msg2--> Subscriber2
                                |       | --Msg3--> Subscriber3
                                ---------

共享订阅支持两种使用方式:

+-----------------+-------------------------------------------+
|  订阅前缀       | 使用示例                                  |
+-----------------+-------------------------------------------+
| $queue/         | mosquitto_sub -t '$queue/topic'           |
+-----------------+-------------------------------------------+
| $share/<group>/ | mosquitto_sub -t '$share/group/topic'     |
+-----------------+-------------------------------------------+

示例::

    mosquitto_sub -t '$share/group/topic'

    mosquitto_pub -t 'topic' -m msg -q 2


目前在 *EMQ X* R3.1 的版本中支持按以下几种策略进行负载共享的消息：

+---------------------------+-------------------------+
| 策略                      | 说明                    |
+===========================+=========================+
| random                    | 在所有共享订阅者中随机  |
+---------------------------+-------------------------+
| round_robin               | 按订阅顺序              |
+---------------------------+-------------------------+
| sticky                    | 使用上次派发的订阅者    |
+---------------------------+-------------------------+
| hash                      | 根据发送者的 ClientId   |
+---------------------------+-------------------------+

.. note:: 当所有的订阅者都不在线时，仍会挑选一个订阅者，并存至其 Session 的消息队列中


.. _sys_topic:

$SYS-系统主题
-------------

*EMQ X* 消息服务器周期性发布自身运行状态、消息统计、客户端上下线事件到 以 ``$SYS/`` 开头系统主题。

$SYS 主题路径以 ``$SYS/brokers/{node}/`` 开头。 ``{node}`` 是指产生该 事件/消息 所在的节点名称，例如::

    $SYS/brokers/emqx@127.0.0.1/version

    $SYS/brokers/emqx@127.0.0.1/uptime

.. NOTE:: 默认只允许 localhost 的 MQTT 客户端订阅 $SYS 主题，可通过 etc/acl.config 修改访问控制规则。

$SYS 系统消息发布周期，通过 etc/emqx.conf 配置:

.. code:: properties

    ## System interval of publishing $SYS messages.
    ##
    ## Value: Duration
    ## Default: 1m, 1 minute
    broker.sys_interval = 1m

.. _sys_brokers:

集群状态信息
>>>>>>>>>>>>

+--------------------------------+-----------------------+
| 主题                           | 说明                  |
+================================+=======================+
| $SYS/brokers                   | 集群节点列表          |
+--------------------------------+-----------------------+
| $SYS/brokers/${node}/version   | EMQ 服务器版本        |
+--------------------------------+-----------------------+
| $SYS/brokers/${node}/uptime    | EMQ 服务器启动时间    |
+--------------------------------+-----------------------+
| $SYS/brokers/${node}/datetime  | EMQ 服务器时间        |
+--------------------------------+-----------------------+
| $SYS/brokers/${node}/sysdescr  | EMQ 服务器描述        |
+--------------------------------+-----------------------+

.. _sys_clients:

客户端上下线事件
>>>>>>>>>>>>>>>>

$SYS 主题前缀: $SYS/brokers/${node}/clients/

+--------------------------+------------------------------------+
| 主题(Topic)              | 说明                               |
+==========================+====================================+
| ${clientid}/connected    | 上线事件。当某客户端上线时，会发布 |
|                          | 该消息                             |
+--------------------------+------------------------------------+
| ${clientid}/disconnected | 下线事件。当某客户端离线时，会发布 |
|                          | 该消息                             |
+--------------------------+------------------------------------+

'connected' 事件消息的 Payload 可解析成 JSON 格式:

.. code:: json

    {
        "clientid":"id1",
        "username":"u",
        "ipaddress":"127.0.0.1",
        "connack":0,
        "ts":1554047291,
        "proto_ver":3,
        "proto_name":"MQIsdp",
        "clean_start":true,
        "keepalive":60
    }


'disconnected' 事件消息的 Payload 可解析成 JSON 格式:

.. code:: json
    
    {
        "clientid":"id1",
        "username":"u",
        "reason":"normal",
        "ts":1554047291
    }


.. _sys_stats:

系统统计(Statistics)
>>>>>>>>>>>>>>>>>>>>

系统主题前缀: $SYS/brokers/${node}/stats/


客户端统计
::::::::::

+---------------------+---------------------------------------------+
| 主题(Topic)         | 说明                                        |
+---------------------+---------------------------------------------+
| connections/count   | 当前客户端总数                              |
+---------------------+---------------------------------------------+
| connections/max     | 最大客户端数量                              |
+---------------------+---------------------------------------------+


会话统计
::::::::

+-----------------------------+---------------------------------------------+
| 主题(Topic)                 | 说明                                        |
+-----------------------------+---------------------------------------------+
| sessions/count              | 当前会话总数                                |
+-----------------------------+---------------------------------------------+
| sessions/max                | 最大会话数量                                |
+-----------------------------+---------------------------------------------+
| sessions/persistent/count   | 当前持久会话总数                            |
+-----------------------------+---------------------------------------------+
| sessions/persistent/max     | 最大持久会话数量                            |
+-----------------------------+---------------------------------------------+


订阅统计
::::::::

+---------------------------------+---------------------------------------------+
| 主题(Topic)                     | 说明                                        |
+---------------------------------+---------------------------------------------+
| suboptions/count                | 当前订阅选项个数                            |
+---------------------------------+---------------------------------------------+
| suboptions/max                  | 最大订阅选项总数                            |
+---------------------------------+---------------------------------------------+
| subscribers/max                 | 最大订阅者总数                              |
+---------------------------------+---------------------------------------------+
| subscribers/count               | 当前订阅者数量                              |
+---------------------------------+---------------------------------------------+
| subscriptions/max               | 最大订阅数量                                |
+---------------------------------+---------------------------------------------+
| subscriptions/count             | 当前订阅总数                                |
+---------------------------------+---------------------------------------------+
| subscriptions/shared/count      | 当前共享订阅个数                            |
+---------------------------------+---------------------------------------------+
| subscriptions/shared/max        | 当前共享订阅总数                            |
+---------------------------------+---------------------------------------------+


主题统计
::::::::

+---------------------+---------------------------------------------+
| 主题(Topic)         | 说明                                        |
+---------------------+---------------------------------------------+
| topics/count        | 当前 Topic 总数                             |
+---------------------+---------------------------------------------+
| topics/max          | 最大 Topic 数量                             |
+---------------------+---------------------------------------------+


路由统计
::::::::

+---------------------+---------------------------------------------+
| 主题(Topic)         | 说明                                        |
+---------------------+---------------------------------------------+
| routes/count        | 当前 Routes 总数                            |
+---------------------+---------------------------------------------+
| routes/max          | 最大 Routes 数量                            |
+---------------------+---------------------------------------------+

.. note:: ``topics/count`` 和 ``topics/max`` 与 ``routes/count`` 和 ``routes/max`` 数值上是想等的


收发流量/报文/消息统计
>>>>>>>>>>>>>>>>>>>>>>

系统主题(Topic)前缀: $SYS/brokers/${node}/metrics/

收发流量统计
::::::::::::

+---------------------+---------------------------------------------+
| 主题(Topic)         | 说明                                        |
+---------------------+---------------------------------------------+
| bytes/received      | 累计接收流量                                |
+---------------------+---------------------------------------------+
| bytes/sent          | 累计发送流量                                |
+---------------------+---------------------------------------------+

MQTT报文收发统计
::::::::::::::::

+-----------------------------+---------------------------------------------+
| 主题(Topic)                 | 说明                                        |
+-----------------------------+---------------------------------------------+
| packets/received            | 累计接收 MQTT 报文                          |
+-----------------------------+---------------------------------------------+
| packets/sent                | 累计发送 MQTT 报文                          |
+-----------------------------+---------------------------------------------+
| packets/connect             | 累计接收 MQTT CONNECT 报文                  |
+-----------------------------+---------------------------------------------+
| packets/connack             | 累计发送 MQTT CONNACK 报文                  |
+-----------------------------+---------------------------------------------+
| packets/publish/received    | 累计接收 MQTT PUBLISH 报文                  |
+-----------------------------+---------------------------------------------+
| packets/publish/sent        | 累计发送 MQTT PUBLISH 报文                  |
+-----------------------------+---------------------------------------------+
| packets/puback/received     | 累计接收 MQTT PUBACK 报文                   |
+-----------------------------+---------------------------------------------+
| packets/puback/sent         | 累计发送 MQTT PUBACK 报文                   |
+-----------------------------+---------------------------------------------+
| packets/puback/missed       | 累计丢失 MQTT PUBACK 报文                   |
+-----------------------------+---------------------------------------------+
| packets/pubrec/received     | 累计接收 MQTT PUBREC 报文                   |
+-----------------------------+---------------------------------------------+
| packets/pubrec/sent         | 累计发送 MQTT PUBREC 报文                   |
+-----------------------------+---------------------------------------------+
| packets/pubrec/missed       | 累计丢失 MQTT PUBREC 报文                   |
+-----------------------------+---------------------------------------------+
| packets/pubrel/received     | 累计接收 MQTT PUBREL 报文                   |
+-----------------------------+---------------------------------------------+
| packets/pubrel/sent         | 累计发送 MQTT PUBREL 报文                   |
+-----------------------------+---------------------------------------------+
| packets/pubrel/missed       | 累计丢失 MQTT PUBREL 报文                   |
+-----------------------------+---------------------------------------------+
| packets/pubcomp/received    | 累计接收 MQTT PUBCOMP 报文                  |
+-----------------------------+---------------------------------------------+
| packets/pubcomp/sent        | 累计发送 MQTT PUBCOMP 报文                  |
+-----------------------------+---------------------------------------------+
| packets/pubcomp/missed      | 累计丢失 MQTT PUBCOMP 报文                  |
+-----------------------------+---------------------------------------------+
| packets/subscribe           | 累计接收 MQTT SUBSCRIBE 报文                |
+-----------------------------+---------------------------------------------+
| packets/suback              | 累计发送 MQTT SUBACK 报文                   |
+-----------------------------+---------------------------------------------+
| packets/unsubscribe         | 累计接收 MQTT UNSUBSCRIBE 报文              |
+-----------------------------+---------------------------------------------+
| packets/unsuback            | 累计发送 MQTT UNSUBACK 报文                 |
+-----------------------------+---------------------------------------------+
| packets/pingreq             | 累计接收 MQTT PINGREQ 报文                  |
+-----------------------------+---------------------------------------------+
| packets/pingresp            | 累计发送 MQTT PINGRESP 报文                 |
+-----------------------------+---------------------------------------------+
| packets/disconnect/received | 累计接收 MQTT DISCONNECT 报文               |
+-----------------------------+---------------------------------------------+
| packets/disconnect/sent     | 累计接收 MQTT DISCONNECT 报文               |
+-----------------------------+---------------------------------------------+
| packets/auth                | 累计接收Auth 报文                           |
+-----------------------------+---------------------------------------------+


MQTT 消息收发统计
:::::::::::::::::

+--------------------------+---------------------------------------------+
| 主题(Topic)              | 说明                                        |
+--------------------------+---------------------------------------------+
| messages/received        | 累计接收消息                                |
+--------------------------+---------------------------------------------+
| messages/sent            | 累计发送消息                                |
+--------------------------+---------------------------------------------+
| messages/expired         | 累计发送消息                                |
+--------------------------+---------------------------------------------+
| messages/retained        | Retained 消息总数                           |
+--------------------------+---------------------------------------------+
| messages/dropped         | 丢弃消息总数                                |
+--------------------------+---------------------------------------------+
| messages/forward         | 节点转发消息总数                            |
+--------------------------+---------------------------------------------+
| messages/qos0/received   | 累计接受QoS0消息                            |
+--------------------------+---------------------------------------------+
| messages/qos0/sent       | 累计发送QoS0消息                            |
+--------------------------+---------------------------------------------+
| messages/qos1/received   | 累计接受QoS1消息                            |
+--------------------------+---------------------------------------------+
| messages/qos1/sent       | 累计发送QoS1消息                            |
+--------------------------+---------------------------------------------+
| messages/qos2/received   | 累计接受QoS2消息                            |
+--------------------------+---------------------------------------------+
| messages/qos2/sent       | 累计发送QoS2消息                            |
+--------------------------+---------------------------------------------+
| messages/qos2/expired    | QoS2过期消息总数                            |
+--------------------------+---------------------------------------------+
| messages/qos2/dropped    | QoS2丢弃消息总数                            |
+--------------------------+---------------------------------------------+

.. _sys_alarms:

Alarms - 系统告警
>>>>>>>>>>>>>>>>>

系统主题(Topic)前缀: $SYS/brokers/${node}/alarms/

+------------------+------------------+
| 主题(Topic)      | 说明             |
+------------------+------------------+
| ${alarmId}/alert | 新产生告警       |
+------------------+------------------+
| ${alarmId}/clear | 清除告警         |
+------------------+------------------+

.. _sys_sysmon:

Sysmon - 系统监控
>>>>>>>>>>>>>>>>>

系统主题(Topic)前缀: $SYS/brokers/${node}/sysmon/

+------------------+--------------------+
| 主题(Topic)      | 说明               |
+------------------+--------------------+
| long_gc          | GC 时间过长警告    |
+------------------+--------------------+
| long_schedule    | 调度时间过长警告   |
+------------------+--------------------+
| large_heap       | Heap 内存占用警告  |
+------------------+--------------------+
| busy_port        | Port 忙警告        |
+------------------+--------------------+
| busy_dist_port   | Dist Port 忙警告   |
+------------------+--------------------+

.. _trace:


追踪
----

EMQ X 消息服务器支持追踪来自某个客户端(Client)，或者发布到某个主题(Topic)的全部消息。

追踪来自客户端(Client)的消息:

.. code:: bash

    $ ./bin/emqx_ctl log primary-level debug

    $ ./bin/emqx_ctl trace start client "clientid" "trace_clientid.log" debug

追踪发布到主题(Topic)的消息:

.. code:: bash

    $ ./bin/emqx_ctl log primary-level debug

    $ ./bin/emqx_ctl trace start topic 't/#' "trace_topic.log" debug

查询追踪:

.. code:: bash

    $ ./bin/emqx_ctl trace list

停止追踪:

.. code:: bash

    $ ./bin/emqx_ctl trace stop client "clientid"

    $ ./bin/emqx_ctl trace stop topic "topic"

.. _emqx_auth_clientid: https://github.com/emqx/emqx-auth-clientid
.. _emqx_auth_username: https://github.com/emqx/emqx-auth-username
.. _emqx_auth_ldap:     https://github.com/emqx/emqx-auth-ldap
.. _emqx_auth_http:     https://github.com/emqx/emqx-auth-http
.. _emqx_auth_mysql:    https://github.com/emqx/emqx-auth-mysql
.. _emqx_auth_pgsql:    https://github.com/emqx/emqx-auth-pgsql
.. _emqx_auth_redis:    https://github.com/emqx/emqx-auth-redis
.. _emqx_auth_mongo:    https://github.com/emqx/emqx-auth-mongo
.. _emqx_auth_jwt:      https://github.com/emqx/emqx-auth-jwt
.. _emqx_psk_file:      https://github.com/emqx/emqx-psk-file

.. _rule_engine:


规则引擎
---------

规则引擎用于配置事件的业务规则。使用规则引擎可以方便地实现诸如将消息筛选和转换成指定格式，并存入数据库表，或者发送到消息队列等功能。

规则引擎相关的概念包括: 规则(rule)、动作(action)、资源(resource) 和 资源类型(resource-type)。

- 规则 (Rule): 规则由 SQL 语句和动作列表组成。
  SQL 语句用于从 emqx 事件中筛选和转换原始数据。
  动作列表包含一个或多个动作及其参数。
  规则需要挂载到某个事件上，比如 “消息发布”，“连接完成”，“连接断开” 等。挂载事件由 SQL 语句的 “FROM” 子句完成。
- 动作 (Action): 动作定义了一个针对数据的操作。
  动作可以绑定资源，也可以不绑定。例如，“inspect” 动作不需要绑定资源，它只是简单打印数据内容和动作参数。而 “data_to_webserver” 动作需要绑定一个 web_hook 类型的资源，此资源中配置了 URL。
- 资源 (Resource): 资源是通过资源类型为模板实例化出来的对象，保存了与资源相关的配置(比如数据库连接地址和端口、用户名和密码等)。
- 资源类型 (Resource Type): 资源类型是资源的静态定义，描述了此类型资源需要的配置项。

.. important:: 动作和资源类型是由 emqx 或插件的代码提供的，不能通过 API 和 CLI 动态创建。

规则、动作、资源的关系::

    规则: {
        SQL 语句,
        动作列表: [
            {
                动作参数: {
                    参数1: 值,
                    参数2: 值,
                    ...
                },
                绑定资源: {
                    资源配置项: {
                        配置项1: 值,
                        配置项2: 值,
                        ...
                    }
                }
            }
        ]
    }

SQL 语句
>>>>>>>>>>>>>>

SQL 语句用于从原始数据中，根据条件筛选出字段，并进行预处理和转换，基本格式为::

    SELECT <字段名> FROM <触发事件> [WHERE <条件>]

SQL 语句示例:

- 从 topic 为 "t/a" 的消息中提取所有字段::

    SELECT * FROM "message.publish" WHERE topic = 't/a'

- 从 topic 能够匹配到 't/#' 的消息中提取所有字段。注意这里使用了 '=~' 操作符进行带通配符的 topic 匹配::

    SELECT * FROM "message.publish" WHERE topic =~ 't/#'

- 从 topic 能够匹配到 't/#' 的消息中提取 qos，username 和 client_id 字段::

    SELECT qos, username, client_id FROM "message.publish" WHERE topic =~ 't/#'

- 从任意 topic 的消息中提取 username 字段，并且筛选条件为 username = 'Steven'::

    SELECT username FROM "message.publish" WHERE username='Steven'

- 从任意 topic 的消息的消息体(payload) 中提取 x 字段，并创建别名 x 以便在 WHERE 子句中使用。WHERE 子句限定条件为 x = 1。注意 payload 必须为 JSON 格式。举例：此 SQL 语句可以匹配到消息体 {"x": 1}, 但不能匹配到消息体 {"x": 2}::

    SELECT payload.x as x FROM "message.publish" WHERE x=1

- 类似于上面的 SQL 语句，但嵌套地提取消息体中的数据，此 SQL 语句可以匹配到消息体 {"x": {"y": 1}}::

    SELECT payload.x.y as a FROM "message.publish" WHERE a=1

- 在 client_id = 'c1' 尝试连接时，提取其来源 IP 地址和端口号::

    SELECT peername as ip_port FROM "client.connected" WHERE client_id = 'c1'

- 筛选所有订阅 't/#' 主题且订阅级别为 QoS1 的 client_id。注意这里用的是严格相等操作符 '='，所以不会匹配主题为 't' 或 't/+/a' 的订阅请求::

    SELECT client_id FROM "client.subscribe" WHERE topic = 't/#' and qos = 1

- 事实上，上例中的 topic 和 qos 字段，是当订阅请求里只包含了一对 (Topic, QoS) 时，为使用方便而设置的别名。但如果订阅请求中 Topic Filters 包含了多个 (Topic, QoS) 组合对，那么必须显式使用 contains_topic() 或 contains_topic_match() 函数来检查 Topic Filters 是否包含指定的 (Topic, QoS)::

    SELECT client_id FROM "client.subscribe" WHERE contains_topic(topic_filters, 't/#')

    SELECT client_id FROM "client.subscribe" WHERE contains_topic(topic_filters, 't/#', 1)

SELECT 子句可用的字段:
>>>>>>>>>>>>>>>>>>>>>>

消息发布(message.publish):

+-----------+------------------------------------+
| client_id | Client ID                          |
+-----------+------------------------------------+
| username  | 用户名                             |
+-----------+------------------------------------+
| event     | 事件类型，固定为 "message.publish" |
+-----------+------------------------------------+
| flags     | MQTT 消息的 flags                  |
+-----------+------------------------------------+
| id        | MQTT 消息 ID                       |
+-----------+------------------------------------+
| topic     | MQTT 主题                          |
+-----------+------------------------------------+
| payload   | MQTT 消息体                        |
+-----------+------------------------------------+
| peername  | 客户端的 IPAddress 和 Port         |
+-----------+------------------------------------+
| qos       | MQTT 消息的 QoS                    |
+-----------+------------------------------------+
| timestamp | 时间戳                             |
+-----------+------------------------------------+

消息投递(message.deliver):

+-------------+------------------------------------+
| client_id   | Client ID                          |
+-------------+------------------------------------+
| username    | 用户名                             |
+-------------+------------------------------------+
| event       | 事件类型，固定为 "message.deliver" |
+-------------+------------------------------------+
| flags       | MQTT 消息的 flags                  |
+-------------+------------------------------------+
| id          | MQTT 消息 ID                       |
+-------------+------------------------------------+
| topic       | MQTT 主题                          |
+-------------+------------------------------------+
| payload     | MQTT 消息体                        |
+-------------+------------------------------------+
| peername    | 客户端的 IPAddress 和 Port         |
+-------------+------------------------------------+
| qos         | MQTT 消息的 QoS                    |
+-------------+------------------------------------+
| timestamp   | 时间戳                             |
+-------------+------------------------------------+
| auth_result | 认证结果                           |
+-------------+------------------------------------+
| mountpoint  | 消息主题挂载点                     |
+-------------+------------------------------------+

消息确认(message.acked):

+-----------+----------------------------------+
| client_id | Client ID                        |
+-----------+----------------------------------+
| username  | 用户名                           |
+-----------+----------------------------------+
| event     | 事件类型，固定为 "message.acked" |
+-----------+----------------------------------+
| flags     | MQTT 消息的 flags                |
+-----------+----------------------------------+
| id        | MQTT 消息 ID                     |
+-----------+----------------------------------+
| topic     | MQTT 主题                        |
+-----------+----------------------------------+
| payload   | MQTT 消息体                      |
+-----------+----------------------------------+
| peername  | 客户端的 IPAddress 和 Port       |
+-----------+----------------------------------+
| qos       | MQTT 消息的 QoS                  |
+-----------+----------------------------------+
| timestamp | 时间戳                           |
+-----------+----------------------------------+

消息丢弃(message.dropped):

+-----------+------------------------------------+
| client_id | Client ID                          |
+-----------+------------------------------------+
| username  | 用户名                             |
+-----------+------------------------------------+
| event     | 事件类型，固定为 "message.dropped" |
+-----------+------------------------------------+
| flags     | MQTT 消息的 flags                  |
+-----------+------------------------------------+
| id        | MQTT 消息 ID                       |
+-----------+------------------------------------+
| topic     | MQTT 主题                          |
+-----------+------------------------------------+
| payload   | MQTT 消息体                        |
+-----------+------------------------------------+
| peername  | 客户端的 IPAddress 和 Port         |
+-----------+------------------------------------+
| qos       | MQTT 消息的 QoS                    |
+-----------+------------------------------------+
| timestamp | 时间戳                             |
+-----------+------------------------------------+
| node      | 节点名                             |
+-----------+------------------------------------+

连接完成(client.connected):

+--------------+-------------------------------------+
| client_id    | Client ID                           |
+--------------+-------------------------------------+
| username     | 用户名                              |
+--------------+-------------------------------------+
| event        | 事件类型，固定为 "client.connected" |
+--------------+-------------------------------------+
| auth_result  | 认证结果                            |
+--------------+-------------------------------------+
| clean_start  | MQTT clean start 标志位             |
+--------------+-------------------------------------+
| connack      | MQTT CONNACK 结果                   |
+--------------+-------------------------------------+
| connected_at | 连接时间戳                          |
+--------------+-------------------------------------+
| is_bridge    | 是否是桥接                          |
+--------------+-------------------------------------+
| keepalive    | MQTT 保活间隔                       |
+--------------+-------------------------------------+
| mountpoint   | 消息主题挂载点                      |
+--------------+-------------------------------------+
| peername     | 客户端的 IPAddress 和 Port          |
+--------------+-------------------------------------+
| proto_ver    | MQTT 协议版本                       |
+--------------+-------------------------------------+


连接断开(client.disconnected):

+-------------+----------------------------------------+
| client_id   | Client ID                              |
+-------------+----------------------------------------+
| username    | 用户名                                 |
+-------------+----------------------------------------+
| event       | 事件类型，固定为 "client.disconnected" |
+-------------+----------------------------------------+
| auth_result | 认证结果                               |
+-------------+----------------------------------------+
| mountpoint  | 消息主题挂载点                         |
+-------------+----------------------------------------+
| peername    | 客户端的 IPAddress 和 Port             |
+-------------+----------------------------------------+
| reason_code | 断开原因码                             |
+-------------+----------------------------------------+

订阅(client.subscribe):

+---------------+-------------------------------------+
| client_id     | Client ID                           |
+---------------+-------------------------------------+
| username      | 用户名                              |
+---------------+-------------------------------------+
| event         | 事件类型，固定为 "client.subscribe" |
+---------------+-------------------------------------+
| auth_result   | 认证结果                            |
+---------------+-------------------------------------+
| mountpoint    | 消息主题挂载点                      |
+---------------+-------------------------------------+
| peername      | 客户端的 IPAddress 和 Port          |
+---------------+-------------------------------------+
| topic_filters | MQTT 订阅列表                       |
+---------------+-------------------------------------+
| topic         | MQTT 订阅列表中的第一个订阅的主题   |
+---------------+-------------------------------------+
| topic_filters | MQTT 订阅列表中的第一个订阅的 QoS   |
+---------------+-------------------------------------+

取消订阅(client.unsubscribe):

+---------------+---------------------------------------+
| client_id     | Client ID                             |
+---------------+---------------------------------------+
| username      | 用户名                                |
+---------------+---------------------------------------+
| event         | 事件类型，固定为 "client.unsubscribe" |
+---------------+---------------------------------------+
| auth_result   | 认证结果                              |
+---------------+---------------------------------------+
| mountpoint    | 消息主题挂载点                        |
+---------------+---------------------------------------+
| peername      | 客户端的 IPAddress 和 Port            |
+---------------+---------------------------------------+
| topic_filters | MQTT 订阅列表                         |
+---------------+---------------------------------------+
| topic         | MQTT 订阅列表中的第一个订阅的主题     |
+---------------+---------------------------------------+
| topic_filters | MQTT 订阅列表中的第一个订阅的 QoS     |
+---------------+---------------------------------------+

.. important::
    - FROM 子句后面的主题名需要用双引号("") 引起来。
    - WHERE 子句后面接筛选条件，如果使用到字符串需要用单引号 ('') 引起来。
    - SELECT 子句中，若使用 "." 符号对 payload 进行嵌套选择，必须保证 payload 为 JSON 格式。

创建规则举例
------------

例: 创建 Inspect 规则
>>>>>>>>>>>>>>>>>>>>>>>

创建一个测试规则，当有消息发送到 't/a' 主题时，打印消息内容以及动作参数细节。

- 规则的筛选 SQL 语句为: SELECT * FROM "message.publish" WHERE topic = 't/a';
- 动作是: "打印动作参数细节"，需要使用内置动作 'inspect'。

.. code-block:: shell

    $ ./bin/emqx_ctl rules create \
      "SELECT * FROM \"message.publish\" WHERE topic = 't/a'" \
      '[{"name":"inspect", "params": {"a": 1}}]' \
      -d 'Rule for debug'

    Rule rule:803de6db created

上面的 CLI 命令创建了一个 ID 为 'Rule rule:803de6db' 的规则。

参数中前两个为必参数:

- SQL 语句: SELECT * FROM "message.publish" WHERE topic = 't/a'
- 动作列表: [{"name":"inspect", "params": {"a": 1}}]。动作列表是用 JSON Array 格式表示的。name 字段是动作的名字，params 字段是动作的参数。注意 ``inspect`` 动作是不需要绑定资源的。

最后一个可选参数，是规则的描述: 'Rule for debug'。

接下来当发送 "hello" 消息到主题 't/a' 时，上面创建的 "Rule rule:803de6db" 规则匹配成功，然后 "inspect" 动作被触发，将消息和参数内容打印到 emqx 控制台::

    $ tail -f log/erlang.log.1

    (emqx@127.0.0.1)1> [inspect]
        Selected Data: #{client_id => <<"shawn">>,event => 'message.publish',
                         flags => #{dup => false,retain => false},
                         id => <<"5898704A55D6AF4430000083D0002">>,
                         payload => <<"hello">>,
                         peername => <<"127.0.0.1:61770">>,qos => 1,
                         timestamp => 1558587875090,topic => <<"t/a">>,
                         username => undefined}
        Envs: #{event => 'message.publish',
                flags => #{dup => false,retain => false},
                from => <<"shawn">>,
                headers =>
                    #{allow_publish => true,
                      peername => {{127,0,0,1},61770},
                      username => undefined},
                id => <<0,5,137,135,4,165,93,106,244,67,0,0,8,61,0,2>>,
                payload => <<"hello">>,qos => 1,
                timestamp => {1558,587875,89754},
                topic => <<"t/a">>}
        Action Init Params: #{<<"a">> => 1}

- ``Selected Data`` 列出的是消息经过 SQL 筛选、提取后的字段，由于我们用的是 ``select *``，所以这里会列出所有可用的字段。
- ``Envs`` 是动作内部可以使用的环境变量。
- ``Action Init Params`` 是初始化动作的时候，我们传递给动作的参数。


例: 创建 WebHook 规则
>>>>>>>>>>>>>>>>>>>>>>>

创建一个规则，将所有发送自 client_id='Steven' 的消息，转发到地址为 'http://127.0.0.1:9910' 的 Web 服务器:

- 规则的筛选条件为: SELECT username as u, payload FROM "message.publish" where u='Steven';
- 动作是: "转发到地址为 'http://127.0.0.1:9910' 的 Web 服务";
- 资源类型是: web_hook;
- 资源是: "到 url='http://127.0.0.1:9910' 的 WebHook 资源"。

0. 首先我们创建一个简易 Web 服务，这可以使用 ``nc`` 命令实现::

    $ echo -e "HTTP/1.1 200 OK\n\n $(date)" | nc -l localhost 9910

1. 使用 WebHook 类型创建一个资源，并配置资源参数 url:

   1). 列出当前所有可用的资源类型，确保 'web_hook' 资源类型已存在::

    $ ./bin/emqx_ctl resource-types list

    resource_type(name='web_hook', provider='emqx_web_hook', params=#{...}}, on_create={emqx_web_hook_actions,on_resource_create}, description='WebHook Resource')
    ...

   2). 使用类型 'web_hook' 创建一个新的资源，并配置 "url"="http://127.0.0.1:9910"::

    $ ./bin/emqx_ctl resources create \
      'web_hook' \
      -c '{"url": "http://127.0.0.1:9910", "headers": {"token":"axfw34y235wrq234t4ersgw4t"}, "method": "POST"}'

    Resource resource:691c29ba created

   上面的 CLI 命令创建了一个 ID 为 'resource:691c29ba' 的资源，第一个参数是必选参数 - 资源类型(web_hook)。参数表明此资源指向 URL = "http://127.0.0.1:9910" 的 Web 服务，方法为 POST，并且设置了一个 HTTP Header: "token"。

2. 然后创建规则，并选择规则的动作为 'data_to_webserver':

   1). 列出当前所有可用的动作，确保 'data_to_webserver' 动作已存在::

    $ ./bin/emqx_ctl rule-actions list

    action(name='data_to_webserver', app='emqx_web_hook', for='$any', types=[web_hook], params=#{'$resource' => ...}, title ='Data to Web Server', description='Forward Messages to Web Server')
    ...

   2). 创建规则，选择 data_to_webserver 动作，并通过 "$resource" 参数将 resource:691c29ba 资源绑定到动作上::

    $ ./bin/emqx_ctl rules create \
     "SELECT username as u, payload FROM \"message.publish\" where u='Steven'" \
     '[{"name":"data_to_webserver", "params": {"$resource":  "resource:691c29ba"}}]' \
     -d "Forward publish msgs from steven to webserver"

    rule:26d84768

   上面的 CLI 命令与第一个例子里创建 Inspect 规则时类似，区别在于这里需要把刚才创建的资源 'resource:691c29ba' 绑定到 'data_to_webserver' 动作上。这个绑定通过给动作设置一个特殊的参数 '$resource' 完成。'data_to_webserver' 动作的作用是将数据发送到指定的 Web 服务器。

3. 现在我们使用 username "Steven" 发送 "hello" 到任意主题，上面创建的规则就会被触发，Web Server 收到消息并回复 200 OK::

    $ echo -e "HTTP/1.1 200 OK\n\n $(date)" | nc -l localhost 9910

    POST / HTTP/1.1
    content-type: application/json
    content-length: 32
    te:
    host: 127.0.0.1:9910
    connection: keep-alive
    token: axfw34y235wrq234t4ersgw4t

    {"payload":"hello","u":"Steven"}

例: 创建 MySQL 规则
>>>>>>>>>>>>>>>>>>>>>>>

0. 搭建 MySQL 数据库，并设置用户名密码为 root/public，以 MacOS X 为例::

    $ brew install mysql

    $ brew services start mysql

    $ mysql -u root -h localhost -p

      ALTER USER 'root'@'localhost' IDENTIFIED BY 'public';

1. 初始化 MySQL 表:

  $ mysql -u root -h localhost -ppublic

  创建 “test” 数据库::

    CREATE DATABASE test;

  创建 “t_mqtt_msg” 表::

    USE test;

    CREATE TABLE `t_mqtt_msg` (
    `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
    `msgid` varchar(64) DEFAULT NULL,
    `topic` varchar(255) NOT NULL,
    `qos` tinyint(1) NOT NULL DEFAULT '0',
    `payload` blob,
    `arrived` datetime NOT NULL,
    PRIMARY KEY (`id`),
    INDEX topic_index(`id`, `topic`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8MB4;

  .. image:: ./_static/images/mysql_init_1@2x.png

2. 创建规则:

  打开 `emqx dashboard <http://127.0.0.1:18083/#/rules>`_，选择左侧的 “规则” 选项卡。

  选择触发事件 “消息发布”，然后填写规则 SQL::

    SELECT * FROM "message.pubish"

  .. image:: ./_static/images/rule_sql_1@2x.png

3. 关联动作:

  在 “响应动作” 界面选择 “添加”，然后在 “动作” 下拉框里选择 “发送数据到 MySQL”。

  .. image:: ./_static/images/rule_action_1@2x.png

4. 填写动作参数:

  “发送数据到 MySQL” 动作需要两个参数：

  1). SQL 模板。这个例子里我们向 MySQL 插入一条数据，SQL 模板为::

    insert into t_mqtt_msg(msgid, topic, qos, payload, arrived) values (${id}, ${topic}, ${qos}, ${payload}, FROM_UNIXTIME(${timestamp}/1000))

  .. image:: ./_static/images/rule_action_2@2x.png

  2). 关联资源的 ID。现在资源下拉框为空，可以点击右上角的 “新建资源” 来创建一个 MySQL 资源:

  .. image:: ./_static/images/rule_action_3@2x.png

  选择 “MySQL 资源”。

5. 填写资源配置:

  数据库名填写 “test”，用户名填写 “root”，密码填写 “publish”，备注为 “MySQL resource to 127.0.0.1:3306 db=test”

  .. image:: ./_static/images/rule_resource_1@2x.png

  点击 “新建” 按钮。

6. 返回响应动作界面，点击 “确认”。

  .. image:: ./_static/images/rule_action_4@2x.png

7. 返回规则创建界面，点击 “新建”。

  .. image:: ./_static/images/rule_overview_1@2x.png

  在规则列表里，点击 “查看” 按钮或规则 ID 连接，可以预览刚才创建的规则:

  .. image:: ./_static/images/rule_overview_2@2x.png

8. 规则已经创建完成，现在发一条数据:

    Topic: "t/a"

    QoS: 1

    Payload: "hello"

  然后检查 MySQL 表，新的 record 是否添加成功:

  .. image:: ./_static/images/mysql_result_1@2x.png

