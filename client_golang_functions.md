# Functions #
**BuildFQName**   
```
func BuildFQName(namespace, subsystem, name string) string
//namespace_subsystem_name
```
BuildFQName 通过"_"连接给定的三个名称组件。空名称组件被忽略。如果 name 参数本身为空，则无论如何都会返回一个空字符串。此库中包含的Metric实施会在内部使用此函数，从而从其 Opts 中的名称组件生成完全限定的指标名称。库的用户只有在实现自己的 Metric 或直接实例化 Desc（使用 NewDesc）时才需要此功能。

**DescribeByCollect**    
```
func DescribeByCollect(c Collector, descs chan<- *Desc)
```
DescribeByCollect 是实现自定义收集器的 Describe 方法的助手。它从提供的收集器收集指标并将它们的描述符发送到提供的通道。

如果收集器在其整个生命周期中收集相同的指标，则其 Describe 方法可以简单地实现为：
```
func (c customCollector) Describe(ch chan<- *Desc) {
	DescribeByCollect(c, ch)
}
```
但是，如果收集的指标在收集器的生命周期内以它们的组合描述符集也发生变化的方式动态变化，则这将不起作用。快捷方式的实现将违反 Describe 方法的约定。如果收集器有时根本不收集任何指标（例如 CounterVec、GaugeVec 等向量，它们仅在访问具有完全指定标签集的指标后收集指标），它甚至可能被注册为未经检查的收集器（参见.Registerer接口的Register方法）。因此，仅当确定履行合同时，才使用 Describe 的快捷方式实现。   

**ExponentialBuckets**     
```
func ExponentialBuckets(start, factor float64, count int) []float64
```
ExponentialBuckets 创建“计数”桶，其中最低桶的上限为“开始”，每个后续桶的上限是“因子”乘以前一个桶的上限。最后的 +Inf 桶不计算在内，也不包含在返回的切片中。返回的切片用于 HistogramOpts 的 Buckets 字段。

如果 'count' 为 0 或负数，如果 'start' 为 0 或负数，或者如果 'factor' 小于或等于 1，则函数会发生恐慌。   
**ExponentialBucketsRange**    
```
func ExponentialBucketsRange(min, max float64 , count int ) [] float64
```
ExponentialBucketsRange 创建“count”个桶，其中最低桶是“min”，最高桶是“max”。最后的 +Inf 桶不计算在内，也不包含在返回的切片中。返回的切片用于 HistogramOpts 的 Buckets 字段。

如果 'count' 为 0 或负数，如果 'min' 为 0 或负数，该函数会发生恐慌。
**LinearBuckets**
```
func LinearBuckets(start, width float64, count int) []float64
```
LinearBuckets 创建“count”个桶，每个“width”宽，其中最低桶的上限为“start”。最后的 +Inf 桶不计算在内，也不包含在返回的切片中。返回的切片用于 HistogramOpts 的 Buckets 字段。

如果 'count' 为零或负数，该函数会发生恐慌。
**MakeLabelPair**
```
func MakeLabelPairs(desc *Desc, labelValues []string) []*dto.LabelPair
```
MakeLabelPairs 是一个辅助函数，用于从提供的 Desc 中的变量和常量标签创建 protobuf LabelPairs。变量标签的值由 labelValues 切片定义，该切片必须与 Desc 中对应的变量标签的顺序相同。

仅自定义 Metric 实现需要此功能。请参阅 MetricVec 示例。    
**MustRegister**
```
//cs为对应[]Collector的切片类型
func MustRegister(cs ...Collector)
```
MustRegister 将提供的收集器注册到 DefaultRegisterer 并在发生任何错误时发生恐慌。

MustRegister 是 DefaultRegisterer.MustRegister(cs...) 的快捷方式。    
**NewPidFileFn**
```
func NewPidFileFn(pidFilePath string ) func() ( int , error )
```
NewPidFileFn 返回一个从指定文件中检索 pid 的函数。它旨在用于 ProcessCollectorOpts 中的 PidFn 字段。    
**Register**
```
func Register(c Collector) error
```
Register 使用 DefaultRegisterer 注册提供的收集器。

Register 是 DefaultRegisterer.Register(c) 的快捷方式。  
示例：
```
package main

import (
	"fmt"
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
	// 假设您有一个工作池，希望计算已完成的任务。
	taskCounter := prometheus.NewCounter(prometheus.CounterOpts{
		Subsystem: "worker_pool",
		Name:      "completed_tasks_total",
		Help:      "Total number of tasks completed.",
	})
	// 成功注册.
	if err := prometheus.Register(taskCounter); err != nil {
		fmt.Println(err)
	} else {
		fmt.Println("taskCounter registered.")
	}
	// 不要忘记告诉HTTP服务器关于Prometheus处理程序的信息。 
	// (在真实的程序中，需要启动HTTP服务器…)
	http.Handle("/metrics", promhttp.Handler())
	
	// Now you can start workers and give every one of them a pointer to
	// taskCounter and let it increment it whenever it completes a task.
	taskCounter.Inc() // This has to happen somewhere in the worker code.

	// But wait, you want to see how individual workers perform. So you need
	// a vector of counters, with one element for each worker.
	taskCounterVec := prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Subsystem: "worker_pool",
			Name:      "completed_tasks_total",
			Help:      "Total number of tasks completed.",
		},
		[]string{"worker_id"},
	)

	// Registering will fail because we already have a metric of that name.
	if err := prometheus.Register(taskCounterVec); err != nil {
		fmt.Println("taskCounterVec not registered:", err)
	} else {
		fmt.Println("taskCounterVec registered.")
	}

	// To fix, first unregister the old taskCounter.
	if prometheus.Unregister(taskCounter) {
		fmt.Println("taskCounter unregistered.")
	}

	// Try registering taskCounterVec again.
	if err := prometheus.Register(taskCounterVec); err != nil {
		fmt.Println("taskCounterVec not registered:", err)
	} else {
		fmt.Println("taskCounterVec registered.")
	}
	// Bummer! Still doesn't work.

	// Prometheus will not allow you to ever export metrics with
	// inconsistent help strings or label names. After unregistering, the
	// unregistered metrics will cease to show up in the /metrics HTTP
	// response, but the registry still remembers that those metrics had
	// been exported before. For this example, we will now choose a
	// different name. (In a real program, you would obviously not export
	// the obsolete metric in the first place.)
	taskCounterVec = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Subsystem: "worker_pool",
			Name:      "completed_tasks_by_id",
			Help:      "Total number of tasks completed.",
		},
		[]string{"worker_id"},
	)
	if err := prometheus.Register(taskCounterVec); err != nil {
		fmt.Println("taskCounterVec not registered:", err)
	} else {
		fmt.Println("taskCounterVec registered.")
	}
	// Finally it worked!

	// The workers have to tell taskCounterVec their id to increment the
	// right element in the metric vector.
	taskCounterVec.WithLabelValues("42").Inc() // Code from worker 42.

	// Each worker could also keep a reference to their own counter element
	// around. Pick the counter at initialization time of the worker.
	myCounter := taskCounterVec.WithLabelValues("42") // From worker 42 initialization code.
	myCounter.Inc()                                   // Somewhere in the code of that worker.

	// Note that something like WithLabelValues("42", "spurious arg") would
	// panic (because you have provided too many label values). If you want
	// to get an error instead, use GetMetricWithLabelValues(...) instead.
	notMyCounter, err := taskCounterVec.GetMetricWithLabelValues("42", "spurious arg")
	if err != nil {
		fmt.Println("Worker initialization failed:", err)
	}
	if notMyCounter == nil {
		fmt.Println("notMyCounter is nil.")
	}

	// A different (and somewhat tricky) approach is to use
	// ConstLabels. ConstLabels are pairs of label names and label values
	// that never change. Each worker creates and registers an own Counter
	// instance where the only difference is in the value of the
	// ConstLabels. Those Counters can all be registered because the
	// different ConstLabel values guarantee that each worker will increment
	// a different Counter metric.
	counterOpts := prometheus.CounterOpts{
		Subsystem:   "worker_pool",
		Name:        "completed_tasks",
		Help:        "Total number of tasks completed.",
		ConstLabels: prometheus.Labels{"worker_id": "42"},
	}
	taskCounterForWorker42 := prometheus.NewCounter(counterOpts)
	if err := prometheus.Register(taskCounterForWorker42); err != nil {
		fmt.Println("taskCounterVForWorker42 not registered:", err)
	} else {
		fmt.Println("taskCounterForWorker42 registered.")
	}
	// Obviously, in real code, taskCounterForWorker42 would be a member
	// variable of a worker struct, and the "42" would be retrieved with a
	// GetId() method or something. The Counter would be created and
	// registered in the initialization code of the worker.

	// For the creation of the next Counter, we can recycle
	// counterOpts. Just change the ConstLabels.
	counterOpts.ConstLabels = prometheus.Labels{"worker_id": "2001"}
	taskCounterForWorker2001 := prometheus.NewCounter(counterOpts)
	if err := prometheus.Register(taskCounterForWorker2001); err != nil {
		fmt.Println("taskCounterVForWorker2001 not registered:", err)
	} else {
		fmt.Println("taskCounterForWorker2001 registered.")
	}

	taskCounterForWorker2001.Inc()
	taskCounterForWorker42.Inc()
	taskCounterForWorker2001.Inc()

	// Yet another approach would be to turn the workers themselves into
	// Collectors and register them. See the Collector example for details.

}
```
