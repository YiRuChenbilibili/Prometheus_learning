# PromQL聚合操作 #
Prometheus提供了下列内置的聚合操作符，这些操作符作用域瞬时向量。可以将瞬时表达式返回的样本数据进行聚合，形成一个新的时间序列：
```
      sum (求和)
      min (最小值)
      max (最大值)
      avg (平均值)
      stddev (标准差)
      stdvar (标准差异)
      count (计数)
      count_values (对value进行计数)
      bottomk (后n条时序)
      topk (前n条时序)
      quantile (分布统计)
```
聚合操作的语法如下：
```
<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
```
*其中只有count_values, quantile, topk, bottomk支持参数(parameter)。*         
without用于从计算结果中移除列举的标签，而保留其它标签。by则正好相反，结果向量中只保留列出的标签，其余标签则移除。通过without和by可以按照样本的问题对数据进行聚合。
```
sum(http_requests_total) without (instance)
//等价于
sum(http_requests_total) by (code,handler,job,method)
```

count_values用于时间序列中每一个样本值出现的次数。count_values会为每一个唯一的样本值输出一个时间序列，并且每一个时间序列包含一个额外的标签。
```
count_values("count", http_requests_total)
```
![image](https://user-images.githubusercontent.com/24589721/184279696-f1ef683a-086c-4f33-98de-cf8abfb80432.png)        

topk和bottomk则用于对样本值进行排序，返回当前样本值前n位，或者后n位的时间序列:
```
topk(5, http_requests_total)
```
![image](https://user-images.githubusercontent.com/24589721/184279886-499df93d-75c9-4d6b-b5b4-7a84d5c739b4.png)

quantile用于计算当前样本数据值的分布情况quantile(φ, express)其中0 ≤ φ ≤ 1。

例如，当φ为0.5时，即表示找到当前样本数据中的中位数：
```
quantile(0.5, http_requests_total)
```

