---
title: "为 crd controller 添加 apiserver"
date: 2022-03-15T16:12:51+08:00
tags: ["kubernetes", "client-go"]
categories: ["kubernetes"]
author: "gaffey"
---

## crd operator
众所周之， kubernetes 除了 pod，deployment， service 这类内置资源之外，还能额外为业务自定义资源，也就构成了 ***云原生（Cloud Native）*** 这么一个生态，基于 k8s 为底座，开发各类云原生应用

### operator 工作原理
用户创建自定义资源的格式，例如
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.7.0
  creationTimestamp: null
  name: pipelines.devops.devops.io
spec:
  group: devops.devops.io
  names:
    kind: Pipeline
    listKind: PipelineList
    plural: pipelines
    singular: pipeline
  scope: Namespaced
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        description: Pipeline is the Schema for the pipelines API
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation
              of an object. Servers should convert recognized schemas to the latest
              internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this
              object represents. Servers may infer this from the endpoint the client
              submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: PipelineSpec defines the desired state of Pipeline
            properties:
              foo:
                description: Foo is an example field of Pipeline. Edit pipeline_types.go
                  to remove/update
                type: string
            type: object
          status:
            description: PipelineStatus defines the observed state of Pipeline
            type: object
        type: object
    served: true
    storage: true
    subresources:
      status: {}
```
可以看到， 我们利用 k8s `CustomResourceDefinition` 这个 kind 定义了我们自己的 group (devops), kind(pipeline), version(v1alpha1), 然后也展示了这个资源支持哪些字段：有通用的 apiversion，kind，metadata，spec，status 这些，这些也是必须的，我们能自定义的就是 spec 里面的内容，有一个 foo 字段，其中 type 是 string 类型。

我们有了 crd， 那么我们创建的 cr 也就是上面的 pipeline 这个资源，就会被 k8s 存入自己的存储（etcd），然后被 controller 组件监听到执行逻辑，但是k8s默认的控制器只会对它内置的资源进行操作，比如监听到一个pod的资源， 那么它就会读取 pod 的spec字段创建容器等，那么我们同时需要一个自定义的 controller 来处理我们的自定义资源。处理完成之后一般会更新 cr 的 status 字段来表示我已经处理完成了。

这仅仅是一个简单的描述，省略了调度，准入控制等流程。因为需要我们开发的内容就是crd的定义以及controller的编写。

![](https://ypy.7bao.fun/img/202203151632270.png)

因为整个流程都是监听 k8s master api 来完成的，所以让你的应用有自愈的特性成为了可能，而且官方出版的 `client-go`  这个包拥有完善的限流，队列，事件监听等组件，让你不用担心对 k8s master api 造成压力。

当然社区里面普遍推荐用 kubebuilder 来生成项目一整套开发到部署的脚手架，具体可以看教程尝试写一个。

crd + crd 的 controller 就是某个应用的 operatore 了。

## 为什么还需要 apiserver
我们基于上面的开发流程，很快就发现，我们的应用是面向用户的，而不是面向命令行的，我们可以 `kubectl apply -f cr.yaml` 很快的在 k8s 上运行我们的应用，并且应用除了执行我们的业务逻辑，还有具有自愈，备份（看你需求）等特性，但是我们的用户太局限了（仅开发者和集群运维人员），所以你的应用大部分情况下还是得暴露 REST API 接口的形式为前端（或其他系统）对接。

## 解决方案
1. 既然开发可以执行 `kubectl apply -f cr.yaml` 部署应用，那么我写一个 httpServer 代替执行 `kubectl apply xxx` 命令即可，例如你可以直接接收到前端传来的参数调用 `cmd exec` 执行命，或者更有经验可以想用 client-go 这个 sdk 的 dynamic client 来封装 yaml 发送的 master api， 这些都是可行的，但是各有各的问题。

2. 聚合API： 官方明确有表示，k8s master apiserver 是可以扩展的，可以参考 https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation, 这也是当前一部分云原生应用的做法，坑的是，官方给的 demo 并没有太多参考性，好在社区也有一个类似 kubebuilder 的脚手架工具 [apiserver-builder](https://github.com/kubernetes-sigs/apiserver-builder-alpha),  但是这个项目仍然是一个 alpha 阶段的项目，而且 README 明确说明了
   > Unless you absolutely need apiserver-aggregation, you are recommended to use Kubebuilder instead of apiserver-builder for building Kubernetes APIs  
   除非绝对需要apiserver-aggregation，否则建议使用Kubebuilder而不是apiserver-builder来构建Kubernetes api

   这多少让人对于这个解决方案充满了担忧。

## 参考社区 （kubesphere）
基于这类种种原因，我就去参阅社区各类大量使用 crd 的例子， 其中 paas 平台就是一个既需要 crd 控制器也需要暴露 REST 接口的例子，其中我参考的 [kubesphere 平台](https://github.com/kubesphere/kubesphere), 当然，由于这个平台综合性太强，我找到它的关于 [devops 的仓库](https://github.com/kubesphere/ks-devops)，比较精简，只包含 devops 相关的功能, 更好的了解实现。

### 目录
![](https://ypy.7bao.fun/img/202203151654953.png)

看目录，并不是 kubebuilder 直接生成的样子，但是也有 hack、controllers 这类目录，我感觉是结合了 [golang-standards project-layout](https://github.com/golang-standards/project-layout) 进行了改造，这样也很清晰，那么看 cmd 目录，可以发现，它确实是有一个 controller 和一个 apiserver  
![](https://ypy.7bao.fun/img/Snipaste_2022-03-15_17-00-45.png)

对于 controller 就不看了，重点看看如何为 crd 创建 apiserver  
### apiserver 创建
```go
func NewAPIServerCommand() (cmd *cobra.Command) {
	s := options.NewServerRunOptions()

	// Load configuration from file
	conf, err := config.TryLoadFromDisk()
	if err == nil {
		s = &options.ServerRunOptions{
			GenericServerRunOptions: s.GenericServerRunOptions,
			Config:                  conf,
		}
	} else {
		klog.Fatal("Failed to load configuration from disk", err)
	}

	cmd = &cobra.Command{
		Use: "apiserver",
		Long: `The KubeSphere API server validates and configures data for the API objects. 
The API Server services REST operations and provides the frontend to the
cluster's shared state through which all other components interact.`,
		RunE: func(cmd *cobra.Command, args []string) error {
			if errs := s.Validate(); len(errs) != 0 {
				return utilerrors.NewAggregate(errs)
			}

			return Run(s, signals.SetupSignalHandler())
		},
		SilenceUsage: true,
	}

	fs := cmd.Flags()
	namedFlagSets := s.Flags()
	for _, f := range namedFlagSets.FlagSets {
		fs.AddFlagSet(f)
	}

	usageFmt := "Usage:\n  %s\n"
	cmd.SetHelpFunc(func(cmd *cobra.Command, args []string) {
		fmt.Fprintf(cmd.OutOrStdout(), "%s\n\n"+usageFmt, cmd.Long, cmd.UseLine())
		cliflag.PrintSections(cmd.OutOrStdout(), namedFlagSets, 0)
	})

	versionCmd := &cobra.Command{
		Use:   "version",
		Short: "Print the version of KubeSphere DevOps apiserver",
		Run: func(cmd *cobra.Command, args []string) {
			// TODO implement the version output
		},
	}

	cmd.AddCommand(versionCmd)
	return
}
```
用 cobra 库管理应用的生命周期，没什么问题, 但是没看见这么起的 http server 提供 REST API， 来看看注册到 cobra 的 RunE 函数： 里面有调用 Run 方法  

```go
func Run(s *options.ServerRunOptions, stopCh <-chan struct{}) error {
	apiserver, err := s.NewAPIServer(stopCh)
	if err != nil {
		return err
	}

	err = apiserver.PrepareRun(stopCh)
	if err != nil {
		return nil
	}

	return apiserver.Run(stopCh)
}
```
很清晰，先 new 一个 apiserver `s.NewAPIServer(stopCh)`， 然后准备run之前做些什么事情 `apiserver.PrepareRun(stopCh)`, 最后真正 run 起来 `apiserver.Run(stopCh)`

#### new 一个 apiserver
apiserver 结构体
```go
type APIServer struct {

	...

	Server *http.Server

	// webservice container, where all webservice defines
	container *restful.Container

	// kubeClient is a collection of all kubernetes(include CRDs) objects clientset
	KubernetesClient k8s.Client

	// informerFactory is a collection of all kubernetes(include CRDs) objects informers,
	// mainly for fast query
	InformerFactory informers.InformerFactory

	// cache is used for short lived objects, like session
	CacheClient cache.Interface

	DevopsClient devops.Interface

}
```
省略了一点字段，关键我们看到了标准库的 http server，用来启动 http 服务， container 是用 restful 这个包来做 http server 的增强，中间件，路由等，这个包 k8s 的 master apiserver 也在用，然后 k8s 的一些组件 informer， cache 等，然后就是业务相关的 devops client。 很符合逻辑，接下来就是将这些参数填好就行。
> 这里还没注册路由，只是有一个空的 http server, 所以就引申出了下一步 PrepareRun

#### apiserver 预启动
```go
func (s *APIServer) PrepareRun(stopCh <-chan struct{}) error {
	s.container = restful.NewContainer()
	s.container.Filter(logRequestAndResponse)
	s.container.Router(restful.CurlyRouter{})
	// reference: https://pkg.go.dev/github.com/emicklei/go-restful#hdr-Performance_options
	s.container.DoNotRecover(false)
	s.container.RecoverHandler(func(panicReason interface{}, httpWriter http.ResponseWriter) {
		logStackOnRecover(panicReason, httpWriter)
	})

	s.installKubeSphereAPIs()

	for _, ws := range s.container.RegisteredWebServices() {
		klog.V(2).Infof("%s", ws.RootPath())
	}

	s.Server.Handler = s.container

	s.buildHandlerChain(stopCh)

	return nil
}
```
这里new了一个新的容器`s.container = restful.NewContainer()`（restful 包里的概念，不是 docker 容器），下面添加了请求过滤器和recover的中间件，这个我们不关心。重点是调用注册路由方法把 crd 以接口形式暴露 `s.installKubeSphereAPIs()`
```go
func (s *APIServer) installKubeSphereAPIs() {
	...

	var wss []*restful.WebService

	v1alpha2WSS, err := devopsv1alpha2.AddToContainer(s.container,
		s.InformerFactory.KubeSphereSharedInformerFactory(),
		s.DevopsClient,
		s.SonarClient,
		s.KubernetesClient.KubeSphere(),
		s.S3Client,
		s.Config.JenkinsOptions.Host,
		s.KubernetesClient,
		jenkinsCore)
	utilruntime.Must(err)
	wss = append(wss, v1alpha2WSS...)
	wss = append(wss, devopsv1alpha3.AddToContainer(s.container, s.DevopsClient, s.KubernetesClient, s.Client)...)
	wss = append(wss, oauth.AddToContainer(s.container,
		auth.NewTokenOperator(
			s.CacheClient,
			s.Config.AuthenticationOptions),
	))
	...
	doc.AddSwaggerService(wss, s.container)
}
```
前置说明：`var wss []*restful.WebService` 这个 WebService 数组就是我们路由关联 handler 的东西，有了 WebService 和 上面的 Container， 然后放到 httpserver 里面可以 启动一个 http rest api 风格的 api 服务器，所以这里 append 进 wss 的全部都是我们以后apiserver的路由。这里有 `devopsv1alpha2` 和 `devopsv1alpha3` 的 `AddToContainer`,里面都一样，就是开发的业务api版本不同， 我们看一个就行。

```go
var GroupVersion = schema.GroupVersion{Group: api.GroupName, Version: "v1alpha2"}

func AddToContainer(...) (wss []*restful.WebService, err error) {
	wsWithGroup := runtime.NewWebService(GroupVersion)
	wss = append(wss, wsWithGroup)
	// the API endpoint with group version will be removed in the future release
	if err = addToContainerWithWebService(container, ksInformers, devopsClient, sonarqubeClient, ksClient,
		s3Client, endpoint, k8sClient, jenkinsClient, wsWithGroup); err != nil {
		return
	}

	ws := runtime.NewWebServiceWithoutGroup(GroupVersion)
	wss = append(wss, ws)
	...
}

func addToContainerWithWebService(...) error {
	err := AddPipelineToWebService(ws, devopsClient, k8sClient)
	if err != nil {
		return err
	}
    
	...

	container.Add(ws)
	return nil
}
```
我把方法参数删了，太多了，没法看，主要是看, 关键来了，我们来到了crd定义自定义结构体的目录，也就是你的 crd 里面 spec 那里的目录，这里就跟我们 kububuilder 生成的东西关联起来了。（你可能没有 register.go 但是不影响，你有 groupversion_info.go）  
![](https://ypy.7bao.fun/img/202203151724564.png)
里面定义了如何将 GV 注册到 restful 包里的 WebService 里，这里注册方法很简单, 截取一小段
```go
ws.Route(ws.GET("/namespaces/{namespace}/pipelines/{pipeline}/pipelineruns").
		To(handler.listPipelineRuns).
		Doc("Get all runs of the specified pipeline").
		Param(ws.PathParameter("namespace", "Namespace of the pipeline")).
		Param(ws.PathParameter("pipeline", "Name of the pipeline")).
		Param(ws.QueryParameter("branch", "The name of SCM reference")).
		Param(ws.QueryParameter("backward", "Backward compatibility for v1alpha2 API "+
			"`/devops/{devops}/pipelines/{pipeline}/runs`. By default, the backward is true. If you want to list "+
			"full data of PipelineRuns, just set the parameters to false.").
			DataType("bool").
			DefaultValue("true")).
		Returns(http.StatusOK, api.StatusOK, v1alpha3.PipelineRunList{}))
```
当然， 这里仅仅是你业务路径 `/namespaces/{namespace}/pipelines/{pipeline}/pipelineruns` 前面没有你的 GV 信息, 一点都不 k8s，
我们漏了这一行的分析 `runtime.NewWebService(GroupVersion)`
```go
func NewWebService(gv schema.GroupVersion) *restful.WebService {
	webservice := restful.WebService{}
	webservice.Path(ApiRootPath + "/" + gv.String()).
		Produces(restful.MIME_JSON)

	return &webservice
}
```
你看， 在你的业务路由信息前面加上了 `ApiRootPath/gv.String()`, 所以整个路由看起来像这样 `http://{host.kubesphere}/kapis/devops.kubesphere.io/v1alpha3/namespaces/{workspace}/pipelines/dev-build/pipelineruns?page=1&limit=10&backward=false` 看起来顺眼多了，也像本身 k8s master api 的写法，预启动阶段走完，基本上可以直接运行 server 了

### api 服务如何包装 cr 清单存储到 k8s master 中
上面我们仅仅看到了将 cr 抽象成 rest api 路由，并没有看到拿到参数之后封装成 cr 发送到 k8s master 中, 但是我们找到定义路由的地方了，随便看一个路由关联的handler函数，看看是怎么做的, 比如创建凭证的方法 `CreatePipelineObj` 位于 `pkg/models/devops/devops.go`
```go
func (d devopsOperator) CreatePipelineObj(projectName string, pipeline *v1alpha3.Pipeline) (*v1alpha3.Pipeline, error) {
	projectObj, err := d.ksclient.DevopsV1alpha3().DevOpsProjects().Get(d.context, projectName, metav1.GetOptions{})
	if err != nil {
		return nil, err
	}
	projectObj.Annotations[devopsv1alpha3.PipelineSyncStatusAnnoKey] = StatusPending
	projectObj.Annotations[devopsv1alpha3.PipelineSyncTimeAnnoKey] = GetSyncNowTime()
	return d.ksclient.DevopsV1alpha3().Pipelines(projectObj.Status.AdminNamespace).Create(d.context, pipeline, metav1.CreateOptions{})
}
```
可以看到，没有我们想的那样， 用 client-go 的 dynamic client 包装 cr 的 yaml 然后发送给 k8s master 服务器。  
它这里用的客户端和 client-go 操作 clientset 操作内置资源一样简单和直观，它这里也抛弃使用 dynamic client， 毕竟难操作，是个 interface 就往里面仍。  

这里也是我们想要的功能，看看怎么做的, 点进去，代码很多，基本上是为每个 crd 结构体实现了一整套client-set  
![](https://ypy.7bao.fun/img/202203151806128.png)

但是，只用关心文件头一行注释
```
// xCode generated by client-gen. DO NOT EDIT.
```
！！！这些全都是生成的 ！！！  

直接下结论：  
用 client-gen 直接为你的自定义资源生成配套的 crud 客户端，美滋滋。有了这个，还要啥 dynamic client 啊。 至于这个 client-gen 怎么用，就不探究了。只需要知道， crd + 自定义contrller + apiserver（你用 gin， resuful 库无所谓）+ client-gen（生成配套的客户端） 这些都有了，就闭环了

### client-gen 生成 crd client-set 代码
自己改改参数就能用, 可以去仓库找 https://github.com/kubesphere/ks-devops/tree/master/hack
generate_client.sh
```sh
#!/bin/bash

set -e

GV="$1"

./hack/generate_group.sh all kubesphere.io/devops/pkg/client kubesphere.io/devops/api "${GV}" --output-base=./  -h "$PWD/hack/boilerplate.go.txt"
```
generate_group.sh
```sh
#!/usr/bin/env bash

# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

set -o errexit
set -o nounset
set -o pipefail

GOPATH=`go env GOPATH`
# generate-groups generates everything for a project with external types only, e.g. a project based
# on CustomResourceDefinitions.

if [ "$#" -lt 4 ] || [ "${1}" == "--help" ]; then
  cat <<EOF
Usage: $(basename "$0") <generators> <output-package> <apis-package> <groups-versions> ...

  <generators>        the generators comma separated to run (deepcopy,defaulter,client,lister,informer) or "all".
  <output-package>    the output package name (e.g. github.com/example/project/pkg/generated).
  <apis-package>      the external types dir (e.g. github.com/example/api or github.com/example/project/pkg/apis).
  <groups-versions>   the groups and their versions in the format "groupA:v1,v2 groupB:v1 groupC:v2", relative
                      to <api-package>.
  ...                 arbitrary flags passed to all generator binaries.


Examples:
  $(basename "$0") all             github.com/example/project/pkg/client github.com/example/project/pkg/apis "foo:v1 bar:v1alpha1,v1beta1"
  $(basename "$0") deepcopy,client github.com/example/project/pkg/client github.com/example/project/pkg/apis "foo:v1 bar:v1alpha1,v1beta1"
EOF
  exit 0
fi

GENS="$1"
OUTPUT_PKG="$2"
APIS_PKG="$3"
GROUPS_WITH_VERSIONS="$4"
shift 4


GO111MODULE=on go install k8s.io/code-generator/cmd/{client-gen,lister-gen,informer-gen,deepcopy-gen}

function codegen::join() { local IFS="$1"; shift; echo "$*"; }

# enumerate group versions
FQ_APIS=() # e.g. k8s.io/api/apps/v1
for GVs in ${GROUPS_WITH_VERSIONS}; do
  IFS=: read -r G Vs <<<"${GVs}"

  # enumerate versions
  for V in ${Vs//,/ }; do
    FQ_APIS+=("${APIS_PKG}/${G}/${V}")
  done
done

if [ "${GENS}" = "all" ] || grep -qw "deepcopy" <<<"${GENS}"; then
  echo "Generating deepcopy funcs"
  ${GOPATH}/bin/deepcopy-gen --input-dirs $(codegen::join , "${FQ_APIS[@]}") -O zz_generated.deepcopy --bounding-dirs ${APIS_PKG} "$@"
fi

if [ "${GENS}" = "all" ] || grep -qw "client" <<<"${GENS}"; then
  echo "Generating clientset for ${GROUPS_WITH_VERSIONS} at ${OUTPUT_PKG}/${CLIENTSET_PKG_NAME:-clientset}"
  ${GOPATH}/bin/client-gen --clientset-name ${CLIENTSET_NAME_VERSIONED:-versioned} --input-base "" --input $(codegen::join , "${FQ_APIS[@]}") --output-package ${OUTPUT_PKG}/${CLIENTSET_PKG_NAME:-clientset} "$@"
fi

if [ "${GENS}" = "all" ] || grep -qw "lister" <<<"${GENS}"; then
  echo "Generating listers for ${GROUPS_WITH_VERSIONS} at ${OUTPUT_PKG}/listers"
  ${GOPATH}/bin/lister-gen --input-dirs $(codegen::join , "${FQ_APIS[@]}") --output-package ${OUTPUT_PKG}/listers "$@"
fi

if [ "${GENS}" = "all" ] || grep -qw "informer" <<<"${GENS}"; then
  echo "Generating informers for ${GROUPS_WITH_VERSIONS} at ${OUTPUT_PKG}/informers"
  ${GOPATH}/bin/informer-gen \
           --input-dirs $(codegen::join , "${FQ_APIS[@]}") \
           --versioned-clientset-package ${OUTPUT_PKG}/${CLIENTSET_PKG_NAME:-clientset}/${CLIENTSET_NAME_VERSIONED:-versioned} \
           --listers-package ${OUTPUT_PKG}/listers \
           --output-package ${OUTPUT_PKG}/informers \
           "$@"
fi
```