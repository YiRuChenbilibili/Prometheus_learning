# Prometheus告警处理 #
告警能力在Prometheus的架构中被划分为两个部分，在Prometheus Server中定义告警规则以及产生告警，Alertmanager组件则用于处理这些由Prometheus产生的告警。Alertmanager即Prometheus体系中告警的统一处理中心。Alertmanager提供了多种内置第三方告警通知方式。    
通过在Prometheus中定义AlertRule（告警规则），Prometheus会周期性的对告警规则进行计算，如果满足告警触发条件就会向Alertmanager发送告警信息。 
![image](https://user-images.githubusercontent.com/24589721/184827490-4a8d993a-6c1e-4d72-a726-16b9ca60c867.png)

