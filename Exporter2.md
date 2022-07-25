# 常用Exporter #
## docker监控：cAdvisor ##
CAdvisor是Google开源的一款用于展示和分析容器运行状态的可视化工具。通过在主机上运行CAdvisor用户可以轻松的获取到当前主机上容器的运行统计信息，并以图表的形式向用户展示。  
在本地运行CAdvisor，运行命令：
```
docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:rw \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  google/cadvisor:latest
  ```
  
  **与Prometheus集成**   
  修改/etc/prometheus/prometheus.yml，将cAdvisor添加监控数据采集任务目标当中：   
  ```
  - job_name: cadvisor
  static_configs:
  - targets:
    - localhost:8080
  ```
启动Prometheus服务:    
```
prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/data/prometheus
```
启动完成后，可以在Prometheus UI中查看到当前所有的Target状态.    

## 监控MySQL运行状态：MySQLD Exporter ##
https://www.prometheus.wang/exporter/use-promethues-monitor-mysql.html    
**监控数据库吞吐量**     
**连接情况**    
**监控缓冲池使用情况**   
**查询性能**     
## 网络探测：Blackbox Exporter ##
从完整的监控逻辑的角度，除了大量的应用白盒监控以外，还应该添加适当的黑盒监控。黑盒监控即以用户的身份测试服务的外部可见性，常见的黑盒监控包括HTTP探针、TCP探针等用于检测站点或者服务的可访问性，以及访问效率等。   
黑盒监控相较于白盒监控最大的不同在于黑盒监控是以故障为导向当故障发生时，黑盒监控能快速发现故障，而白盒监控则侧重于主动发现或者预测潜在的问题。一个完善的监控目标是要能够**从白盒的角度发现潜在问题，能够在黑盒的角度快速发现已经发生的问题。**
![image](https://user-images.githubusercontent.com/24589721/180708746-d16719ab-4de1-43e1-8a7e-2f4a60e5ad7f.png)     
**使用Blackbox Exporter**
Blackbox Exporter是Prometheus社区提供的官方黑盒监控解决方案，其允许用户通过：HTTP、HTTPS、DNS、TCP以及ICMP的方式对网络进行探测。用户可以直接使用go get命令获取Blackbox Exporter源码并生成本地可执行文件：
```go get prometheus/blackbox_exporter```    
运行Blackbox Exporter时，需要用户提供探针的配置信息，这些配置信息可能是一些自定义的HTTP头信息，也可能是探测时需要的一些TSL配置，也可能是探针本身的验证行为。在Blackbox Exporter每一个探针配置称为一个module，并且以YAML配置文件的形式提供给Blackbox Exporter。 每一个module主要包含以下配置内容，包括探针类型（prober）、验证访问超时时间（timeout）、以及当前探针的具体配置项：
```
  # 探针类型：http、 tcp、 dns、 icmp.
  prober: <prober_string>

  # 超时时间
  [ timeout: <duration> ]

  # 探针的详细配置，最多只能配置其中的一个
  [ http: <http_probe> ]
  [ tcp: <tcp_probe> ]
  [ dns: <dns_probe> ]
  [ icmp: <icmp_probe> ]
  ```
  指定使用的探针配置文件启动Blockbox Exporter实例：
  ```
  blackbox_exporter --config.file=/etc/prometheus/blackbox.yml
  ```
启动成功后，就可以通过访问http://127.0.0.1:9115/probe?module=http_2xx&target=baidu.com 对baidu.com进行探测。这里通过在URL中提供module参数指定了当前使用的探针(http_2xx)，target参数指定探测目标(=baidu.com)，探针的探测结果通过Metrics的形式返回。     
从返回的样本中，用户可以获取站点的DNS解析耗时、站点响应时间、HTTP响应状态码等等和站点访问质量相关的监控指标，从而帮助管理员主动的发现故障和问题。



