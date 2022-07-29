# Type #
## type Summary ##
```
type Summary interface {
	Metric
	Collector

	// Observe将单个观察结果添加到摘要中。 
	// 观察结果通常为正或为零。 
	// 负面的观测结果是可以接受的，但会阻止当前版本的Prometheus正确地检测到观测数据的计数器重置。 
	Observe(float64)
}
```
摘要从事件或样本流中捕获单个观察结果，并以类似于传统摘要统计的方式对其进行总结：1. 观察值总和，2. 观察计数，3. 排名估计。
一个典型的用例是观察请求延迟。默认情况下，摘要提供延迟的中位数、第 90 个和第 99 个百分位数作为排名估计。
要创建摘要实例，使用 NewSummary：
```
package main

import (
	"fmt"
	"math"

	"github.com/golang/protobuf/proto"

	dto "github.com/prometheus/client_model/go"

	"github.com/prometheus/client_golang/prometheus"
)

func main() {
	//func NewSummary(opts SummaryOpts ) Summary
	//NewSummary 根据提供的 SummaryOpts 创建一个新的 Summary。
	temps := prometheus.NewSummary(prometheus.SummaryOpts{
		Name:       "pond_temperature_celsius",
		Help:       "The temperature of the frog pond.",
		Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
	})

	// Simulate some observations.
	for i := 0; i < 1000; i++ {
		temps.Observe(30 + math.Floor(120*math.Sin(float64(i)*0.1))/10)
	}

	// Just for demonstration, let's check the state of the summary by
	// (ab)using its Write method (which is usually only used by Prometheus
	// internally).
	metric := &dto.Metric{}
	temps.Write(metric)
	fmt.Println(proto.MarshalTextString(metric))

}
```
**type SummaryOpts**
```
type SummaryOpts struct {
	// Namespace, Subsystem, and Name are components of the fully-qualified
	// name of the Summary (created by joining these components with
	// "_"). Only Name is mandatory, the others merely help structuring the
	// name. Note that the fully-qualified name of the Summary must be a
	// valid Prometheus metric name.
	Namespace string
	Subsystem string
	Name      string
	Help string

	ConstLabels Labels

	// 目标用它们各自的绝对误差定义了分位数秩估计 
	Objectives map[float64]float64

	// MaxAge定义了observe与summary相关的持续时间 
	MaxAge time.Duration

	// AgeBuckets是用来从summary中排除比MaxAge更老的观察结果的桶的数量。 
	AgeBuckets uint32

	// BufCap定义了默认的样本流缓冲区大小 
	BufCap uint32
}
```
