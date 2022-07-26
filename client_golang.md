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
	cpuTemp = prometheus.NewGauge(prometheus.GaugeOpts{
		Name: "cpu_temperature_celsius",
		Help: "Current temperature of the CPU.",
	})
	hdFailures = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Name: "hd_errors_total",
			Help: "Number of hard-disk errors.",
		},
		[]string{"device"},
	)
)

func init() {
	// 必须注册指标才能公开:
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
## 使用Client_golang构建Exporter程序 ##
