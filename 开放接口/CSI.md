# Overview

容器存储接口CSI是独立于Kubernetes的，作为一个独立project，在Kubernetes v1.9版本的时候引入。CSI为Kubernetes提供了一个插件系统，像云厂商的AWS EBS, Azure File这些都是作为CSI的一个插件被编进去Kubernetes中。

CSI是作为一个标准开发的，用于将任意块和文件存储存储系统暴露给容器编排系统(如Kubernetes)上的容器化工作负载。通过采用容器存储接口，Kubernetes卷层变得真正可扩展。使用CSI，第三方存储提供商可以编写和部署Kubernetes中公开新存储系统的插件，而无需接触Kubernetes的核心代码。这为Kubernetes用户提供了更多的存储选项，并使系统更加安全和可靠。

# 开放接口

代码库：https://github.com/container-storage-interface/spec.git

代码位置： https://github.com/container-storage-interface/spec/blob/master/csi.proto

跟CRI类似，CSI也是定义了gRPC接口，云厂商的存储插件需要实现以下接口。

## Identity接口

该接口主要是用来获取插件信息，查询插件状态

```protobuf
service Identity {
  // 返回插件信息
  rpc GetPluginInfo(GetPluginInfoRequest)
    returns (GetPluginInfoResponse) {}
  // 返回插件提供的能力
  rpc GetPluginCapabilities(GetPluginCapabilitiesRequest)
    returns (GetPluginCapabilitiesResponse) {}
  // 健康探针
  rpc Probe (ProbeRequest)
    returns (ProbeResponse) {}
}
```



## Controller接口

从存储服务端来进行操作卷，包括创建卷，删除卷等

```protobuf
service Controller {
  // 创建卷
  rpc CreateVolume (CreateVolumeRequest)
    returns (CreateVolumeResponse) {}
  // 删除卷
  rpc DeleteVolume (DeleteVolumeRequest)
    returns (DeleteVolumeResponse) {}
  // attach 卷
  rpc ControllerPublishVolume (ControllerPublishVolumeRequest)
    returns (ControllerPublishVolumeResponse) {}
  // deattach 卷
  rpc ControllerUnpublishVolume (ControllerUnpublishVolumeRequest)
    returns (ControllerUnpublishVolumeResponse) {}
 ...
}
```

## Node接口

对Node节点上的卷进行操作

```protobuf
service Node {
 // 格式化卷并且挂载到一个临时目录
  rpc NodeStageVolume (NodeStageVolumeRequest)
    returns (NodeStageVolumeResponse) {}
// 把卷从临时目录卸载
  rpc NodeUnstageVolume (NodeUnstageVolumeRequest)
    returns (NodeUnstageVolumeResponse) {}
// 把卷从临时目录挂载到目标目录
  rpc NodePublishVolume (NodePublishVolumeRequest)
    returns (NodePublishVolumeResponse) {}
// 把卷从目标目录卸载
  rpc NodeUnpublishVolume (NodeUnpublishVolumeRequest)
    returns (NodeUnpublishVolumeResponse) {}
...
}

```



# 背景知识

这篇文章由于很多地方涉及到卷的操作：Attach, Detach, Mount和Unmount，一般来说，卷都需要Attach--> Mount-->Unmount--> Detach这四个操作，只有EmptyDir跟HostPath这两种才不需要Attach和Detach操作。

## 卷的生命周期

当用户创建卷的时候，首先调用CreateVolume接口，去创建卷，然后调用ControllerPublishVolume将卷attch到主机上，接下来会调用NodeStageVolume来进行格式化最后调用NodeStageVolume来mount到指定目录下。

CreateVolume-->ControllerPublishVolume-->NodeStageVolume-->NodePublishVolume

# Sidecar容器

Kubernetes CSI Sidecar容器是一组标准容器，旨在简化在Kubernetes上CSI驱动程序的开发和部署。

这些容器包含公共逻辑，用于监视Kubernetes API，触发针对“CSI卷驱动程序”容器的适当操作，并适当地更新Kubernetes API。

这些容器打算与第三方CSI驱动程序容器捆绑在一起，并作为pod部署在一起。

一般来说，CSI驱动程序应该和以下sidecar (helper)容器一起部署在Kubernetes上:

## external-attacher

监视由controller-manager创建的VolumeAttachment对象，并将卷附加/卸载到/从节点上(即调用ControllerPublish/ controllererunpublish)。

(ControllerPublish是让一个node节点不需要运行任何代码就可以attach卷)

(Detach就是反向操作，从一个node节点中detach卷)

代码库： https://github.com/kubernetes-csi/external-attacher.git

```go
func NewCSIAttachController(client kubernetes.Interface, attacherName string, handler Handler, volumeAttachmentInformer storageinformers.VolumeAttachmentInformer, pvInformer coreinformers.PersistentVolumeInformer, vaRateLimiter, paRateLimiter workqueue.RateLimiter, shouldReconcileVolumeAttachment bool, reconcileSync time.Duration) *CSIAttachController {
	...
	ctrl := &CSIAttachController{
		client:                          client,
		attacherName:                    attacherName,
		handler:                         handler,
		eventRecorder:                   eventRecorder,
		vaQueue:                         workqueue.NewNamedRateLimitingQueue(vaRateLimiter, "csi-attacher-va"),
		pvQueue:                         workqueue.NewNamedRateLimitingQueue(paRateLimiter, "csi-attacher-pv"),
		shouldReconcileVolumeAttachment: shouldReconcileVolumeAttachment,
		reconcileSync:                   reconcileSync,
	}
    // 监听volumeAttachment的增删改，触发写入队列
	volumeAttachmentInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    ctrl.vaAdded,
		UpdateFunc: ctrl.vaUpdated,
		DeleteFunc: ctrl.vaDeleted,
	})
	...
	pvInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    ctrl.pvAdded,
		UpdateFunc: ctrl.pvUpdated,
		//DeleteFunc: ctrl.pvDeleted, TODO: do we need this?
	})
...
	return ctrl
}

```

然后执行ControllerPublishVolume

```go
func (a *attacher) Attach(ctx context.Context, volumeID string, readOnly bool, nodeID string, caps *csi.VolumeCapability, context, secrets map[string]string) (metadata map[string]string, detached bool, err error) {
	client := csi.NewControllerClient(a.conn)
	req := csi.ControllerPublishVolumeRequest{
		VolumeId:         volumeID,
		NodeId:           nodeID,
		VolumeCapability: caps,
		Readonly:         readOnly,
		VolumeContext:    context,
		Secrets:          secrets,
	}    
	rsp, err := client.ControllerPublishVolume(ctx, &req)
	...
	return rsp.PublishContext, false, nil
}

```



## external-provisioner

监视Kubernetes PersistentVolumeClaim对象，并针对CSI端点触CreateVolume和DeleteVolume操作。

例如当我们创建一个`PersistentVolumeClaim`对象的时候，如果PVC指定了一个特定的`StorageClass`，并且存储类的provisioner字段中的名称与GetPluginInfo调用中指定的CSI端点返回的名称相匹配，则通过创建新的Kubernetes PersistentVolumeClaim对象触发卷供应。

成功地配给新卷之后，sidecar容器创建一个Kubernetes PersistentVolume对象来表示卷。

使用删除回收策略删除绑定到与此驱动程序对应的`PersistentVolumeClaim`对象，将导致sidecar容器对指定的CSI端点触发DeleteVolume操作以删除卷。成功删除卷之后，sidecar容器还删除表示卷的PersistentVolume对象。

代码库：https://github.com/kubernetes-csi/external-provisioner.git





## node-driver-registrar

使用kubelet设备插件机制向kubelet注册CSI驱动程序。



## cluster-driver-registrar

通过创建CSIDriver对象向Kubernetes集群注册CSI驱动程序，该对象使驱动程序能够自定义Kubernetes与它的交互方式。

## external-snapshotter

监视Kubernetes VolumeSnapshot CRD对象，并针对CSI端点触发CreateSnapshot和DeleteSnapshot操作。

## livenessprobe

可能包含在CSI插件pod中，以启用Kubernetes的活性探测机制。

# 部署CSI 驱动

# 如何测试驱动

# 如何编写一个本地存储CSI 





















