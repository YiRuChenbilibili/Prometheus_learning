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

