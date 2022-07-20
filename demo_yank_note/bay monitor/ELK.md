# ELK


[Include document](https://blog.csdn.net/qq_33204709/article/details/117256185)


## elasticsearch  start issues note

::: tip elasticsearch.yml configure
* network.host: 0.0.0.0
* http.port: 9200
* node.name: node-1
* cluster.initial_master_nodes: [“node-1”]
* bootstrap.system_call_filter: false
* xpack.security.enabled: false 
:::


:::: warning 
bootstrap checks failed:max virtual memory areas vm.max_map_count [65530] is too low,increase to at least [262144]

- [ ] su - 
- [ ] vim /etc/sysctl.conf 编辑此文件 添加  vm.max_map_count=262144 在文件末尾
- [ ] sysctl -p 立即生效
::::
:::: warning
bootstrap checks failed：the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured

- [ ] vim elasticsearch.yml
- [ ] 修改 discovery下的 cluster.initial_master_nodes: ["node-1","node-2"] 
- [ ] node.name: node-1
      cluster.initial_master_nodes: ["node-1"] 
::::
:::: warning
max file descriptors [4096] for elasticsearch process is too low, increase to at least [65535]

- [ ] su -
- [ ] vim /etc/security/limits.conf
- [ ] * soft nofile 65536
      * hard nofile 65536
::::
::: tip 跨域访问
http.cors.enabled: true
http.cors.allow-origin: "*"
:::

## kibana start issue
::: tip kibana.yml configure
* server.port: 5601
* server.host: "IP"
* elasticsearch.hosts: ["http://IP:9200"]
:::

::: warning error one
'master_not_discovered_exception: [master_not_discovered_exception] Reason: null'
* vim elasticsearch.yml
* 添加 node.name: node-1
       cluster.initial_master_nodes: [“node-1”]
* restart elasticsearch
:::
