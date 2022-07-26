# Functions #
```
func BuildFQName(namespace, subsystem, name string) string
```
BuildFQName 通过"_"连接给定的三个名称组件。空名称组件被忽略。如果 name 参数本身为空，则无论如何都会返回一个空字符串。此库中包含的Metric实施会在内部使用此函数，从而从其 Opts 中的名称组件生成完全限定的指标名称。库的用户只有在实现自己的 Metric 或直接实例化 Desc（使用 NewDesc）时才需要此功能。
