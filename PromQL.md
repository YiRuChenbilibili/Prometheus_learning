# PromQL #
通过PromQL用户可以非常方便地对监控样本数据进行统计分析，PromQL支持常见的运算操作符，同时PromQL中还提供了大量的内置函数可以实现对数据的高级处理。   
## PromQL基础 ##
**查询时间序列**
直接使用监控指标名称查询时，可以查询该指标下的所有时间序列。     
``` nginx_http_requests_total/ nginx_http_requests_total{}  ```       
PromQL还支持用户根据时间序列的标签匹配模式来对时间序列进行过滤，目前主要支持两种匹配模式：完全匹配和正则匹配。     

完全匹配：      
   1、通过使用label=value可以选择那些标签满足表达式定义的时间序列；    
   2、反之使用label!=value则可以根据标签匹配排除时间序列；   
   
正则匹配（多个表达式之间使用|进行分离）：   
   1、使用label=~regx表示选择那些标签符合正则表达式定义的时间序列；       
   2、反之使用label!~regx进行排除；

**范围查询**     
如果想知道过去一段时间范围内的样本数据时，需要使用区间向量表达式。区间向量表达式和瞬时向量表达式之间的差异在于在区间向量表达式中我们需要定义时间选择的范围，时间范围通过时间范围选择器[]进行定义(m代表分钟，h代表小时）。例如：  
``` nginx_http_requests_total[2m]```      
![image](https://user-images.githubusercontent.com/24589721/183388296-ba094564-7f5d-4cf5-88b4-b75b38f24ab0.png)     
*通过区间向量表达式查询到的结果我们称为区间向量。*     

**时间位移操作**      
在瞬时向量表达式或者区间向量表达式中，都是以当前时间为基准。如果想查询5分钟前的瞬时样本数据，或昨天一天的区间内的样本数据时。可以使用位移操作，位移操作的关键字为offset：    
``` nginx_http_requests_total[2m] offset 1d```       
![image](https://user-images.githubusercontent.com/24589721/183390745-ca9a10fe-1f06-49f9-955e-8abfb0a001d8.png)

**聚合操作**     
如果描述样本特征的标签(label)在并非唯一的情况下，通过PromQL查询数据，会返回多条满足这些特征维度的时间序列。而PromQL提供的聚合操作可以用来对这些时间序列进行处理，形成一条新的时间序列：    
```
# 查询系统所有http请求的总量
sum(http_request_total)

# 按照mode计算主机CPU的平均使用时间
avg(node_cpu) by (mode)

# 按照主机查询各个主机的CPU使用率
sum(sum(irate(node_cpu{mode!='idle'}[5m]))  / sum(irate(node_cpu[5m]))) by (instance)
```




