# Type #
## type Summary ##
```
type Summary interface {
	Metric
	Collector

Observe将单个观察结果添加到摘要中。 观察结果通常为正或为零。 负面的观测结果是可以接受的，但会阻止当前版本的Prometheus正确地检测到观测数据的计数器重置。 
	Observe(float64)
}
```
