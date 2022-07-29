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

	// MaxAge定义了observations与summary相关的持续时间 
	MaxAge time.Duration

	// AgeBuckets是用来从summary中排除比MaxAge更老的观察结果的桶的数量。 
	AgeBuckets uint32

	// BufCap定义了默认的样本流缓冲区大小 
	BufCap uint32
}
```
SummaryOpts 捆绑了用于创建摘要指标的选项。必须将 Name 设置为非空字符串。虽然所有其他字段都是可选的并且可以安全地保留为零值，但建议设置帮助字符串并将 Objectives 字段显式设置为所需的值。
## Type SummaryVec ##
```
type SummaryVec struct {
	*MetricVec
}
```
SummaryVec 是一个收集器，它捆绑了一组共享相同 Desc 的汇总，但它们的变量标签具有不同的值。如果想计算按不同维度划分的同一事物（例如 HTTP 请求延迟，按状态代码和方法划分），则使用此选项。使用 NewSummaryVec 创建实例:
```
package main

import (
	"fmt"
	"math"

	"github.com/golang/protobuf/proto"

	"github.com/prometheus/client_golang/prometheus"
)

func main() {
	temps := prometheus.NewSummaryVec(
		prometheus.SummaryOpts{
			Name:       "pond_temperature_celsius",
			Help:       "The temperature of the frog pond.",
			Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
		},
		[]string{"species"},
	)

	// 模拟一些 observations.
	for i := 0; i < 1000; i++ {
		temps.WithLabelValues("litoria-caerulea").Observe(30 + math.Floor(120*math.Sin(float64(i)*0.1))/10)
		temps.WithLabelValues("lithobates-catesbeianus").Observe(32 + math.Floor(100*math.Cos(float64(i)*0.11))/10)
	}

	// 创建一个不带任何observations的Summary
	temps.WithLabelValues("leiopelma-hochstetteri")

	// 为了演示检查汇总向量的状态
 	//通过向自定义注册中心注册它，然后让它收集指标。
	reg := prometheus.NewRegistry()
	reg.MustRegister(temps)

	metricFamilies, err := reg.Gather()
	if err != nil || len(metricFamilies) != 1 {
		panic("unexpected behavior of custom test registry")
	}
	fmt.Println(proto.MarshalTextString(metricFamilies[0]))

}
```
## type Timer ##
```
type Timer struct {
	// contains filtered or unexported fields
}
```
Timer 是时间函数的辅助类型。使用 NewTimer 创建新实例

**示例（简单）**
```
package main

import (
	"math/rand"
	"time"

	"github.com/prometheus/client_golang/prometheus"
)

var (
	requestDuration = prometheus.NewHistogram(prometheus.HistogramOpts{
		Name:    "example_request_duration_seconds",
		Help:    "Histogram for the runtime of a simple example function.",
		Buckets: prometheus.LinearBuckets(0.01, 0.01, 10),
	})
)

func main() {
	// 模拟一些observationstimer times在这个示例函数中。 
	//它使用了直方图，但也可以使用摘要，因为两者都实现了Observer 
	//NewTimer 创建一个新的 Timer。提供的 Observer 用于观察持续时间（以秒为单位）。
	timer := prometheus.NewTimer(requestDuration)
	defer timer.ObserveDuration()

	// Do something here that takes time.
	time.Sleep(time.Duration(rand.NormFloat64()*10000+50000) * time.Microsecond)
}
```
**示例（复杂）**
```
package main

import (
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
)

var (
	// apiRequestDuration跟踪每个HTTP状态分开的持续时间  
	// class (1xx, 2xx，…) 这就产生了相当多的时间序列
	// 分区的请求计数器状态代码通常是OK的，因为每个计数器只创建一次  
	// 直方图的开销要大得多，所以要小心分区 
	apiRequestDuration = prometheus.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "api_request_duration_seconds",
			Help:    "Histogram for the request duration of the public API, partitioned by status class.",
			Buckets: prometheus.ExponentialBuckets(0.1, 1.5, 5),
		},
		[]string{"status_class"},
	)
)

func handler(w http.ResponseWriter, r *http.Request) {
	status := http.StatusOK
	// ObserverFunc被延迟的ObserveDuration调用，并决定调用哪个Histogram的Observe方法。 
	timer := prometheus.NewTimer(prometheus.ObserverFunc(func(v float64) {
		switch {
		//不同的状态码
		case status >= 500: // Server error.
		//Observe添加观察值v
			apiRequestDuration.WithLabelValues("5xx").Observe(v)
		case status >= 400: // Client error.
			apiRequestDuration.WithLabelValues("4xx").Observe(v)
		case status >= 300: // Redirection.
			apiRequestDuration.WithLabelValues("3xx").Observe(v)
		case status >= 200: // Success.
			apiRequestDuration.WithLabelValues("2xx").Observe(v)
		default: // Informational.
			apiRequestDuration.WithLabelValues("1xx").Observe(v)
		}
	}))
	defer timer.ObserveDuration()

	// Handle the request. Set status accordingly.
	// ...
}

func main() {
	http.HandleFunc("/api", handler)
}
```
