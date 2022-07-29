# Type #
## type Histogram ##
```
type Histogram interface {
	Metric
	Collector
  //Observe向直方图添加单个观察结果。 
  //观察结果通常为正或为零。 
  //负面的观测结果是可以接受的，但会阻止当前版本的普罗米修斯正确地检测到观测数据的计数器重置 
	Observe(float64)
}
```
