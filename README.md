# achieve-kube-scheduler

## K8S - 创建一个 kube-scheduler 插件
[https://mp.weixin.qq.com/s/8sQ_ZaRqQRz-D2lm8lWtkA](https://mp.weixin.qq.com/s/8sQ_ZaRqQRz-D2lm8lWtkA)
简单地说，K8S 调度器负责将 Pod 分配给Node 。一旦创建了新的 pod，它就会进入调度队列。调度 pod 的尝试分为两个阶段：调度和绑定周期。
在调度周期中，节点被过滤，删除那些不符合 Pod 要求的节点。接下来，可行节点（其余节点）根据给定分数进行排名。最后，选择得分最高的节点。这些步骤称为过滤和评分[1]
一旦选择了一个节点，调度程序需要确保kubelet知道它需要在所选节点中启动 pod（容器）。与将 pod 启动到选定节点相关的步骤称为Binding Cycle[2].
调度和绑定周期由按顺序执行以计算 pod 放置的阶段组成。这些阶段称为扩展点，可用于塑造布局行为。不同 pod 的调度周期是按顺序运行的，这意味着调度周期步骤将一次针对一个 pod 执行，而不同 pod 的Binding Cycles可以同时执行。
实现 kubernetes 调度器扩展点的组件称为插件。本机调度行为也使用插件模式实现，与自定义扩展相同，使 kube-scheduler 的核心轻量级，作为主要调度逻辑放置在插件中。
插件可以应用的扩展点如图1所示。一个插件可以实现一个或多个扩展点，每个扩展点的详细描述可以在[4]中找到（我不会在这里复制东西，检查在继续之前先把它放在那里😃）。
![](https://cdn.nlark.com/yuque/0/2022/png/26096423/1646471684899-546628c8-0585-403a-9d92-0473a5740c95.png#clientId=u53bc9389-8d64-4&crop=0&crop=0&crop=1&crop=1&from=paste&id=u8a053ceb&margin=%5Bobject%20Object%5D&originHeight=485&originWidth=1080&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=uac646bba-9905-4a44-a1ad-a386efa3832&title=)
为了配置应该在每个扩展点执行的插件，然后改变调度行为，kube-scheduler 提供了Profiles [3] 。调度配置文件描述了应该在 [4] 中提到的每个阶段执行哪些插件。可以提供多个配置文件，这意味着无需部署多个调度程序来具有不同的调度行为 [5]。
#### kube-scheduler
kube-scheduler 是在 Golang 中实现的，并且插件在编译时包含在其中。因此，如果您想拥有自己的插件，则需要拥有自己的调度程序镜像。
需要注册一个新插件并将其配置到插件 API。此外，它需要实现kubernetes 调度程序框架包中定义的扩展点接口。查看它的外观：
​

> kubernetes 调度程序框架https://github.com/kubernetes/kubernetes/blob/ed3e0d302fb546653b78df583569b0311687a7a8/pkg/scheduler/framework/interface.go#L268

```go
// Plugin是所有调度框架插件的父类型。
type Plugin interface {
   Name() string
}

type QueueSortPlugin interface {
  Plugin 
  Less(*QueuedPodInfo, *QueuedPodInfo) bool
}

type PreFilterPlugin interface {
  Plugin
  PreFilter(CycleState, *v1.Pod) *Status
  PreFilterExtensions() PreFilterExtensions
}
```
调度程序的代码允许添加新插件而无需分叉。main()为此，开发人员只需要围绕调度程序编写自己的包装器。由于必须使用调度程序编译插件，因此编写包装器允许以干净的方式重用调度程序的代码 [7]。
​

为此，主函数将导入k8s.io/kubernetes/cmd/kube-scheduler/app并使用NewSchedulerCommand注册自定义插件，提供相应的名称和构造函数：
```go
import (
    "k8s.io/kubernetes/cmd/kube-scheduler/app"
)

func main() {
    command := app.NewSchedulerCommand(
      app.WithPlugin("example-plugin1", ExamplePlugin1.New),
      app.WithPlugin("example-plugin2", ExamplePlugin2.New))
    if err := command.Execute(); err != nil {
        fmt.Fprintf(os.Stderr, "%v\n", err)
        os.Exit(1)
    }
}
```
#### 配置
kube-scheduler 配置是可以配置配置文件的地方。每个配置文件都允许根据插件定义的配置参数启用、禁用和配置插件。每个配置文件配置分为两部分 [9]：
​

每个扩展点的启用插件列表以及它们应该运行的顺序。如果省略扩展点列表之一，将使用默认列表。
每个插件的一组可选的自定义插件参数。省略插件的配置参数等同于使用该插件的默认配置。
​

在不同扩展点中启用的插件必须在每个插件中显式配置。配置是通过KubeSchedulerConfiguration结构提供的。要启用它，需要将其写入配置文件，并将其路径作为命令行参数提供给 kube-scheduler。例如：
> KubeSchedulerConfiguration结构: [https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1beta1/#kubescheduler-config-k8s-io-v1beta1-KubeSchedulerConfiguration](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1beta1/#kubescheduler-config-k8s-io-v1beta1-KubeSchedulerConfiguration)

```go
kube-scheduler --config=/etc/kubernetes/networktraffic-config.yaml
```


NetworkTraffic下面您可以看到插件的示例配置。在示例中，clientConnection.kubeconfig指向 kube-scheduler 使用的 kubeconfig 路径，以及在控制平面节点中定义的授权。该profiles部分覆盖了default-scheduler分数阶段，启用NetworkTraffic插件并禁用默认定义的其他插件。设置插件的pluginConfig配置，将在其初始化期间提供[8]。
```go
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/etc/kubernetes/scheduler.conf"
profiles:
- schedulerName: default-scheduler
  plugins:
    score:
      enabled:
      - name: NetworkTraffic
      disabled:
      - name: "*"
  pluginConfig:
  - name: NetworkTraffic
    args:
      prometheusAddress: "http://prometheus-1616380099-server.monitor"
      networkInterface: "ens192"
      timeRangeInMinutes: 3
```
> PS：如果您有多个控制平面节点的 HA，则需要为每个节点应用配置。

#### 创建自定义插件
现在我们了解了 kube-scheduler 的基础知识，我们可以做我们来这里的目的了。正如我们之前所见，添加自定义插件需要在编译期间包含我们的代码，我们不需要为此分叉调度程序代码。
​

为了继续，我们可以创建一个空的存储库并如前所述包装调度程序，但是，项目调度程序插件已经这样做了，并提供了一些自定义插件，这些插件是很好的示例。所以，我们将从那里开始。
​

fork scheduler-plugins存储库并将其拉入$GOPATH/src/sigs.k8s.io. 完成后，我们可以开始了:)
​

要继续执行后续步骤，您需要：有一个 K8S 集群（我使用的是用kubespray创建的集群）。
​

使用 node-exporter 配置 prometheus。检查kube-prometheus-stack。
> 调度程序插件: https://github.com/kubernetes-sigs/scheduler-plugins

> kubespray: https://github.com/kubernetes-sigs/kubespray

> kube-prometheus-stack: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

#### 网络流量插件
对于这个例子，我们将构建一个名为“NetworkTraffic”的评分插件，它有利于网络流量较低的节点。为了收集这些信息，我们将查询 prometheus。
首先，在调度程序插件的分支中创建文件夹和文件pkg/networktraffic。结构应如下所示：networktraffic.goprometheus.go
```go
|- pkg 
|-- networktraffic 
|--- networktraffic.go 
|--- prometheus.go
```
我们将networktraffic.go实现 ScorePlugin 接口，并prometheus.go保留与 prometheus 交互的逻辑。
#### Prometheus communication
在本文中，prometheus.go我们将首先声明用于与 Prometheus 交互的结构。它将具有字段networkInterface和timeRange，可用于配置我们将执行的查询。该字段address指向K8S上的prometheus服务，也可以配置。该字段api将用于存储 prometheus 客户端，该客户端是基于address提供的。
```go
type PrometheusHandle struct { 
  networkInterface string 
  timeRange time.Duration 
  address string 
  api v1.API 
}
```
现在我们有了基本的结构，我们也可以实现查询了。我们将使用特定网络接口中每个节点在某个时间范围内接收到的字节的总和。过滤器将查询所提供节点的kubernetes_node指标，如下面的查询所述。device过滤器将查询提供的网络接口上的指标，中间的最后一个值定义[%s]考虑的时间范围。sum_over_time将对提供的时间范围内的所有值求和。
```go
sum_over_time(node_network_receive_bytes_total{kubernetes_node=\"%s\",device=\"%s\"}[%s])
```
最后，prometheus.go文件将如下所示：
```go
package networktraffic

import (
 "context"
 "fmt"
 "time"

 "github.com/prometheus/client_golang/api"
 v1 "github.com/prometheus/client_golang/api/prometheus/v1"
 "github.com/prometheus/common/model"
 "k8s.io/klog/v2"
)

const (
 // nodeMeasureQueryTemplate is the template string to get the query for the node used bandwidth
 nodeMeasureQueryTemplate = "sum_over_time(node_network_receive_bytes_total{kubernetes_node=\"%s\",device=\"%s\"}[%s])"
)

// Handles the interaction of the networkplugin with Prometheus
type PrometheusHandle struct {
 networkInterface string
 timeRange        time.Duration
 address          string
 api              v1.API
}

func NewPrometheus(address, networkInterface string, timeRange time.Duration) *PrometheusHandle {
 client, err := api.NewClient(api.Config{
  Address: address,
 })
 if err != nil {
  klog.Fatalf("[NetworkTraffic] Error creating prometheus client: %s", err.Error())
 }

 return &PrometheusHandle{
  networkInterface: networkInterface,
  timeRange:        timeRange,
  address:          address,
  api:              v1.NewAPI(client),
 }
}

func (p *PrometheusHandle) GetNodeBandwidthMeasure(node string) (*model.Sample, error) {
 query := getNodeBandwidthQuery(node, p.networkInterface, p.timeRange)
 res, err := p.query(query)
 if err != nil {
  return nil, fmt.Errorf("[NetworkTraffic] Error querying prometheus: %w", err)
 }

 nodeMeasure := res.(model.Vector)
 if len(nodeMeasure) != 1 {
  return nil, fmt.Errorf("[NetworkTraffic] Invalid response, expected 1 value, got %d", len(nodeMeasure))
 }

 return nodeMeasure[0], nil
}

func getNodeBandwidthQuery(node, networkInterface string, timeRange time.Duration) string {
 return fmt.Sprintf(nodeMeasureQueryTemplate, node, networkInterface, timeRange)
}

func (p *PrometheusHandle) query(query string) (model.Value, error) {
 results, warnings, err := p.api.Query(context.Background(), query, time.Now())

 if len(warnings) > 0 {
  klog.Warningf("[NetworkTraffic] Warnings: %v\n", warnings)
 }

 return results, err
}
```
#### 
#### ScorePlugin 接口


完成与 Prometheus 的交互后，我们可以转到 Score Plugin 的实现。如前所述，我们需要从调度程序框架中实现 Score Plugin Interface：
```go
// ScorePlugin is an interface that must be implemented by "Score" 
// plugins to rank nodes that passed the filtering phase.
type ScorePlugin interface {
  Plugin
  
  // Score is called on each filtered node. It must return success 
  // and an integer indicating the rank of the node. All scoring
  // plugins must return success or the pod will be rejected.
  Score(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string) (int64, *Status)
  
  // ScoreExtensions returns a ScoreExtensions interface if it 
  // implements one, or nil if does not.
  ScoreExtensions() ScoreExtensions
}

// ScoreExtensions is an interface for Score extended functionality.
type ScoreExtensions interface {
  // NormalizeScore is called for all node scores produced by the
  // same plugin's "Score" method. A successful run of 
  // NormalizeScore will update the scores list and return a success       
  // status.
  NormalizeScore(ctx context.Context, state *CycleState, p *v1.Pod, scores NodeScoreList) *Status
}
```


为每个节点调用该Score函数并返回它是否成功和一个指示节点排名的整数。在 Score 插件执行结束时，我们应该有一个从 0 到 100 范围内的 Score 值。例如，在某些情况下，在不知道其他节点的分数的情况下，可能很难在该范围内获得一个值。对于那些场景，我们可以使用接口NormalizeScore中实现的ScoreExtensions功能。该NormalizeScore函数接收所有节点的结果并允许更改它们。
​

此外，ScorePlugin 接口也有Plugin作为嵌入字段的接口。所以，我们必须实现它的Name() string功能。
> 嵌入字段的接口: https://travix.io/type-embedding-in-go-ba40dd4264df

现在我们了解了 ScorePlugin 接口，让我们进入networktraffic.go文件。我们将从定义NetworkTraffic结构开始：
```go
// NetworkTraffic is a score plugin that favors nodes based on their
// network traffic amount. Nodes with less traffic are favored.
// Implements framework.ScorePlugin
type NetworkTraffic struct {
  handle     framework.FrameworkHandle
  prometheus *PrometheusHandle
}
```
定义好结构后，我们就可以进行Score功能实现了。这将是直截了当的。我们只会GetNodeBandwidthMeasure从提供节点名称的 Prometheus 结构中调用该函数。该调用将返回一个Sample保存Value字段中的值的值。我们基本上会为每个节点返回它。
```go

func (n *NetworkTraffic) Score(ctx context.Context, state *framework.CycleState, p *v1.Pod, nodeName string) (int64, *framework.Status) {
 nodeBandwidth, err := n.prometheus.GetNodeBandwidthMeasure(nodeName)
 if err != nil {
  return 0, framework.NewStatus(framework.Error, fmt.Sprintf("error getting node bandwidth measure: %s", err))
 }

 klog.Infof("[NetworkTraffic] node '%s' bandwidth: %s", nodeName, nodeBandwidth.Value)
 return int64(nodeBandwidth.Value), nil
}
```
接下来，我们将返回每个节点在确定的时间段内接收到的总字节数。但是，调度程序框架需要一个从 0 到100的值，因此，我们仍然需要对这些值进行归一化来满足这个要求。
​

为了进行规范化，我们将实现ScoreExtensions前面提到的接口。我们将实现嵌入在NetworkTraffic结构中的接口。在ScoreExtensions函数中，我们将简单地返回实现接口的结构。逻辑放在NormalizeScore函数下面。
```go
func (n *NetworkTraffic) ScoreExtensions() framework.ScoreExtensions {
 return n
}

func (n *NetworkTraffic) NormalizeScore(ctx context.Context, state *framework.CycleState, pod *v1.Pod, scores framework.NodeScoreList) *framework.Status {
 var higherScore int64
 for _, node := range scores {
  if higherScore < node.Score {
   higherScore = node.Score
  }
 }

 for i, node := range scores {
  scores[i].Score = framework.MaxNodeScore - (node.Score * framework.MaxNodeScore / higherScore)
 }

 klog.Infof("[NetworkTraffic] Nodes final score: %v", scores)
 return nil
}
```


基本上会取prometheus返回的NormalizeScore最高值作为可能的最高值，对应framework.MaxNodeScore(100)。其他值将使用三规则相对于最高分数进行计算。
最后，我们将有一个列表，其中具有更多网络流量的节点在 [0,100] 范围内具有更高的分数。如果我们按原样使用它，我们会偏爱具有更高流量的节点，因此，我们需要反转这些值。为此，我们将简单地将节点分数替换为三规则的结果，减去最大分数。以三个节点（ a、b和c ）为例，其值以字节为单位，计算示例如下：
```go
a => 1000000   # 1MB
b => 1200000   # 1,2MB
c => 1400000   # 1,4MB
higherScore = 1400000
Y = (node.Score * framework.MaxNodeScore) / higherScore
Ya = 1000000 * 100 / 1400000
Yb = 1200000 * 100 / 1400000
Yc = 1400000 * 100 / 1400000
Ya = 71,42
Yb = 85,71
Yc = 100
Xa = 100 - Ya
Xb = 100 - Yb
Xc = 100 - Yc
Xa = 28,58
Xb = 14,29
Xc = 0
```


有了这个解释，我们就有了插件的主要部分。然而，这还不是全部。如前所述，可以配置调度程序插件，我们将在我们的网络流量插件中允许三种配置，这些配置已经提到：

- Prometheus 地址
- Prometheus 查询时间范围
- Prometheus 查询节点网络接口

这些值将由调度程序框架在插件实例化期间提供NetworkTraffic，我们将需要声明一个名为的新结构NetworkTrafficArgs，该结构将用于解析KubeSchedulerConfiguration. 为此，我们需要添加一个具有创建NetworkTraffic插件实例的逻辑的新函数，如下所述：
```go
// New initializes a new plugin and returns it.
func New(obj runtime.Object, h framework.FrameworkHandle) (framework.Plugin, error) {
 args, ok := obj.(*config.NetworkTrafficArgs)
 if !ok {
  return nil, fmt.Errorf("want args to be of type NetworkTrafficArgs, got %T", obj)
 }

 return &NetworkTraffic{
  handle:     h,
  prometheus: NewPrometheus(args.Address, args.NetworkInterface, time.Minute*time.Duration(args.TimeRangeInMinutes)),
 }, nil
}
```
该New函数遵循调度程序框架PluginFactory接口。
> PluginFactory接口: https://github.com/kubernetes/kubernetes/blob/4aae71695a8dd43918702fafd81e0401721d79d9/pkg/scheduler/framework/runtime/registry.go#L29

我们还没有声明NetworkTrafficArgs结构，接下来会出现。但是，我们（几乎）拥有我们所需要的一切networktraffic.go：
```go
package networktraffic

import (
 "context"
 "fmt"
 "time"

 v1 "k8s.io/api/core/v1"
 "k8s.io/apimachinery/pkg/runtime"
 "k8s.io/klog/v2"
 framework "k8s.io/kubernetes/pkg/scheduler/framework/v1alpha1"
 "sigs.k8s.io/scheduler-plugins/pkg/apis/config"
)

// NetworkTraffic is a score plugin that favors nodes based on their
// network traffic amount. Nodes with less traffic are favored.
// Implements framework.ScorePlugin
type NetworkTraffic struct {
 handle     framework.FrameworkHandle
 prometheus *PrometheusHandle
}

// Name is the name of the plugin used in the Registry and configurations.
const Name = "NetworkTraffic"

var _ = framework.ScorePlugin(&NetworkTraffic{})

// New initializes a new plugin and returns it.
func New(obj runtime.Object, h framework.FrameworkHandle) (framework.Plugin, error) {
 args, ok := obj.(*config.NetworkTrafficArgs)
 if !ok {
  return nil, fmt.Errorf("[NetworkTraffic] want args to be of type NetworkTrafficArgs, got %T", obj)
 }

 klog.Infof("[NetworkTraffic] args received. NetworkInterface: %s; TimeRangeInMinutes: %d, Address: %s", args.NetworkInterface, args.TimeRangeInMinutes, args.Address)

 return &NetworkTraffic{
  handle:     h,
  prometheus: NewPrometheus(args.Address, args.NetworkInterface, time.Minute*time.Duration(args.TimeRangeInMinutes)),
 }, nil
}

// Name returns name of the plugin. It is used in logs, etc.
func (n *NetworkTraffic) Name() string {
 return Name
}

func (n *NetworkTraffic) Score(ctx context.Context, state *framework.CycleState, p *v1.Pod, nodeName string) (int64, *framework.Status) {
 nodeBandwidth, err := n.prometheus.GetNodeBandwidthMeasure(nodeName)
 if err != nil {
  return 0, framework.NewStatus(framework.Error, fmt.Sprintf("error getting node bandwidth measure: %s", err))
 }

 klog.Infof("[NetworkTraffic] node '%s' bandwidth: %s", nodeName, nodeBandwidth.Value)
 return int64(nodeBandwidth.Value), nil
}

func (n *NetworkTraffic) ScoreExtensions() framework.ScoreExtensions {
 return n
}

func (n *NetworkTraffic) NormalizeScore(ctx context.Context, state *framework.CycleState, pod *v1.Pod, scores framework.NodeScoreList) *framework.Status {
 var higherScore int64
 for _, node := range scores {
  if higherScore < node.Score {
   higherScore = node.Score
  }
 }

 for i, node := range scores {
  scores[i].Score = framework.MaxNodeScore - (node.Score * framework.MaxNodeScore / higherScore)
 }

 klog.Infof("[NetworkTraffic] Nodes final score: %v", scores)
 return nil
}
```


#### 配置
scheduler-plugins 项目将配置保存在pkg/apis文件夹下。所以，我们也将在那里进行我们的插件配置。
​

我们将在两个地方添加配置：pkg/apis/config/types.go和pkg/apis/config/v1beta1/types.go. config/types.go保存我们将在New函数中使用的结构，而保存v1beta1/types.go用于解析来自KubeSchedulerConfiguration. 此外，配置结构必须遵循名称模式<Plugin Name>Args，否则将无法正确解码，您将面临问题。
```go
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// NetworkTrafficArgs holds arguments used to configure NetworkTraffic plugin.
type NetworkTrafficArgs struct {
 metav1.TypeMeta

 // Address of the Prometheus Server
 Address string
 // NetworkInterface to be monitored, assume that nodes OS is homogeneous
 NetworkInterface string
 // TimeRangeInMinutes used to aggregate the network metrics
 TimeRangeInMinutes int64
}
```
```go
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +k8s:defaulter-gen=true

// NetworkTrafficArgs holds arguments used to configure NetworkTraffic plugin.
type NetworkTrafficArgs struct {
 metav1.TypeMeta `json:",inline"`

 // Address of the Prometheus Server
 Address *string `json:"prometheusAddress,omitempty"`
 // NetworkInterface to be monitored, assume that nodes OS is homogeneous
 NetworkInterface *string `json:"networkInterface,omitempty"`
 // TimeRangeInMinutes used to aggregate the network metrics
 TimeRangeInMinutes *int64 `json:"timeRangeInMinutes,omitempty"`
}
```


添加结构后，我们需要执行hack/update-codegen.sh脚本。它将使用DeepCopy添加结构的功能更新生成的文件。此外，我们将SetDefaultNetworkTrafficArgs在config/v1beta1/defaults.go. 该函数将为NetworkInterface和值设置默认值TimeRangeInMinutes，但Address仍需要提供。
```go
// SetDefaultNetworkTrafficArgs sets the default parameters for the NetworkTraffic plugin
func SetDefaultNetworkTrafficArgs(args *NetworkTrafficArgs) {
 if args.TimeRangeInMinutes == nil {
  defaultTime := int64(5)
  args.TimeRangeInMinutes = &defaultTime
 }

 if args.NetworkInterface == nil || *args.NetworkInterface == "" {
  netInterface := "ens192"
  args.NetworkInterface = &netInterface
 }
}

```
要完成默认值配置，我们需要确保上述函数已在 v1beta1 模式中注册。因此，请确保它已在文件中注册pkg/apis/config/v1beta1/zz_generated.defaults.go。
```go
package v1beta1

import (
 runtime "k8s.io/apimachinery/pkg/runtime"
)

// RegisterDefaults adds defaulters functions to the given scheme.
// Public to allow building arbitrary schemes.
// All generated defaulters are covering - they call all nested defaulters.
func RegisterDefaults(scheme *runtime.Scheme) error {
 scheme.AddTypeDefaultingFunc(&NetworkTrafficArgs{}, func(obj interface{}) {
  SetObjectDefaultNetworkTrafficArgs(obj.(*NetworkTrafficArgs))
 })

 return nil
}

func SetObjectDefaultNetworkTrafficArgs(in *NetworkTrafficArgs) {
 SetDefaultNetworkTrafficArgs(in)
}
```


#### 注册插件和配置
现在已经定义了参数结构，我们的插件就准备好了。但是，我们仍然需要在调度程序框架中注册插件和配置。
​

scheduler-plugins 项目已经注册了几个插件，这使得事情变得更容易，因为我们有例子。插件配置的注册放在pkg/apis/config. 在文件register.go中，我们需要在对函数NetworkTrafficArgs的调用中添加。AddKnownTypes同样需要在pkg/apis/config/v1beta1/register.go文件中完成。更改两个文件后，配置注册就完成了。
​

接下来，我们转到插件注册，这是在cmd/scheduler/main.go文件中完成的。在main函数中，NetworkTraffic插件名称和构造函数需要作为参数提供给NewSchedulerCommand. 它应该如下所示：
```go
command := app.NewSchedulerCommand(
  app.WithPlugin(networktraffic.Name, networktraffic.New),
)
```
另外，请注意在main.go文件中我们有 的导入sigs.k8s.io/scheduler-plugins/pkg/apis/config/scheme，它使用我们在文件中引入的所有配置来初始化方案pkg/apis/config。
​

> main.go : https://github.com/juliorenner/scheduler-plugins/blob/921af74d339e99842d48bec88c27778749cff32b/cmd/scheduler/main.go#L29

这样我们就从代码的角度完成了。完整的实现可以在这里找到，它还包括几个单元测试，所以看看吧！
> 完整实现：https://github.com/juliorenner/scheduler-plugins

#### 部署和使用插件
现在我们已经完成了插件，我们可以将它部署在我们的 K8S 集群中并开始使用它。在 scheduler-plugins 存储库中，有一个关于如何执行此操作的文档，请在此处查看。我们基本上需要使用我们刚刚实现的插件来调整这些步骤。
> 请在此处查看: https://github.com/kubernetes-sigs/scheduler-plugins/blob/master/doc/install.md

尽管如此，在将更改应用到集群之前，请确保您已构建调度程序容器镜像并将其推送到可从您的 kubernetes 访问的容器仓库。我不会详细介绍，因为它会根据使用的环境而有所不同。您也可以检查 Makefile，因为有一些命令可以构建和推送镜像，而且这个开发文档可能会对您有所帮助。
> 开发文档：https://github.com/kubernetes-sigs/scheduler-plugins/blob/master/doc/develop.md#how-to-build

由于我们的插件没有引入任何 CRD，因此可以跳过scheduler-plugins 安装文档中的几个步骤。正如我所提到的，我正在使用通过kubespray创建的集群和 HA。因此，我需要在每个控制平面节点上重复以下步骤。
1、 登录控制平面节点。
2、 备份kube-scheduler.yaml
> https://github.com/kubernetes-sigs/scheduler-plugins/blob/master/doc/install.md

```go
cp /etc/kubernetes/manifests/kube-scheduler.yaml /etc/kubernetes/kube-scheduler.yaml
```
/etc/kubernetes/networktraffic-config.yaml3、根据您的环境创建和更改值。
```go
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/etc/kubernetes/scheduler.conf"
profiles:
- schedulerName: default-scheduler
  plugins:
    score:
      enabled:
      - name: NetworkTraffic
      disabled:
      - name: "*"
  pluginConfig:
  - name: NetworkTraffic
    args:
      prometheusAddress: "http://prometheus-1616380099-server.monitor"
      networkInterface: "ens192"
      timeRangeInMinutes: 3
```


4、修改/etc/kubernetes/manifests/kube-scheduler.yaml以运行带有网络流量的调度程序插件。我们所做的更改是：

- 添加命令 arg --config=/etc/kubernetes/networktraffic-config.yaml。
- 更改image名称。
- 添加一个volume指向配置的绝对路径。
- 添加一个volumeMount以使配置可用于调度程序 pod。检查以下示例：
```go
apiVersion: v1
kind: Pod
metadata:
  labels:
    component: kube-scheduler
    tier: control-plane
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=0.0.0.0
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    - --leader-elect-lease-duration=15s
    - --leader-elect-renew-deadline=10s
    - --port=0
    - --config=/etc/kubernetes/networktraffic-config.yaml
    image: <YOUR_CONTAINER_REGISTRY>/scheduler-plugins/kube-scheduler:<YOUR_TAG>
    imagePullPolicy: Always
    livenessProbe:
      failureThreshold: 8
      httpGet:
        path: /healthz
        port: 10259
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    name: kube-scheduler
    resources:
      requests:
        cpu: 100m
    startupProbe:
      failureThreshold: 30
      httpGet:
        path: /healthz
        port: 10259
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    volumeMounts:
    - mountPath: /etc/kubernetes/scheduler.conf
      name: kubeconfig
      readOnly: true
    - mountPath: /etc/kubernetes/networktraffic-config.yaml
      name: networktraffic-config
      readOnly: true
  hostNetwork: true
  priorityClassName: system-node-critical
  volumes:
  - hostPath:
      path: /etc/kubernetes/scheduler.conf
      type: FileOrCreate
    name: kubeconfig
  - hostPath:
      path: /etc/kubernetes/networktraffic-config.yaml
      type: File
    name: networktraffic-config
```
现在，我们可以开始利用我们的自定义插件了。检查运行 pod 的日志后，您应该会看到从prometheus返回的节点带宽行，您可以确保行为符合预期。下面，我们可以看到node4正确的得分较高，因为它是网络流量较少的节点：
![](https://cdn.nlark.com/yuque/0/2022/png/26096423/1646471685038-086d1ab1-5789-4f5c-9d7b-eff70d669e35.png#clientId=u53bc9389-8d64-4&crop=0&crop=0&crop=1&crop=1&from=paste&id=u1510a8d6&margin=%5Bobject%20Object%5D&originHeight=125&originWidth=878&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u578e5c49-ea62-41c0-bea8-019a9b2921d&title=)
### 参考链接

- 1：https ://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/
- 2：https ://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/
- 3：https ://kubernetes.io/docs/reference/scheduling/config/#profiles
- 4：https ://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/#extension-points
- 5：https ://kubernetes.io/docs/reference/scheduling/config/#multiple-profiles
- 6：https ://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/624-scheduling-framework/README.md
- 7：https ://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/624-scheduling-framework/README.md#custom-scheduler-plugins-out-of-tree
- 8：https ://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/624-scheduling-framework/README.md#optional-args
- 9：https ://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/624-scheduling-framework/README.md#configuring-plugins
### 更多文档
请关注微信公众号 **云原生CTO[1]**
### 参考资料
[1]
参考地址: _https://medium.com/codex/how-to-diagnose-oomkilled-error-in-kubernetes-application-201a9966be0d_
