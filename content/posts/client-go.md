---
title: "client-go 简单介绍和使用"
date: 2022-03-08T20:13:25+08:00
tags: ["kubernetes", "client-go"]
categories: ["kubernetes"]
author: "gaffey"
---
# client-go 库
client-go 是用 Golang 语言编写的官方编程式交互客户端库，提供对 Kubernetes API server 服务的交互访问。
其源码目录结构如下：
![](https://ypy.7bao.fun/img/202203082046212.png)

## RESTClient 客户端
RESTClient 是最基础的客户端，其他的 ClientSet、DynamicClient 以及 DiscoveryClient 都是基于 RESTClient 实现的  
![](https://ypy.7bao.fun/img/202203082047621.png)

```go
package main

import (
	"context"
	"fmt"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	"path/filepath"
)

func main() {
	home := homedir.HomeDir()
	config, err := clientcmd.BuildConfigFromFlags("", filepath.Join(home, ".kube", "config"))
	if err != nil {
		panic(err)
	}

	config.APIPath = "api"
	config.GroupVersion = &corev1.SchemeGroupVersion
	config.NegotiatedSerializer = scheme.Codecs

	restClient, err := rest.RESTClientFor(config)
	if err != nil {
		panic(err)
	}

	result := &corev1.PodList{}
	err = restClient.Get().Namespace(corev1.NamespaceAll).Resource("pods").Do(context.TODO()).Into(result)
	if err != nil {
		panic(err)
	}

	for _, d := range result.Items {
		fmt.Printf("namespace: %v\t name: %v\t statu:%v\n", d.Namespace, d.Name, d.Status.Phase)
	}
}
```
![](https://ypy.7bao.fun/img/20220308202824.png)  
1. 首先加载 kubeconfig 配置信息
2. 设置请求的HTTP路径 `config.APIPath = "api"`
3. 设置请求的资源组和版本 `config.GroupVersion = &corev1.SchemeGroupVersion`
4. 最后设置数据的编解码器 `config.NegotiatedSerializer = scheme.Codecs`

---
### ClientSet 客户端
ClientSet 客户端在 RESTClient 的基础上封装了对资源和版本的管理方法。每个资源可以理解为一个客户端，而 ClientSet 则是多个客户端的集合，每一个资源和版本都以函数的方式暴露给开发者。  
![](https://ypy.7bao.fun/img/202203082054642.png)
```go
package main

import (
	"context"
	"fmt"
	v1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	"path/filepath"
)

func main() {
	home := homedir.HomeDir()
	config, err := clientcmd.BuildConfigFromFlags("", filepath.Join(home, ".kube", "config"))
	if err != nil {
		panic(err)
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	podClient := clientset.CoreV1().Pods(v1.NamespaceAll)
	podList, err := podClient.List(context.TODO(), v1.ListOptions{Limit: 500})
	if err != nil {
		panic(err)
	}

	for _, d := range podList.Items {
		fmt.Printf("namespace: %v\t name: %v\t statu:%v\n", d.Namespace, d.Name, d.Status.Phase)
	}
}
```
![](https://ypy.7bao.fun/img/20220308202824.png)

### DynamicClient 客户端
DynamicClient 是一种动态的客户端，它可以对任意 Kubernetes 资源进行 RESTful 操作，主要是用来访问 CRD 自定义资源。  
它内部实现了 Unstructured，用于处理非结构化的数据结构(无法提前预知的数据结构)
这也是 DynamicClient 能够处理 CRD 自定义资源的关键。
```go
package main

import (
	"context"
	"fmt"
	corev1 "k8s.io/api/core/v1"
	v1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	"path/filepath"
)

func main() {
	home := homedir.HomeDir()
	config, err := clientcmd.BuildConfigFromFlags("", filepath.Join(home, ".kube", "config"))
	if err != nil {
		panic(err)
	}

	dynamicClient, err := dynamic.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	gvr := schema.GroupVersionResource{Version: "v1", Resource: "pods"}
	unstructObj, err := dynamicClient.Resource(gvr).Namespace(v1.NamespaceAll).List(context.TODO(), v1.ListOptions{Limit: 500})
	if err != nil {
		panic(err)
	}

	podList := &corev1.PodList{}
	err = runtime.DefaultUnstructuredConverter.FromUnstructured(unstructObj.UnstructuredContent(), podList)
	if err != nil {
		panic(err)
	}

	for _, d := range podList.Items {
		fmt.Printf("namespace: %v\t name: %v\t statu:%v\n", d.Namespace, d.Name, d.Status.Phase)
	}
}

```
![](https://ypy.7bao.fun/img/20220308202824.png)

### DiscoveryClient 客户端
DiscoveryClient 是一个发现客户端，它主要用于发现 K8S API Server 支持的资源组，资源版本和资源信息。所以开发者可以通过使用 DiscoveryClient 客户端查看所支持的资源组，资源版本和资源信息。  
它还可以将这些信息存储到本地。默认 `~/kube/cache` 来减轻 API Server 的压力
```go
package main

import (
	"fmt"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/discovery"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	"path/filepath"
)

func main() {
	home := homedir.HomeDir()
	config, err := clientcmd.BuildConfigFromFlags("", filepath.Join(home, ".kube", "config"))
	if err != nil {
		panic(err)
	}

	discoveryClient, err := discovery.NewDiscoveryClientForConfig(config)
	if err != nil {
		panic(err)
	}

	_, ApiResourceList, err := discoveryClient.ServerGroupsAndResources()
	if err != nil {
		panic(err)
	}

	for _, list := range ApiResourceList {
		gv, err := schema.ParseGroupVersion(list.GroupVersion)
		if err != nil {
			panic(err)
		}

		for _, resource := range list.APIResources {
			fmt.Printf("name: %v\t group: %v\t version: %v\n", resource.Name, gv.Group, gv.Version)
		}
	}
}

```
![](https://ypy.7bao.fun/img/202203082043356.png)