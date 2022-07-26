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
	
	//现在你可以启动workers并给它们每个一个指针  
	//taskCounter，并让它在完成任务时递增。  
	//这必须发生在工作代码的某个地方。  

	//你想看看每个worker的表现。 所以你需要一个计数器向量，每个worker有一个element。 
	taskCounterVec := prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Subsystem: "worker_pool",
			Name:      "completed_tasks_total",
			Help:      "Total number of tasks completed.",
		},
		//为指标的每个特征维度定义一个label，一个label本质上就是一组键值对。
		//一个指标可以和多个label相关联，而一个指标和一组具体的label可以唯一确定一条时间序列。
		//[]string{"worker_id","type"}
		[]string{"worker_id"},
	)

	// 注册会失败，因为我们已经有了这个名称的指标。 
	if err := prometheus.Register(taskCounterVec); err != nil {
		fmt.Println("taskCounterVec not registered:", err)
	} else {
		fmt.Println("taskCounterVec registered.")
	}

	// 要修复这个问题，首先要注销旧的taskCounter。
	if prometheus.Unregister(taskCounter) {
		fmt.Println("taskCounter unregistered.")
	}

	// 再次尝试注册taskCounterVec。
	if err := prometheus.Register(taskCounterVec); err != nil {
		fmt.Println("taskCounterVec not registered:", err)
	} else {
		fmt.Println("taskCounterVec registered.")
	}
	// Bummer! 仍然不工作。


	//普罗米修斯将不允许您使用不一致的帮助字符串或标签名称export度量。取消注册后,
	//未注册的度量将不再显示在/metrics HTTP响应中
	//但注册中心仍然记得那些度量曾经export
	//对于本例，我们现在选择一个不同的名称。
	//(在真实的程序中，您显然不会首先export过时的度量。)
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

	// workers必须告诉taskCounterVec他们的worker_id来增加度量向量中正确元素。
	taskCounterVec.WithLabelValues("42").Inc() // Code from worker 42.

	// 每个worker也可以在周围保留一个对自己counter元素的引用。 在工作程序的初始化时选择计数器。 
	myCounter := taskCounterVec.WithLabelValues("42") // From worker 42 initialization code.
	myCounter.Inc()                                   // Somewhere in the code of that worker.

	//注意，像WithLabelValues("42"， "spurious arg")这样的东西会引起恐慌(因为您提供了太多的标签值)。
	//如果你想获得一个error，使用GetMetricWithLabelValues(…)代替。
	notMyCounter, err := taskCounterVec.GetMetricWithLabelValues("42", "spurious arg")
	if err != nil {
		fmt.Println("Worker initialization failed:", err)
	}
	if notMyCounter == nil {
		fmt.Println("notMyCounter is nil.")
	}

	//另一种不同的(有点棘手的)方法是使用ConstLabels。 
	//ConstLabels是永远不变的标签名称和标签值对。
	//每个工作者创建并注册一个自己的Counter  
	//实例中唯一不同的是ConstLabels的值。这些计数器都可以注册，因为  
	//不同的ConstLabel值保证每个worker将增加一个不同的Counter度量。 
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
	//显然，在实际代码中，taskCounterForWorker42将是一个worker结构的成员变量，
	//而“42”将通过GetId()方法或其他方法检索。 
	//Counter将被创建并注册到工作者的初始化代码中。 

	//为了创建下一个Counter，我们可以循环使用  
	//counterOpts。 只需更改ConstLabels。
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
