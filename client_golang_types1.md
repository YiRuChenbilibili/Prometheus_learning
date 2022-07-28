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
Collector是由 Prometheus 可以用来收集指标的任何东西实现的接口。收集者必须注册收集。    
这个包提供的度量（Gauge、Counter、Summary、Histogram、Untyped）也是收集器（它只收集一个指标，即它自己）。然而，收集器的实现者可以以协调的方式收集多个指标（和/或）动态创建指标。已经在这个库中实现的Collector的示例是度量向量（即相同度量但具有不同标签值的多个实例的集合），如 GaugeVec 或 SummaryVec，以及 ExpvarCollector。    
**示例**
```
package main

import (
	"log"
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

type ClusterManager struct {
	Zone string
	// Contains many more fields not listed in this example.
}

// 对真实集群管理器所进行的数据收集的模拟。
func (c *ClusterManager) ReallyExpensiveAssessmentOfTheSystemState() (
	oomCountByHost map[string]int, ramUsageByHost map[string]float64,
) {
	// Just example fake data.
	oomCountByHost = map[string]int{
		"foo.example.org": 42,
		"bar.example.org": 2001,
	}
	ramUsageByHost = map[string]float64{
		"foo.example.org": 6.023e23,
		"bar.example.org": 3.14,
	}
	return
}

// ClusterManagerCollector实现Collector接口
//需要实现一个Describe和一个collect
type ClusterManagerCollector struct {
	ClusterManager *ClusterManager
}

// ClusterManagerCollector使用的描述符如下。 
var (
	oomCountDesc = prometheus.NewDesc(
		"clustermanager_oom_crashes_total",
		"Number of OOM crashes.",
		[]string{"host"}, nil,
	)
	ramUsageDesc = prometheus.NewDesc(
		"clustermanager_ram_usage_bytes",
		"RAM usage as reported to the cluster manager.",
		[]string{"host"}, nil,
	)
)

//description使用DescribeByCollect实现。这是可能的，因为Collect方法将总是返回具有相同两个描述符的相同两个指标。 
func (cc ClusterManagerCollector) Describe(ch chan<- *prometheus.Desc) {
	prometheus.DescribeByCollect(cc, ch)
}

// Collect首先触发reallyexpensiveassessmentofsystemstate。然后，它根据返回的数据动态地为每个主机创建常量指标。 
//
// 注意，Collect可以并发调用，因此我们依赖于reallyexpensiveassessmentofsystemstate是并发安全的 
func (cc ClusterManagerCollector) Collect(ch chan<- prometheus.Metric) {
	oomCountByHost, ramUsageByHost := cc.ClusterManager.ReallyExpensiveAssessmentOfTheSystemState()
	for host, oomCount := range oomCountByHost {
		ch <- prometheus.MustNewConstMetric(
			oomCountDesc,
			prometheus.CounterValue,
			float64(oomCount),
			host,
		)
	}
	for host, ramUsage := range ramUsageByHost {
		ch <- prometheus.MustNewConstMetric(
			ramUsageDesc,
			prometheus.GaugeValue,
			ramUsage,
			host,
		)
	}
}

// NewClusterManager首先创建一个Prometheus-ignorant的ClusterManager实例。 
//然后，它为刚刚创建的ClusterManager创建一个ClusterManagerCollector。
//最后，它使用包装寄存器注册ClusterManagerCollector，包装寄存器将区域添加为标签。
//这样，不同的clustermanagercollector收集的指标就不会发生冲突。 
func NewClusterManager(zone string, reg prometheus.Registerer) *ClusterManager {
	c := &ClusterManager{
		Zone: zone,
	}
	cc := ClusterManagerCollector{ClusterManager: c}
	prometheus.WrapRegistererWith(prometheus.Labels{"zone": zone}, reg).MustRegister(cc)
	return c
}

func main() {
	// 注册
	reg := prometheus.NewPedanticRegistry()

	// 构建集群管理器。在实际代码中，我们会将它们赋值给变量，然后使用它们做一些事情。 
	NewClusterManager("db", reg)
	NewClusterManager("ca", reg)

	// 向自定义注册中心添加标准流程和Go指标,添加 process 和 Go 运行时指标到我们自定义的注册表中。
	reg.MustRegister(
		prometheus.NewProcessCollector(prometheus.ProcessCollectorOpts{}),
		prometheus.NewGoCollector(),
	)

	http.Handle("/metrics", promhttp.HandlerFor(reg, promhttp.HandlerOpts{}))
	log.Fatal(http.ListenAndServe(":8080", nil))
}
``` 
##  Type Counter ##
```
type Counter interface {
	Metric 
	Collector 
	// Inc 将计数器递增 1。使用 Add 将其递增
	// 非负值。
	Inc() 
	// Add 将给定值添加到计数器。如果值为 <  0 ，它会恐慌。 
	Add( float64 ) 
}	
```
Counter 是一个 Metric，它代表一个只会上升的单个数值。这意味着它不能用于计算数量也可能下降的项目，例如当前运行的 goroutine 的数量。这些“计数器”由Gauges表示。
计数器通常用于对服务的请求、完成的任务、发生的错误等进行计数。要创建 Counter 实例，需要使用 NewCounter。    
**func NewCounter**
```
func NewCounter(opts CounterOpts) Counter
```
NewCounter 根据提供的 CounterOpts 创建一个新的 Counter。返回的实现还实现了 ExemplarAdder。
## type CounterFunc ##
```
type CounterFunc interface {
	Metric
	Collector
}
```
CounterFunc 是一个计数器，其值在收集时通过调用提供的函数来确定。要创建 CounterFunc 实例，使用 NewCounterFunc。     

**func NewCounterFunc**
```
func NewCounterFunc(opts CounterOpts , function func() float64 ) CounterFunc
```
NewCounterFunc 基于提供的 CounterOpts 创建一个新的 CounterFunc。报告的值是通过从 Write 方法中调用给定函数来确定的。

## type CounterOpts ##
```
type CounterOpts Opts
```
CounterOpts 是 Opts 的别名。
## type CounterVec ##
```
type CounterVec struct {
	*MetricVec
}
```
CounterVec 是一个Collector，它捆绑了一组Counters，这些Counter都共享相同的 Desc，但它们的变量标签具有不同的值。如果您想计算按不同维度划分的同一事物（例如 HTTP 请求的数量，按响应代码和方法划分），则使用此方法。使用 NewCounterVec 创建实例。

**示例**
```
package main

import (
	"github.com/prometheus/client_golang/prometheus"
)

func main() {
	//NewCounterVec是一个Collector,绑定了一组Counter（相同度量但具有不同标签值的多个实例的集合）
	httpReqs := prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Name: "http_requests_total",
			Help: "How many HTTP requests processed, partitioned by status code and HTTP method.",
		},
		[]string{"code", "method"},
	)
	prometheus.MustRegister(httpReqs)
	
	//与GetMetricWithLabels类似，在 GetMetricWithLabels 会返回错误的地方出现恐慌。
	//WithLabelValues返回给定标签值切片的计数器（与 Desc 中的变量标签顺序相同）。
	//如果第一次访问该标签值组合，则会创建一个新计数器。
	//可以在不使用返回的 Counter 的情况下调用此方法，仅创建新的 Counter，但将其保留为起始值 0。
	//标签为"404", "POST"
	httpReqs.WithLabelValues("404", "POST").Add(42)

	//标签为"200", "GET"
	m := httpReqs.WithLabelValues("200", "GET")
	for i := 0; i < 1000000; i++ {
		m.Inc()
	}
	//从vector中删除一个度量(不同的两种方法)
	httpReqs.DeleteLabelValues("200", "GET")
	httpReqs.Delete(prometheus.Labels{"method": "GET", "code": "200"})
}
```
## type Desc ##
```
type Desc struct {
	// contains filtered or unexported fields
}
```
Desc 是每个 Prometheus Metric 使用的描述符。它本质上是 Metric 的不可变元数据。此包中包含的常规 Metric 实现在后台管理它们的 Desc。如果用户使用 ExpvarCollector 或自定义Collector和Metric等高级功能，他们只需要处理 Desc。     
如果在同一注册表中注册的描述符具有相同的完全限定名称，则它们必须满足某些一致性和唯一性标准：它们必须在每个 constLabels 和 variableLabels 中具有相同的帮助字符串和相同的标签名称（也称为标签尺寸），但它们constLabels 的值必须不同。     

**func NewDesc**
```
func NewDesc(fqName, help string , variableLabels [] string , constLabels Labels ) * Desc
```
NewDesc 分配并初始化一个新的 Desc。错误记录在 Desc 中，并将在注册时报告。如果不应设置此类标签，则 variableLabels 和 constLabels 可以为 nil。fqName 不能为空。     

## type Gatherer ##
```
type Gatherer interface {
	//将收集到的指标收集到一个按字典顺序排序的切片中 
	Gather() ([]*dto.MetricFamily, error)
}
```
Gatherer 是注册表中负责将收集到的指标收集到多个 MetricFamilies 中的部分的接口。Gatherer 接口具有与 Registerer 接口相同的一般含义。    
```
type Gatherers []Gatherer
```
Gatherers 是 Gatherer 实例的一部分，它实现了 Gatherer 接口本身。它的 Gather 方法按顺序调用切片中所有 Gatherer 上的 Gather，并返回合并后的结果。   
Gatherers 可用于合并来自多个注册表的 Gather 结果。它还提供了一种将现有 MetricFamily protobufs 直接注入到收集中的方法，方法是使用 Gather 方法创建一个自定义 Gatherer，该方法简单地返回现有 MetricFamily protobufs。   

**示例**
```
package main

import (
	"bytes"
	"fmt"
	"strings"

	"github.com/prometheus/common/expfmt"

	dto "github.com/prometheus/client_model/go"

	"github.com/prometheus/client_golang/prometheus"
)

func main() {
	reg := prometheus.NewRegistry()
	temp := prometheus.NewGaugeVec(
		prometheus.GaugeOpts{
			Name: "temperature_kelvin",
			Help: "Temperature in Kelvin.",
		},
		[]string{"location"},
	)
	reg.MustRegister(temp)
	temp.WithLabelValues("outside").Set(273.14)
	temp.WithLabelValues("inside").Set(298.44)

	var parser expfmt.TextParser

	text := `
# TYPE humidity_percent gauge
# HELP humidity_percent Humidity in %.
humidity_percent{location="outside"} 45.4
humidity_percent{location="inside"} 33.2
# TYPE temperature_kelvin gauge
# HELP temperature_kelvin Temperature in Kelvin.
temperature_kelvin{location="somewhere else"} 4.5
`

	parseText := func() ([]*dto.MetricFamily, error) {
		parsed, err := parser.TextToMetricFamilies(strings.NewReader(text))
		if err != nil {
			return nil, err
		}
		var result []*dto.MetricFamily
		for _, mf := range parsed {
			result = append(result, mf)
		}
		return result, nil
	}

	gatherers := prometheus.Gatherers{
		reg,
		prometheus.GathererFunc(parseText),
	}

	gathering, err := gatherers.Gather()
	if err != nil {
		fmt.Println(err)
	}

	out := &bytes.Buffer{}
	for _, mf := range gathering {
		if _, err := expfmt.MetricFamilyToText(out, mf); err != nil {
			panic(err)
		}
	}
	fmt.Print(out.String())
	fmt.Println("----------")

	// Note how the temperature_kelvin metric family has been merged from
	// different sources. Now try
	text = `
# TYPE humidity_percent gauge
# HELP humidity_percent Humidity in %.
humidity_percent{location="outside"} 45.4
humidity_percent{location="inside"} 33.2
# TYPE temperature_kelvin gauge
# HELP temperature_kelvin Temperature in Kelvin.
# Duplicate metric:
temperature_kelvin{location="outside"} 265.3
 # Missing location label (note that this is undesirable but valid):
temperature_kelvin 4.5
`

	gathering, err = gatherers.Gather()
	if err != nil {
		fmt.Println(err)
	}
	// Note that still as many metrics as possible are returned:
	out.Reset()
	for _, mf := range gathering {
		if _, err := expfmt.MetricFamilyToText(out, mf); err != nil {
			panic(err)
		}
	}
	fmt.Print(out.String())

}
```
