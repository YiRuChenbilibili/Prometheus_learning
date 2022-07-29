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

## type Labels ##
```
type Labels map[string]string
```
Labels 表示标签名称 -> 值映射的集合。这种类型通常与度量向量收集器的 With(Labels) 和 GetMetricWith(Labels) 方法一起使用，例如：
```
myVec.With(Labels{"code": "404", "method": "GET"}).Add(42)
```
另一个用例是在 Opts 中指定常量标签对或创建 Desc。

## type Metric ##
```
type Metric interface {
	// Desc 返回 Metric 的描述符。
	Desc() *Desc
	// 建议按字典顺序对标签进行排序。Write的调用者仍然应该确保排序，如果它们依赖于排序的话。 
	Write(*dto.Metric) error
}
```
Metric 对单个样本值进行建模，并将其元数据导出到 Prometheus。此包中 Metric的实现包括 Gauge、Counter、Histogram、Summary 和 Untyped。     
**func NewConstHistogram/func NewConstMetric/func NewConstSummary**        
NewConstHistogram 返回一个表示 Prometheus 直方图的指标，该直方图的计数、总和和存储桶计数具有固定值。由于这些参数无法更改，因此返回值未实现 Histogram 接口（仅实现 Metric 接口）。此软件包的用户在常规操作中不会有太多用处。但是，在实现自定义收集器时，它可用作即时生成的一次性指标，以在 Collect 方法中将其发送到 Prometheus。其它几个函数也是相似的作用。     

**type MetricVec** 
```
type MetricVec struct {
	 // 包含过滤或未导出的字段
}
```
MetricVec 是一个收集器，用于捆绑标签值不同的同名指标。MetricVec 不直接使用，而是作为给定度量类型的向量实现的构建块，例如 GaugeVec、CounterVec、SummaryVec 和 HistogramVec。它被导出，以便可用于**自定义 Metric 实现。**      
要为自定义 Metric Foo 创建 FooVec，请在 FooVec 中嵌入指向 MetricVec 的指针，并使用 NewMetricVec 对其进行初始化。同时要给自定义的Metric添加GetMetricWithLabelValues/GetMetricWith/CurryWith等等方法以及便捷方法。
## type MultiError ##
```
type MultiError []error
```
MultiError 是实现错误接口的错误片段。Gatherer 使用它来报告 MetricFamily 收集期间的多个错误。

**func (\*MultiError) Append**
```
func (errs *MultiError) Append(err error)
```
如果不为零，则附加附加提供的错误。     

**func (MultiError) Error**
```
func (errs MultiError) Error() string
```
Error将包含的错误格式化为项目符号列表，前面是错误总数。请注意，这会导致多行字符串。    

**func (MultiError) MaybeUnwrap**
```
func (errs MultiError ) MaybeUnwrap() error
```
如果 len(errs) 为 0，MaybeUnwrap 返回 nil。如果 len(errs 为 1)，它返回第一个且唯一包含的错误作为错误。在所有其他情况下，它直接返回 MultiError。这有助于以仅在需要时使用 MultiError 的方式返回 MultiError。
