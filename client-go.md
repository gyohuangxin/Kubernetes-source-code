# client-go 源码学习

## client-go介绍

Kubernetes系统使用client-go作为Go语言的官方编程式交互客户端，支持4种客户端对象与Kubernetes API Server交互的方式，分别是RESTClinet、ClientSet、DynamicClient以及DiscoveryClient。RESTClient对HTTP Request进行了封装，实现了RESTful风格的API，ClientSet、DynamicClient和DiscoveryClient则是在RESTClient的基础上实现的。
以上4种客户端都可以通过kubeconfig连接到指定的Kubernetes API Server。

## kubeconfig

client-go会读取kubeconfig配置并生成config对象，用于与kube-apiserver通信，示例代码如下：

```
package main

import (
  "k8s.io/client-go/tools/clientcmd"
)

func main() {
  config, err := clientcmd.BuildConfigFromFlags("",
  "root/.kube/config")
  if err != nil {
    panic(err)
  }
}

```

## RESTClient 客户端
RESTClient是最基本的客户端，具有很高的灵活性，数据不依赖与方法和资源，因此RESTClient能够处理多种类型的调用，返回不同的数据格式。
类似于kubectl命令，通过RESTClient列出所有运行的Pod资源对象，RESTClient Example代码示例如下：

```
package main

import (
  "fmt"
  
  corev1 "k8s.io/api/core/v1"
  metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
  "k8s.io/client-go/kubernetes/scheme"
  "k8s.io/client-go/rest"
  "k8s.io/client-go/tools/clientcmd"
)

func main() {
  config, err := clientcmd.BuildConfigFromFlags("",
  "root/.kube/config")
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

  result := &corev1.PodList()

  err = restClient.Get().
    Namespace("default").
	Resource("pods").
	VersionedParams(&metav1.ListOptions{Limit:500},
	scheme.ParameterCodec).
	Do().
	Into(result)
	if err != nil {
	  panic(err)
	}

  for _, d := range result.Items {
    fmt.Printf("NAMESPACE: %v \t NAME: %v \t STATU: %+v\n", d.Namespace,
	d.Name, d.Status.Phase)
  }
}

```
