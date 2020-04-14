# Prometheus 笔记

Prometheus 

## 监控指标（Metric）

监控指标由客户端提供（格式如下），metric_name 和 label value 确定唯一的指标

```
http_requests_latencies_bucket{method="GET"} 1
http_requests_latencies_bucket{method="POST"} 2
```

Metric在数据库里的存储模式类似下面的形式

```
{__name__="http_requests_latencies_bucket", method="GET"}: (@timestamp1, value1), (@timestamp2, value2) ...

{__name__="http_requests_latencies_bucket", method="POST"}: (@timestamp1, value1), (@timestamp2, value2) ...
```

每一行称为一个Time Series（时间序列），上面的 @timestamp 就是 Prometheus 每次抓取（Scrape）监控指标的时刻。

下面所有Metric类型都会转换为一个或者多个Time Series

#### 类型

1. Counter  
	在 Time Series 生命周期，这个数值一直增加，Prometheus 通过 rate 等函数计算每秒的变化
	
	典型应用：错误数，请求数
	
	使用方式：
	
	```
	pushCounter = prometheus.NewCounter(prometheus.CounterOpts{
	    Name: "repository_pushes",
	    Help: "Number of pushes to external repository.",
	})
	pushCounter.Inc()
	```
	
2. Gauge（计量器）  
	每次抓取（Scrape）得到的都是一个独立的数值，表示当前状态
	
	典型应用：当前 Goroutine 数目，当前内存用量
	
	使用方式：
	
	```
	opsQueued := prometheus.NewGauge(prometheus.GaugeOpts{
	    Namespace: "our_company",
	    Subsystem: "blob_storage",
	    Name:      "ops_queued",
	    Help:      "Number of blob storage operations waiting to be processed.",
	})
	// Set arbitrary value.
	opsQueued.Set(10)
	// 10 operations queued by the goroutine managing incoming requests.
	opsQueued.Add(10)
	// A worker goroutine has picked up a waiting operation.
	opsQueued.Dec()
	```
	
3. Histogram（直方图）
	
	为了计算直方图，客户端提供汇总（sum），区间计数（bucket) 和 总数（count）
	
	直方图的生成由 Prometheus 查询时聚合
	
	典型应用：任务执行时长，返回数据大小
	
	会生成以下三种 Metric
	
	```
	<metric_name>_bucket{le="<upper inclusive bound>"}
	<metric_name>_sum
	<metric_name>_count
	```
	
	* _sum 监控值的累加
	* _count 监控的总次数
	* _bucket{le="0.3"} 监控值小于或等于 0.3 的总次数

	比如，假设bucket设置为 [0.2, 0.5, 1, 2]，观察（Observe）到一个值 0.6，那么
	
	```
	<metric_name>_bucket{le="1"} += 1
	<metric_name>_bucket{le="2"} += 1
	<metric_name>_bucket{le="+Inf"} += 1
	<metric_name>_sum{} += 0.6
	<metric_name>_count{} += 1
	```
	
	使用方式：

	```
	temps := prometheus.NewHistogram(prometheus.HistogramOpts{
	    Name:    "pond_temperature_celsius",
	    Help:    "The temperature of the frog pond.", // Sorry, we can't measure how badly it smells.
	    Buckets: prometheus.LinearBuckets(20, 5, 5),  // 5 buckets, each 5 centigrade wide.
	})
	for i := 0; i < 1000; i++ {
	    temps.Observe(30 + math.Floor(120*math.Sin(float64(i)*0.1))/10)
	}
	```
	
4. Summary  
	
	客户端计算汇总（sum），区间计数（bucket) 和分位数（Quantile）
	
	因为客户端对分位数计算的过程中丢失了权重等信息，在 Prometheus 的查询中聚合Quantile 是错误行为，但 Summary 提供的 Quantile 比 Histogram 的精确。
	
	分位数的计算相对更消耗客户端CPU。
	
	典型应用：任务执行时长，返回数据大小
	
	会生成以下三种Metric：  
	
	```
	<metric_name>{quantile="<φ>"}
	<metric_name>_count
	<metric_name>_sum
	```
	
	Quantile 是类似中位数的概念，区间为[0,1]，比如0.7表示在一串数字里70%的数字小于 q，这个 q 是这串数字里的一个，这个 q 就是 quantile=0.7 要求得的数值。
	
	设置Quantile的时候根据需要设置，比如想知道中位数和90%的数值落在哪个区间就设置{0.5: 0.05, 0.9: 0.01}
	
	其中的 0.05 和 0.01 是允许的计算误差。
	
	使用方式：
	
	```
	// 如果不设置 quantile 数组则客户端不会提供 <metric_name>{quantile="<φ>"} 这个维度
	temps := prometheus.NewSummary(prometheus.SummaryOpts{
	    Name:       "pond_temperature_celsius",
	    Help:       "The temperature of the frog pond.",
	    Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
	})
	
	// Simulate some observations.
	for i := 0; i < 1000; i++ {
	    temps.Observe(30 + math.Floor(120*math.Sin(float64(i)*0.1))/10)
	}
	```
	
## Go客户端集成方式

#### Step 0 import

```
go get github.com/prometheus/client_golang/prometheus
```

#### Step 1 定义指标

```
import (
	"github.com/prometheus/client_golang/prometheus"
)

var HttpRequestsLatencies = prometheus.NewHistogramVec(
	prometheus.HistogramOpts{
		Namespace: "vr",
		Name:      "http_requests_latencies",
		Help:      "Measure the http request latencies handled by VR",
		Buckets:   []float64{20, 50, 100, 200, 500, 1000}
	},
	[]string{"code", "method", "handler"},
)
```

指标定义通常为全局变量

上面代码将会产生以下这些指标，其中 metric_name 由 Namespace 和 Name 拼接而成

```
vr_http_requests_latencies_bucket{code="",handler="",method="GET",le="20.0"} 0
vr_http_requests_latencies_bucket{code="",handler="",method="GET",le="50.0"} 0
vr_http_requests_latencies_bucket{code="",handler="",method="GET",le="100.0"} 0
...
vr_http_requests_latencies_bucket{code="",handler="",method="GET",le="+Inf"} 1
vr_http_requests_latencies_sum{code="",handler="",method="GET"} 21
vr_http_requests_latencies_count{code="",handler="",method="GET"} 1
......
```

#### Step 2 注册指标

```
prometheus.MustRegister(HttpRequestsLatencies)
```

通常在 init 函数或者 main 刚开始的时候注册指标

#### Step 3 提供获取 Metric 的 API

```
import (
	"net/http"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

func main()
	...
	
	go func() {
		http.Handle("/metrics", promhttp.Handler())
		http.ListenAndServe(":9090", nil))
	}()
	...
}
```

#### Step 4 采集数据

```
start := time.Now()
...
ServeHTTP(ctx)
...

// Get label values and value
elapsed := time.Since(start)
msElapsed := elapsed / time.Millisecond
status := ctx.Get("StatusCode").(string)
handlerName := ctx.HanderName()
g.HttpRequestsLatencies.WithLabelValues(status, ctx.Request().Method, handlerName).Observe(float64(msElapsed))
```

## Recording&Alerting配置

Recording rules 用于将常用且耗时的查询结果保存到时间序列（time series），可提高查询效率，[详细文档](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)

```
groups:
  - name: example
    rules:
    - record: job:http_inprogress_requests:sum
      expr: sum(http_inprogress_requests) by (job)
```

Alerting rule 用于定期监控某个条件是否符合，并可根据当前值设定报警信息 [详细文档](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)

```
groups:
- name: example
  rules:
  - alert: HighErrorRate
    expr: job:request_latency_seconds:mean5m{job="myjob"} > 0.5
    for: 10m
    labels:
      severity: page
    annotations:
      summary: High request latency, current value: {{ $value }}
```

#### Record rule 命名

以 ```level:metric:operations``` 的方式命名，比如 ```instance_path:requests:rate5m```, ```path:request_failures_per_requests:ratio_rate5m```, 详细请看[文档](https://prometheus.io/docs/practices/rules/)

## 查询

### 概念

1. Instant vector

	某一个时刻多个Time Series的值的集合，比如：
	
	```
	<time_series_a> @timestamp1 va1
	<time_series_b> @timestamp1 vb1
	<time_series_c> @timestamp1 vc1
	```
	
2. Range vector

	某时间段多个Time Series的值的集合，比如
	
	```
	<time_series_a> @timestamp1 va1, @timestamp2 va2, @timestamp3 va3
	<time_series_b> @timestamp1 vb1, @timestamp2 vb2, @timestamp3 vb3
	<time_series_c> @timestamp1 vc1, @timestamp2 vc2, @timestamp3 vc3
	```
	
某些操作符和函数是针对 Instance vector 的，另外一些则针对 Range vector

### Example

假设有：http\_requests\_count{job="..", method="..", instance=".."}

1. 筛选指标 

	```
	// 当前
	http_requests_count{job="vr", method=~"GET|POST"}
	// 1天前
	http_requests_count{job="vr", method=~"GET|POST"} offset 1d
	```
2. 每秒请求数（QPS）

	```
	// 当前15分钟内平均值
	rate(http_requests_count{job="vr", method=~"GET|POST"}[15m])
	```
3. 汇总某个维度或者取某个维度的均值

	```
	// 所有 instance QPS 之和
	sum(rate(http_requests_count{job="vr", method=~"GET|POST"}[15m])) by (job, method)
	// 所有 instance QPS 均值
	avg(rate(http_requests_count{job="vr", method=~"GET|POST"}[15m])) by (job, method)
	```
	
4. 5分钟请求总数，每秒请求数均值，每秒请求数标准差

	```
	// 需先设置 record
	- record: instance:http_requests:rate5m
	  expr: sum without(job, method) rate(http_requests_count{job="vr"}[5m])
	  
	sum_over_time(instance:http_requests:rate5m[5m])
	avg_over_time(instance:http_requests:rate5m[5m])
	stddev_over_time(instance:http_requests:rate5m[5m])
	```
5. 错误率
	
	```
	instance:request_failures:rate5m / on(instance) instance:requests:rate5m
	```
	
6. 90% 的请求时延小于多少毫秒

	```
	histogram_quantile(0.9, sum(rate(http_request_duration_seconds_bucket[10m])) by (job, le))
	```
	
* 注：metric_name{...} 是一个 instance vector
* 注：metric_name{...}[5m] 是一个 range vector
* 注：rate 的输入是 range vector, 输出是 instance vector
* 注：sum，avg 输入是 instance vector, 输出是 instance vector
* 住：sum\_over\_time 输入是 range vector, 输出是 instance vector，所以不能直接组合 sum\_over\_time 和 rate，sum 函数，需要通过 record 将 rate, sum 的结果积累成 range vector 后才能使用 sum\_over\_time
	
#### 操作符

PromQL 的操作符有 + - * / and or == != 等，操作符两边会有几种情况

操作符两边有三种情况：

1. 两边都是数值（scalars）
2. 一边是数值，一遍是 instance vector
3. 两边都是 instance vector


前两种情况比较简单，第三种情况则相对复杂。假设有以下输入：

```
method_code:http_errors:rate5m{method="get", code="500"}  24
method_code:http_errors:rate5m{method="get", code="404"}  30
method_code:http_errors:rate5m{method="put", code="501"}  3
method_code:http_errors:rate5m{method="post", code="500"} 6
method_code:http_errors:rate5m{method="post", code="404"} 21

method:http_requests:rate5m{method="get"}  600
method:http_requests:rate5m{method="del"}  34
method:http_requests:rate5m{method="post"} 120
```

第一组和第二组的维度不一样，不能直接做算术和比较操作

可以用 ignoring() 将两组的维度不同的地方忽略，或者用 on() 确定两组的共同维度，比如

```
method_code:http_errors:rate5m{code="500"} / ignoring(code) method:http_requests:rate5m
```

表示匹配的时候忽略 code， 因为左边限定了 code="500" 所以如果两组内的time series如果匹配肯定是1对1的匹配，可以直接除得到结果，

```
method_code:http_errors:rate5m{code="500"} / on(method) method:http_requests:rate5m
```

表示匹配的时候只只要method一样，就是一致，另外也需要限定 code

上述情况，如果不限定左边的code，就会出现第一组 {method="get", code="500"} , {method="get", code="404"} 对应第二组 {method="get"} 的情况。 on 和 ignoring 只是指示匹配的字段，并不会消除或者合并其余字段。这时需要：

```
method_code:http_errors:rate5m / ignoring(code) group_left method:http_requests:rate5m
```

group_left 表示多对一的情况，按照左边的维度来运算，结果如下：

```
{method="get", code="500"}  0.04            //  24 / 600
{method="get", code="404"}  0.05            //  30 / 600
{method="post", code="500"} 0.05            //   6 / 120
{method="post", code="404"} 0.175           //  21 / 120
```

同理，在一对多的情况下，使用 group_right.

[详见文档](https://prometheus.io/docs/prometheus/latest/querying/operators/)
	


## 存储

Prometheus 2.0 开始内置了重新实现的 [tsdb](https://fabxc.org/tsdb/), Prometheus 的本地存储是短期存储，在启动参数里设置数据保留时间，默认15天。

Prometheus的磁盘占用可用以下公式计算

```
needed_disk_space = retention_time_seconds * ingested_samples_per_second * bytes_per_sample
```

其中 bytes_per_sample 大概是 1-2 bytes。

假设一个微服务实例有100个metric，2000个容器，15s采样间隔，数据保留30天

```
100 * 2000 * 30 * 24 * 3600 / 15 * 1.5 = 2160000000(B) = 48G
```

* 注：Prometheus 2.0 已经不需要设置内存相关参数，不需要设置从内存写入磁盘的相关参数。
* 注：若要持久存储只能整合远程的存储如OpenTSDB之类，并且需要 Storage Adapter

## 规范

[命名规范](https://prometheus.io/docs/practices/naming/)
	