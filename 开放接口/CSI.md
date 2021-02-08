# Overview

容器存储接口CSI是独立于Kubernetes的，作为一个独立project，在Kubernetes v1.9版本的时候引入。CSI为Kubernetes提供了一个插件系统，像云厂商的AWS EBS, Azure File这些都是作为CSI的一个插件被编进去Kubernetes中。

CSI是作为一个标准开发的，用于将任意块和文件存储存储系统暴露给容器编排系统(如Kubernetes)上的容器化工作负载。通过采用容器存储接口，Kubernetes卷层变得真正可扩展。使用CSI，第三方存储提供商可以编写和部署Kubernetes中公开新存储系统的插件，而无需接触Kubernetes的核心代码。这为Kubernetes用户提供了更多的存储选项，并使系统更加安全和可靠。

# 开放接口

代码库：https://github.com/container-storage-interface/spec.git

代码位置： https://github.com/container-storage-interface/spec/blob/master/csi.proto

跟CRI类似，CSI也是定义了gRPC接口，云厂商的存储插件需要实现以下接口。

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



# Sidecar容器

一般来说，CSI驱动程序应该和以下sidecar (helper)容器一起部署在Kubernetes上:

## external-attacher

监视Kubernetes VolumeAttachment对象，并触发针对CSI端点ControllerPublish和ControllerPublish操作



## external-provisioner

监视Kubernetes PersistentVolumeClaim对象，并针对CSI端点触CreateVolume和DeleteVolume操作。

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





















