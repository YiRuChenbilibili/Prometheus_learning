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
Histogram计算来自可配置存储桶中事件或样本流的单个观察值。与summary类似，它还提供观察值总和和观察值计数。在 Prometheus 服务器上，可以使用查询语言中的 histogram_quantile 函数从直方图计算分位数。     
与summary相比，直方图可以使用 Prometheus 查询语言进行聚合（有关详细过程，请参阅文档）。然而，Histogram需要用户预先定义合适的桶，而且它们通常不太准确。与summary的观察方法相比，Histogram的观察方法具有非常低的性能开销。要创建直方图实例，使用 NewHistogram:
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
	temps := prometheus.NewHistogram(prometheus.HistogramOpts{
		Name:    "pond_temperature_celsius",
		Help:    "The temperature of the frog pond.", 
		Buckets: prometheus.LinearBuckets(20, 5, 5),  // 5个桶，每个桶宽5摄氏度
	})

	//模拟观察
	for i := 0; i < 1000; i++ {
		//Observe向直方图添加单个观察结果
		temps.Observe(30 + math.Floor(120*math.Sin(float64(i)*0.1))/10)
	}

	//为了演示，让通过(ab)使用它的Write方法(通常只在Prometheus内部使用)来检查直方图的状态。 
	metric := &dto.Metric{}
	temps.Write(metric)
	fmt.Println(proto.MarshalTextString(metric))

}
```
**func NewHistogram**
```
func NewHistogram(opts HistogramOpts) Histogram
```
NewHistogram 根据提供的 HistogramOpts 创建一个新的 Histogram。如果 HistogramOpts 中的桶不是严格递增的顺序，它会恐慌。返回的实现还实现了 ExemplarObserver。执行相应的类型断言是安全的。为每个桶单独跟踪示例。    

**type HistogramOpts**
```
type HistogramOpts struct {
	// Namespace, Subsystem, 和Name 是直方图全名的组成部分
	//将全名的各个组件用 "_"连接). 
	//只有Name是强制的，其他的只是帮助构造
	Namespace string
	Subsystem string
	Name      string

	// 帮助提供关于此直方图的信息。
	//
	// 具有相同全限定名称的指标必须具有相同的帮助字符串。
	Help string

	// ConstLabels用于在这个度量值上附加固定的标签。 
	// 指标具有相同全限定名称的标签必须在其ConstLabels中具有相同的标签名称。 
	ConstLabels Labels

	// Buckets定义了统计观察结果的桶。
	// 切片中的每个元素都是一个桶的上限。 
	// 这些值必须严格按照递增的顺序排序。 
	Buckets []float64
}
```

**type HistogramVec**
```
type HistogramVec struct {
	*MetricVec
}
```
HistogramVec 是一个收集器，它捆绑了一组直方图，这些直方图都共享相同的 Desc，但它们的变量标签具有不同的值。如果您想计算按不同维度划分的同一事物（例如 HTTP 请求延迟，按状态代码和方法划分），则使用此选项。使用 NewHistogramVec 创建实例。

**func NewHistogramVec**
```
func NewHistogramVec(opts HistogramOpts , labelNames [] string ) * HistogramVec
```
NewHistogramVec 基于提供的 HistogramOpts 创建一个新的 HistogramVec，并按给定的标签名称进行分区。

*Histogram与Counter及Gauge一样，使用CurryWith/ MustCurryWith 返回一个带有提供标签的向量;      
使用GetMetricWith/with返回给定标签的直方图（标签名称必须与 Desc 中的变量标签匹配）；使用 GetMetricWithLabelValues/WithLabelValues返回给定标签值切片的直方图（与 Desc 中的变量标签顺序相同）（如果第一次访问该标签，则会创建一个新的直方图。）*
