# Types #
## type AlreadyRegisteredError ##
```
type AlreadyRegisteredError struct {
	ExistingCollector, NewCollector Collector
}
```
如果要注册的 Collector 之前已经注册过，或者之前已经注册过收集相同指标的不同 Collector，Register 方法会返回 AlreadyRegisteredError。在这种情况下注册会失败，但您可以从错误类型中检测到发生了什么。该错误包含现有收集器和等于现有收集器的（已拒绝）新收集器的字段。这可用于查明之前是否注册了相同的收集器并切换到使用旧的收集器，如示例中所示。
```
package main

import (
	"github.com/prometheus/client_golang/prometheus"
)

func main() {
	reqCounter := prometheus.NewCounter(prometheus.CounterOpts{
		Name: "requests_total",
		Help: "The total number of requests served.",
	})
	//尝试注册
	if err := prometheus.Register(reqCounter); err != nil {
		if are, ok := err.(prometheus.AlreadyRegisteredError); ok {
			// 该指标的计数器以前已经注册过。
			// 从现在开始用旧计数器。 
			reqCounter = are.ExistingCollector.(prometheus.Counter)
		} else {
			// Something else went wrong!
			panic(err)
		}
	}
	reqCounter.Inc()
}
```
**func (AlreadyRegisteredError) Error**
```
func (err AlreadyRegisteredError) Error() string
```
## type Collector ##
```
type Collector interface {
        // Describe 将由此收集器收集的所有可能的指标描述符的超集
	// 如果收集器在执行此方法时遇到错误，它
	// 必须发送一个无效描述符（使用 NewInvalidDesc 创建）来
	// 向注册表发出错误信号。
	Describe(chan<- *Desc)
	// 在收集指标时，Prometheus 注册表调用 Collect。
	// 该实现通过提供的通道发送每个收集到的metric，并在发送最后一个metric后返回。
	// 每个已发送度量的描述符是Describe返回的描述符之一
	Collect(chan<- Metric)
}
```
