# Overview

这篇文章是基于Kubernetes的v1.14.10 分支写下的源码分析文档。

我们可以使用`go get k8s.io/client-go@kubernetes-1.14.10` 来安装client-go

主要内容是理解并使用client-go四种客户端，为什么需要四种客户端，场景分别是什么，如何初始化四种客户端，并使用四个客户端分别去获取资源。

# 客户端

Client-go提供了四种客户端，简单描述如下

| 客户端名称      | 源码目录              | 简单描述                                                     |
| --------------- | --------------------- | ------------------------------------------------------------ |
| RESTClient      | client-go/rest/       | 基础客户端，对HTTP Request封装                               |
| ClientSet       | client-go/kubernetes/ | 在RESTClient基础上封装了对Resource和Version，也就是说我们使用ClientSet的话是必须要知道Resource和Version， 例如AppsV1().Deployments或者CoreV1.Pods，缺点是不能访问CRD自定义资源 |
| DynamicClient   | client-go/dynamic/    | 包含一组动态的客户端，可以对任意的K8S API对象执行通用操作，包括CRD自定义资源 |
| DiscoveryClient | client-go/discovery/  | 在上述我们试过ClientSet是必须要知道Resource和Version, 但人是记不住的(例如我)，这个DiscoveryClient是提供一个发现客户端，发现API Server支持的资源组，资源版本和资源信息 |



# 练习

## 使用RESTClient去Get Node列表

源码`staging/src/k8s.io/client-go/rest/config.go` 当我们使用`RESTClientFor`的时候要注意把GroupVersion/ NegotiatedSerializer都要初始化

```go
func RESTClientFor(config *Config) (*RESTClient, error) {
	if config.GroupVersion == nil {
		return nil, fmt.Errorf("GroupVersion is required when initializing a RESTClient")
	}
	if config.NegotiatedSerializer == nil {
		return nil, fmt.Errorf("NegotiatedSerializer is required when initializing a RESTClient")
	}
```

练习代码

```go
package main

import (
	"fmt"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/kubernetes/scheme"
)

func main()  {
	config, err := clientcmd.BuildConfigFromFlags("","/root/.kube/config")
	if err != nil {
		panic(err.Error())
	}

	config.APIPath = "api"
	config.GroupVersion = &corev1.SchemeGroupVersion
	config.NegotiatedSerializer = scheme.Codecs

	restClient, err := rest.RESTClientFor(config)
	if err != nil {
		panic(err.Error())
	}


	result := &corev1.NodeList{}
	err = restClient.Get().Namespace("").Resource("nodes").VersionedParams(&metav1.ListOptions{Limit: 100}, scheme.ParameterCodec).Do().Into(result)
	if err != nil {
		panic(err)
	}

	for _, d := range result.Items {
		fmt.Printf("Node Name %v \n", d.Name)
	}
}

```





## 使用ClientSet监听有namespace就注入Secret

```go
package main

import (
	"encoding/base64"
	"encoding/json"
	apiv1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/cache"
	"log"
	// apiv1 "k8s.io/api/core/v1"
	"time"
)

const create_secret_name = "xx-secret"

func  generateDockerConfig() []byte {
	type dockerConfig struct {
		Auths map[string]map[string]string `json:"auths"`
	}
	username, password := "xx", "xx"
	auth := map[string]string{
		"username": "username",
		"password": "password",
		"auth": base64.StdEncoding.EncodeToString([]byte(username + ":" + password)),
	}
	dockerconfig := dockerConfig{
		Auths: map[string]map[string]string{
			"docker-registry-url": auth,
		},
	}
	bytes, _ := json.Marshal(dockerconfig)
	return bytes
}

func create_secret(namespace string,clientset *kubernetes.Clientset) error  {
	secretClient := clientset.CoreV1().Secrets(namespace)

	// check secret exist in the namespace or not
	_, err := secretClient.Get(create_secret_name, v1.GetOptions{})

	if errors.IsNotFound(err) {
		// if not exist, then create secret
		log.Printf("Secret %s in namespace %s not found\n", create_secret_name, namespace)
		log.Println("Start to create secret..")

		secretObj := &apiv1.Secret{

			TypeMeta: v1.TypeMeta{
				Kind:       "Secret",
				APIVersion: "apps/v1",
			},
			ObjectMeta: v1.ObjectMeta{
				Name: create_secret_name,
			},
			Data: map[string][]byte{
				".dockerconfigjson": generateDockerConfig(),
			},
			Type: apiv1.SecretTypeDockerConfigJson,
		}

		_, err := secretClient.Create(secretObj)
		if err != nil{
			return err
		} else {
			log.Println("create secret success")
			return nil
		}
	} else if statusError, isStatus := err.(*errors.StatusError); isStatus {
		log.Printf("Error getting Secret %s in namespace %s: %v\n",
		create_secret_name, namespace, statusError.ErrStatus.Message)
		return statusError
	} else if err != nil {
		return(err)
	} else {
		// if exist, then return
		log.Printf("Found secret in namespace %s\n",  namespace)
		return nil
	}		

}

func main() {
	// receive env 

	// in cluster get config
	config, err := rest.InClusterConfig()
	if err != nil {
		panic(err.Error())
	}

	// cilentset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	// listen namespace informer for AddFunc
	stopCh := make(chan struct{})
	defer close(stopCh)

	shareInformers := informers.NewSharedInformerFactory(clientset, time.Second)
	informer := shareInformers.Core().V1().Namespaces().Informer()

	informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func (obj interface{})  {
			nObj := obj.(v1.Object)
			log.Printf("New namespaces add %s", nObj.GetName())

			// create secret in the ns
			if nObj.GetName() == "kube-node-lease" {
				log.Println("skip this namespace")
			} else if nObj.GetName() == "kube-public" {
				log.Println("skip this namespace")
			} else if nObj.GetName() == "kube-system" {
				log.Println("skip this namespace")
			} else {
				err := create_secret(nObj.GetName(), clientset)
				if err != nil {
					log.Println("create secret fail, fail at %s", err)
				}
			}
		},
	})
	informer.Run(stopCh)
	
}

```





## DynamicClient

看`kubectl api-resources` 命令背后是否使用了DynamicClient， 代码块`pkg/kubectl/cmd/apiresources/apiresources.go`

```go
func (o *APIResourceOptions) RunAPIResources(cmd *cobra.Command, f cmdutil.Factory) error {
	w := printers.GetNewTabWriter(o.Out)
	defer w.Flush()

    // 这里可以看出来的确是使用了discoveryCilent
	discoveryclient, err := f.ToDiscoveryClient()
	if err != nil {
		return err
	}

	if !o.Cached {
		// Always request fresh data from the server
		discoveryclient.Invalidate()
	}

	errs := []error{}
    
	lists, err := discoveryclient.ServerPreferredResources()	
...
}
```



## 使用DiscoveryClient去发现集群现所有的GVR

```go
package main

import (
	"fmt"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/discovery"
	"k8s.io/client-go/tools/clientcmd"
)

func main()  {
	config, err := clientcmd.BuildConfigFromFlags("","/root/.kube/config")
	if err != nil {
		panic(err.Error())
	}

	discoveryClient, err := discovery.NewDiscoveryClientForConfig(config)
	if err != nil {
		panic(err.Error())
	}

	_, APIResourceList, err := discoveryClient.ServerGroupsAndResources()
	if err != nil {
		panic(err.Error())
	}
	for _, list := range APIResourceList {
		gv, err := schema.ParseGroupVersion(list.GroupVersion)
		if err != nil {
			panic(err.Error())
		}
		for _, resource := range list.APIResources {
			fmt.Printf("name: %v, group: %v, version %v\n", resource.Name, gv.Group, gv.Version)
		}
	}
}

```







