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
