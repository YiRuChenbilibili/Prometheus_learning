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
