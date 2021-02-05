# Overview

这篇文章是基于Kubernetes的master  commitid: 写下的源码分析文档。

此篇文档主要是围绕endpoint controller的介绍以及工作原理。

代码位置 `pkg/controller/endpoint`

# 概念



# 实例化endpoint





```go
func (e *EndpointController) syncService(key string) error {
	startTime := time.Now()
	defer func() {
		klog.V(4).Infof("Finished syncing service %q endpoints. (%v)", key, time.Since(startTime))
	}()

	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
	service, err := e.serviceLister.Services(namespace).Get(name)
	if err != nil {
		if !errors.IsNotFound(err) {
			return err
		}

		// Delete the corresponding endpoint, as the service has been deleted.
		// TODO: Please note that this will delete an endpoint when a
		// service is deleted. However, if we're down at the time when
		// the service is deleted, we will miss that deletion, so this
		// doesn't completely solve the problem. See #6877.
		err = e.client.CoreV1().Endpoints(namespace).Delete(context.TODO(), name, metav1.DeleteOptions{})
		if err != nil && !errors.IsNotFound(err) {
			return err
		}
		e.triggerTimeTracker.DeleteService(namespace, name)
		return nil
	}

	if service.Spec.Selector == nil {
		// services without a selector receive no endpoints from this controller;
		// these services will receive the endpoints that are created out-of-band via the REST API.
		return nil
	}

	klog.V(5).Infof("About to update endpoints for service %q", key)
	pods, err := e.podLister.Pods(service.Namespace).List(labels.Set(service.Spec.Selector).AsSelectorPreValidated())
	if err != nil {
		// Since we're getting stuff from a local cache, it is
		// basically impossible to get this error.
		return err
	}

	// If the user specified the older (deprecated) annotation, we have to respect it.
	tolerateUnreadyEndpoints := service.Spec.PublishNotReadyAddresses
	if v, ok := service.Annotations[TolerateUnreadyEndpointsAnnotation]; ok {
		b, err := strconv.ParseBool(v)
		if err == nil {
			tolerateUnreadyEndpoints = b
		} else {
			utilruntime.HandleError(fmt.Errorf("Failed to parse annotation %v: %v", TolerateUnreadyEndpointsAnnotation, err))
		}
	}

	// We call ComputeEndpointLastChangeTriggerTime here to make sure that the
	// state of the trigger time tracker gets updated even if the sync turns out
	// to be no-op and we don't update the endpoints object.
	endpointsLastChangeTriggerTime := e.triggerTimeTracker.
		ComputeEndpointLastChangeTriggerTime(namespace, service, pods)

	subsets := []v1.EndpointSubset{}
	var totalReadyEps int
	var totalNotReadyEps int

	for _, pod := range pods {
		if len(pod.Status.PodIP) == 0 {
			klog.V(5).Infof("Failed to find an IP for pod %s/%s", pod.Namespace, pod.Name)
			continue
		}
		if !tolerateUnreadyEndpoints && pod.DeletionTimestamp != nil {
			klog.V(5).Infof("Pod is being deleted %s/%s", pod.Namespace, pod.Name)
			continue
		}

		ep, err := podToEndpointAddressForService(service, pod)
		if err != nil {
			// this will happen, if the cluster runs with some nodes configured as dual stack and some as not
			// such as the case of an upgrade..
			klog.V(2).Infof("failed to find endpoint for service:%v with ClusterIP:%v on pod:%v with error:%v", service.Name, service.Spec.ClusterIP, pod.Name, err)
			continue
		}

		epa := *ep
		if endpointutil.ShouldSetHostname(pod, service) {
			epa.Hostname = pod.Spec.Hostname
		}

		// Allow headless service not to have ports.
		if len(service.Spec.Ports) == 0 {
			if service.Spec.ClusterIP == api.ClusterIPNone {
				subsets, totalReadyEps, totalNotReadyEps = addEndpointSubset(subsets, pod, epa, nil, tolerateUnreadyEndpoints)
				// No need to repack subsets for headless service without ports.
			}
		} else {
			for i := range service.Spec.Ports {
				servicePort := &service.Spec.Ports[i]
				portNum, err := podutil.FindPort(pod, servicePort)
				if err != nil {
					klog.V(4).Infof("Failed to find port for service %s/%s: %v", service.Namespace, service.Name, err)
					continue
				}
				epp := endpointPortFromServicePort(servicePort, portNum)

				var readyEps, notReadyEps int
				subsets, readyEps, notReadyEps = addEndpointSubset(subsets, pod, epa, epp, tolerateUnreadyEndpoints)
				totalReadyEps = totalReadyEps + readyEps
				totalNotReadyEps = totalNotReadyEps + notReadyEps
			}
		}
	}
	subsets = endpoints.RepackSubsets(subsets)

	// See if there's actually an update here.
	currentEndpoints, err := e.endpointsLister.Endpoints(service.Namespace).Get(service.Name)
	if err != nil {
		if errors.IsNotFound(err) {
			currentEndpoints = &v1.Endpoints{
				ObjectMeta: metav1.ObjectMeta{
					Name:   service.Name,
					Labels: service.Labels,
				},
			}
		} else {
			return err
		}
	}

	createEndpoints := len(currentEndpoints.ResourceVersion) == 0

	if !createEndpoints &&
		apiequality.Semantic.DeepEqual(currentEndpoints.Subsets, subsets) &&
		apiequality.Semantic.DeepEqual(currentEndpoints.Labels, service.Labels) {
		klog.V(5).Infof("endpoints are equal for %s/%s, skipping update", service.Namespace, service.Name)
		return nil
	}
	newEndpoints := currentEndpoints.DeepCopy()
	newEndpoints.Subsets = subsets
	newEndpoints.Labels = service.Labels
	if newEndpoints.Annotations == nil {
		newEndpoints.Annotations = make(map[string]string)
	}

	if !endpointsLastChangeTriggerTime.IsZero() {
		newEndpoints.Annotations[v1.EndpointsLastChangeTriggerTime] =
			endpointsLastChangeTriggerTime.Format(time.RFC3339Nano)
	} else { // No new trigger time, clear the annotation.
		delete(newEndpoints.Annotations, v1.EndpointsLastChangeTriggerTime)
	}

	if newEndpoints.Labels == nil {
		newEndpoints.Labels = make(map[string]string)
	}

	if !helper.IsServiceIPSet(service) {
		newEndpoints.Labels = utillabels.CloneAndAddLabel(newEndpoints.Labels, v1.IsHeadlessService, "")
	} else {
		newEndpoints.Labels = utillabels.CloneAndRemoveLabel(newEndpoints.Labels, v1.IsHeadlessService)
	}

	klog.V(4).Infof("Update endpoints for %v/%v, ready: %d not ready: %d", service.Namespace, service.Name, totalReadyEps, totalNotReadyEps)
	if createEndpoints {
		// No previous endpoints, create them
		_, err = e.client.CoreV1().Endpoints(service.Namespace).Create(context.TODO(), newEndpoints, metav1.CreateOptions{})
	} else {
		// Pre-existing
		_, err = e.client.CoreV1().Endpoints(service.Namespace).Update(context.TODO(), newEndpoints, metav1.UpdateOptions{})
	}
	if err != nil {
		if createEndpoints && errors.IsForbidden(err) {
			// A request is forbidden primarily for two reasons:
			// 1. namespace is terminating, endpoint creation is not allowed by default.
			// 2. policy is misconfigured, in which case no service would function anywhere.
			// Given the frequency of 1, we log at a lower level.
			klog.V(5).Infof("Forbidden from creating endpoints: %v", err)

			// If the namespace is terminating, creates will continue to fail. Simply drop the item.
			if errors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
				return nil
			}
		}

		if createEndpoints {
			e.eventRecorder.Eventf(newEndpoints, v1.EventTypeWarning, "FailedToCreateEndpoint", "Failed to create endpoint for service %v/%v: %v", service.Namespace, service.Name, err)
		} else {
			e.eventRecorder.Eventf(newEndpoints, v1.EventTypeWarning, "FailedToUpdateEndpoint", "Failed to update endpoint %v/%v: %v", service.Namespace, service.Name, err)
		}

		return err
	}
	return nil
}

```

