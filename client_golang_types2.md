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

## type Observer ##
```
type Observer interface {
	Observe(float64)
}
```
Observer是封装了Observe方法的接口，Histogram和Summary使用该方法来添加观察结果。 

**type ObserverFunc**
```
type ObserverFunc func(float64)
```
`ObserverFunc` 类型是一个适配器，允许将普通函数用作观察者。如果 f 是具有适当签名的函数，则 `ObserverFunc(f)` 是调用 f 的 Observer。
此适配器通常与 `Timer` 类型结合使用，一般有两种用例：最常见的一种是使用 `Gauge` 作为 `Timer` 的 Observer;更高级的用例是创建一个函数，动态决定使用哪个 Observer 来观察持续时间。

**func (ObserverFunc) Observe**
```
func (f ObserverFunc) Observe(value float64)
```
`Observe`调用 f(value)。它实现了普通函数的`Observer`。

## type ObserverVec ##
```
type ObserverVec interface {
	GetMetricWith(Labels) (Observer, error)
	GetMetricWithLabelValues(lvs ...string) (Observer, error)
	With(Labels) Observer
	WithLabelValues(...string) Observer
	CurryWith(Labels) (ObserverVec, error)
	MustCurryWith(Labels) ObserverVec

	Collector
}
```
ObserverVec 是由 `HistogramVec` 和 `SummaryVec` 实现的接口。

## type Registerer ##
```
type Registerer interface {
	Register(Collector) error
	
	// MustRegister的工作原理类似于Register，但可以注册任意数量的collector。
	MustRegister(...Collector)
	
	Unregister(Collector) bool
}
```
Registerer （注册器）是注册表中负责注册和注销部分的接口。自定义注册表的用户应使用 Registerer 作为注册类型（而不是直接使用 Registry 类型）。这样，他们就可以自由地使用自定义注册器实现（例如，用于测试目的）。

**func WrapRegistererWith**
```
func WrapRegistererWith(labels Labels , reg Registerer ) Registerer
```
WrapRegistererWith 返回一个包装(提供的注册器)的注册器。使用返回的注册器注册的收集器将以修改的方式注册到包装的注册器。修改后的收集器将提供的标签添加到它收集的所有指标（作为 ConstLabels）。未经修改的 Collector 收集的 Metrics 不得重复任何这些标签。包装 nil 值是有效的，导致无操作注册器。

**func WrapRegistererWithPrefix**
```
func WrapRegistererWithPrefix(prefix string, reg Registerer) Registerer
```
WrapRegistererWithPrefix 返回包装提供的注册器的注册器。使用返回的注册器注册的收集器将以修改的方式注册到包装的注册器。修改后的 Collector 将提供的前缀添加到它收集的所有 Metrics 的名称中。包装 nil 值是有效的，导致无操作注册器。

WrapRegistererWithPrefix 在一个位置为子系统的所有指标添加前缀很有用。为了使这个工作，使用 WrapRegistererWithPrefix 返回的包装注册器注册子系统的度量。对所有公开的指标使用相同的前缀很少有用。

## type Registry ##
```
type Registry struct {
	// contains filtered or unexported fields
}
```
Registry (注册表）注册 Prometheus 收集器，收集它们的指标，并将它们收集到 MetricFamilies 中以供展示。它实现了 Registerer 和 Gatherer。零值不可用。使用 NewRegistry 或 NewPedanticRegistry 创建实例。     

**func NewPedanticRegistry**
```
func NewPedanticRegistry() *Registry
```
NewPedanticRegistry 返回一个注册表，该注册表在收集期间检查每个收集的 Metric 是否与其报告的 Desc 一致，以及 Desc 是否实际上已在注册表中注册。未经检查的收集器（那些其 Describe 方法不产生任何描述符的收集器）被排除在检查之外。

**func NewRegistry**
```
func NewRegistry() *Registry
```
NewRegistry 创建一个新的 vanilla Registry，没有预先注册任何收集器。

![image](https://user-images.githubusercontent.com/24589721/181715426-f5e303e3-ec0e-44c2-a850-e60aef75d5f2.png)


