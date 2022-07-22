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
