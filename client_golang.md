# client_golang #
client_golang是Prometheus针对go语言的client library库，我们可以直接基于client_java用户可以快速实现独立运行的Exporter程序，也可以在我们的项目源码中集成client_golang以支持Prometheus。      
包 prometheus 是核心检测包。它为检测代码提供度量原语以进行监控。它还提供了一个指标注册表。子包允许通过 HTTP（包 promhttp）公开注册的指标或将它们推送到 Pushgateway（包推送）。还有一个子包 promauto，它为度量构造函数提供自动注册。    
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
	//计数器
	cpuTemp = prometheus.NewGauge(prometheus.GaugeOpts{
		Name: "cpu_temperature_celsius",
		Help: "Current temperature of the CPU.",
	})
	//仪表
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
## 使用Client_golang构建Exporter程序 ##
除了基本的度量类型 Gauge、Counter、Summary 和 Histogram 之外，Prometheus 数据模型的一个非常重要的部分是沿着称为标签的维度对样本进行分区，从而产生度量向量。基本类型是 GaugeVec、CounterVec、SummaryVec 和 HistogramVec。     
虽然只有基本的度量类型实现了 Metric 接口，但度量和它们的向量版本都实现了 Collector 接口。一个 Collector 管理多个 Metrics 的收集，但为了方便起见，一个 Metric 也可以“收集自己”。需要注意，Gauge、Counter、Summary 和 Histogram 本身是接口，而 GaugeVec、CounterVec、SummaryVec 和 HistogramVec 不是。      
要创建 Metrics 及其矢量版本的实例，则需要一个合适的Opts 结构，即 GaugeOpts、CounterOpts、SummaryOpts 或 HistogramOpts。