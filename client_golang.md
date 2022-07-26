# client_golang #
client_golang是Prometheus针对go语言的client library库，我们可以直接基于client_java用户可以快速实现独立运行的Exporter程序，也可以在我们的项目源码中集成client_golang以支持Prometheus。      
包 prometheus 是核心检测包。它为检测代码提供度量原语以进行监控。它还提供了一个指标注册表。子包允许通过 HTTP（包 promhttp）公开注册的指标或将它们推送到 Pushgateway（包推送）。还有一个子包 promauto，它为度量构造函数提供自动注册。    

## 使用Client_golang构建Exporter程序 ##
基础例子：
```
package main

import (
	"log"
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	//仪表
	cpuTemp = prometheus.NewGauge(prometheus.GaugeOpts{
		Name: "cpu_temperature_celsius",
		Help: "Current temperature of the CPU.",
	})
	//计数器
	hdFailures = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Name: "hd_errors_total",
			Help: "Number of hard-disk errors.",
		},
		[]string{"device"},
	)
)

func init() {
	// 必须注册指标才能公开,否则HTTPServer找不到任何的Collector实例
	prometheus.MustRegister(cpuTemp)
	prometheus.MustRegister(hdFailures)
}

func main() {
	cpuTemp.Set(65.3)
	hdFailures.With(prometheus.Labels{"device":"/dev/sda"}).Inc()

	// Handler函数提供了一个默认值
	// 通过 HTTP 服务器公开指标的处理程序。“/metrics”是通常的端点。
	http.Handle("/metrics", promhttp.Handler())
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```
这是一个完整的程序，它导出两个指标，一个 Gauge 和一个 Counter，后者带有一个附加标签以将其转换为（一维）向量。     

**Metrics（度量）**     
除了基本的度量类型 Gauge、Counter、Summary 和 Histogram 之外，Prometheus 数据模型的一个非常重要的部分是沿着称为标签的维度对样本进行分区，从而产生度量向量。基本类型是 GaugeVec、CounterVec、SummaryVec 和 HistogramVec。     

虽然只有基本的度量类型实现了 Metric 接口，但度量和它们的向量版本都实现了 Collector 接口。一个 Collector 管理多个 Metrics 的收集，但为了方便起见，一个 Metric 也可以“收集自己”。需要注意，Gauge、Counter、Summary 和 Histogram 本身是接口，而 GaugeVec、CounterVec、SummaryVec 和 HistogramVec 不是。      

要创建 Metrics 及其矢量版本的实例，则需要一个合适的Opts 结构，即 GaugeOpts、CounterOpts、SummaryOpts 或 HistogramOpts。     

**HTTP Exposition**     
Registry 实现了 Gatherer 接口。然后，Gather 方法的调用者可以以某种方式公开收集的指标。通常，指标是通过 /metrics 端点上的 HTTP 提供的。这发生在上面的例子中。通过 HTTP 公开指标的工具位于 promhttp 子包中。 

**注册表的使用**    
MustRegister 是迄今为止最常用的注册收集器的方法。正如名称所暗示的那样，如果发生错误，MustRegister 就会发生恐慌。使用Register函数，错误被返回并且可以被处理。       

如果注册的 Collector 与已注册的指标不兼容或不一致，则会返回错误。注册表旨在根据Prometheus数据模型使收集到的指标保持一致。理想情况下，在注册时检测到不一致，而不是在收集时检测到不一致。前者通常会在程序启动时被检测到，而后者只会在初始化时被检测到，甚至可能不会在第一次初始化时被检测到（如果不一致的部分只是后来才相关的话）。这就是 Collector 和 Metric 必须向注册表描述自己的主要原因。   

使用 NewRegistry，可以创建自定义注册表，甚至可以自己实现 Registerer 或 Gatherer 接口。方法 Register 和 Unregister 在自定义注册表上的工作方式与全局函数 Register 和 Unregister 在默认注册表上的工作方式相同。     

**推送到 Pushgateway**    
推送到 Pushgateway 的功能可以在 push 子包中找到。   

**Graphite Bridge**     
将指标从 Gatherer 推送到 Graphite 的函数和示例可以在 Graphite 子包中找到。    
## 常量 ##
SummaryOpts 的默认值。
```
const (
 	// DefMaxAge 是观察保持相关的默认持续时间。 
	DefMaxAge time . Duration = 10 * time . Minute
	// DefAgeBuckets 是用于计算// 观察年龄的默认桶数。	
	DefAgeBuckets = 5
	// DefBufCap 是用于收集汇总观察的标准缓冲区大小。
	DefBufCap = 500
)
```
ExemplarMaxRunes 是示例标签中允许的最大符文总数。      
```
const ExemplarMaxRunes = 64
```

## 变量 ##
DefaultRegisterer 和 DefaultGatherer 是 Registerer 和 Gatherer 接口的实现，该包中的许多便利功能都起作用。
```
var (
 	DefaultRegisterer Registerer = defaultRegistry
 	DefaultGatherer    Gatherer    = defaultRegistry
 )
```
DefBuckets 是默认的直方图存储桶。默认存储桶专门用于广泛测量网络服务的响应时间（以秒为单位）。(然而更多的情况下，需要定义针对用例定制的存储桶。）
```
var (
 	DefBuckets = [] float64 {.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10}
 )
```
