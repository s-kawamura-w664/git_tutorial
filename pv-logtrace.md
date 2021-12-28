# PVログ調査

```text
volume_manager.go:406] "Waiting for volumes to attach and mount for pod" pod="default/pod2"
desired_state_of_world_populator.go:489] "Found PVC" PVC="default/pvc1"
desired_state_of_world_populator.go:508] "Found bound PV for PVC" PVC="default/pvc1" PVCUID=38c829ea-060b-482a-84e7-39fdecc173da PVName="pvc-38c829ea-060b-482a-84e7-39fdecc173da"
desired_state_of_world_populator.go:519] "Extracted volumeSpec from bound PV and PVC" PVC="default/pvc1" PVCUID=38c829ea-060b-482a-84e7-39fdecc173da PVName="pvc-38c829ea-060b-482a-84e7-39fdecc173da" volumeSpecName="pvc-38c829ea-060b-482a-84e7-39fdecc173da"
desired_state_of_world_populator.go:316] "Added volume to desired state" pod="default/pod2" volumeName="myvolume" volumeSpecName="pvc-38c829ea-060b-482a-84e7-39fdecc173da"
desired_state_of_world_populator.go:316] "Added volume to desired state" pod="default/pod2" volumeName="kube-api-access-vq2z9" volumeSpecName="kube-api-access-vq2z9"
```

**pkg\kubelet\volumemanager\volume_manager.go**

```go
func (vm *volumeManager) WaitForAttachAndMount(pod *v1.Pod) error {
	if pod == nil {
		return nil
	}

	expectedVolumes := getExpectedVolumes(pod)
	if len(expectedVolumes) == 0 {
		// No volumes to verify
		return nil
	}

	klog.V(3).InfoS("Waiting for volumes to attach and mount for pod", "pod", klog.KObj(pod))       //★
	uniquePodName := util.GetUniquePodName(pod)

	// Some pods expect to have Setup called over and over again to update.
	// Remount plugins for which this is true. (Atomically updating volumes,
	// like Downward API, depend on this to update the contents of the volume).
	vm.desiredStateOfWorldPopulator.ReprocessPod(uniquePodName)

	err := wait.PollImmediate(
		podAttachAndMountRetryInterval,
		podAttachAndMountTimeout,
		vm.verifyVolumesMountedFunc(uniquePodName, expectedVolumes))

	if err != nil {
		unmountedVolumes :=
			vm.getUnmountedVolumes(uniquePodName, expectedVolumes)
		// Also get unattached volumes for error message
		unattachedVolumes :=
			vm.getUnattachedVolumes(expectedVolumes)

		if len(unmountedVolumes) == 0 {
			return nil
		}

		return fmt.Errorf(
			"unmounted volumes=%v, unattached volumes=%v: %s",
			unmountedVolumes,
			unattachedVolumes,
			err)
	}

	klog.V(3).InfoS("All volumes are attached and mounted for pod", "pod", klog.KObj(pod))
	return nil
}
```

**pkg\kubelet\volumemanager\populator\desired_state_of_world_populator.go**

desiredStateOfWorldPopulator.Run()
    desiredStateOfWorldPopulator.populatorLoop()
        desiredStateOfWorldPopulator.findAndAddNewPods()
            desiredStateOfWorldPopulator.processPodVolumes()
                desiredStateOfWorldPopulator.createVolumeSpec()

```go
func (dswp *desiredStateOfWorldPopulator) ReprocessPod(
	podName volumetypes.UniquePodName) {
	dswp.markPodProcessingFailed(podName)
}

// markPodProcessingFailed marks the specified pod from processedPods as false to indicate that it failed processing
func (dswp *desiredStateOfWorldPopulator) markPodProcessingFailed(
	podName volumetypes.UniquePodName) {
	dswp.pods.Lock()
	dswp.pods.processedPods[podName] = false
	dswp.pods.Unlock()
}
```

```
func (dswp *desiredStateOfWorldPopulator) Run(sourcesReady config.SourcesReady, stopCh <-chan struct{}) {
	// Wait for the completion of a loop that started after sources are all ready, then set hasAddedPods accordingly
	klog.InfoS("Desired state populator starts to run")
	wait.PollUntil(dswp.loopSleepDuration, func() (bool, error) {
		done := sourcesReady.AllReady()
		dswp.populatorLoop()												//■
		return done, nil
	}, stopCh)
	dswp.hasAddedPodsLock.Lock()
	dswp.hasAddedPods = true
	dswp.hasAddedPodsLock.Unlock()
	wait.Until(dswp.populatorLoop, dswp.loopSleepDuration, stopCh)			//■
}
```

```
func (dswp *desiredStateOfWorldPopulator) populatorLoop() {                 //たぶんGoルーチンから定期的に呼び出される
	dswp.findAndAddNewPods()												//■

	// findAndRemoveDeletedPods() calls out to the container runtime to
	// determine if the containers for a given pod are terminated. This is
	// an expensive operation, therefore we limit the rate that
	// findAndRemoveDeletedPods() is called independently of the main
	// populator loop.
	if time.Since(dswp.timeOfLastGetPodStatus) < dswp.getPodStatusRetryDuration {
		klog.V(5).InfoS("Skipping findAndRemoveDeletedPods(). ", "nextRetryTime", dswp.timeOfLastGetPodStatus.Add(dswp.getPodStatusRetryDuration), "retryDuration", dswp.getPodStatusRetryDuration)
		return
	}

	dswp.findAndRemoveDeletedPods()
}
```

```
// Iterate through all pods and add to desired state of world if they don't
// exist but should
func (dswp *desiredStateOfWorldPopulator) findAndAddNewPods() {
	// Map unique pod name to outer volume name to MountedVolume.
	mountedVolumesForPod := make(map[volumetypes.UniquePodName]map[string]cache.MountedVolume)
	if utilfeature.DefaultFeatureGate.Enabled(features.ExpandInUsePersistentVolumes) {
		for _, mountedVolume := range dswp.actualStateOfWorld.GetMountedVolumes() {
			mountedVolumes, exist := mountedVolumesForPod[mountedVolume.PodName]
			if !exist {
				mountedVolumes = make(map[string]cache.MountedVolume)
				mountedVolumesForPod[mountedVolume.PodName] = mountedVolumes
			}
			mountedVolumes[mountedVolume.OuterVolumeSpecName] = mountedVolume
		}
	}

	processedVolumesForFSResize := sets.NewString()
	for _, pod := range dswp.podManager.GetPods() {                                         //新しいPodだけなのか、すべてのPodなのかどちらでしょう？
		if dswp.podStateProvider.ShouldPodContainersBeTerminating(pod.UID) {
			// Do not (re)add volumes for pods that can't also be starting containers
			continue
		}
		dswp.processPodVolumes(pod, mountedVolumesForPod, processedVolumesForFSResize)		//■
	}
}
```

```
// processPodVolumes processes the volumes in the given pod and adds them to the
// desired state of the world.
func (dswp *desiredStateOfWorldPopulator) processPodVolumes(
	pod *v1.Pod,
	mountedVolumesForPod map[volumetypes.UniquePodName]map[string]cache.MountedVolume,
	processedVolumesForFSResize sets.String) {
	if pod == nil {
		return
	}

	uniquePodName := util.GetUniquePodName(pod)
	if dswp.podPreviouslyProcessed(uniquePodName) {
		return
	}

	allVolumesAdded := true
	mounts, devices := util.GetPodVolumeNames(pod)

	expandInUsePV := utilfeature.DefaultFeatureGate.Enabled(features.ExpandInUsePersistentVolumes)
	// Process volume spec for each volume defined in pod
	for _, podVolume := range pod.Spec.Volumes {                                                                //Podに定義されたVolumesを舐める
		if !mounts.Has(podVolume.Name) && !devices.Has(podVolume.Name) {
			// Volume is not used in the pod, ignore it.
			klog.V(4).InfoS("Skipping unused volume", "pod", klog.KObj(pod), "volumeName", podVolume.Name)
			continue
		}

		pvc, volumeSpec, volumeGidValue, err :=
			dswp.createVolumeSpec(podVolume, pod, mounts, devices)														//■
		if err != nil {
			klog.ErrorS(err, "Error processing volume", "pod", klog.KObj(pod), "volumeName", podVolume.Name)
			dswp.desiredStateOfWorld.AddErrorToPod(uniquePodName, err.Error())
			allVolumesAdded = false
			continue
		}

		// Add volume to desired state of world
		uniqueVolumeName, err := dswp.desiredStateOfWorld.AddPodToVolume(
			uniquePodName, pod, volumeSpec, podVolume.Name, volumeGidValue)
		if err != nil {
			klog.ErrorS(err, "Failed to add volume to desiredStateOfWorld", "pod", klog.KObj(pod), "volumeName", podVolume.Name, "volumeSpecName", volumeSpec.Name())
			dswp.desiredStateOfWorld.AddErrorToPod(uniquePodName, err.Error())
			allVolumesAdded = false
		} else {
			klog.V(4).InfoS("Added volume to desired state", "pod", klog.KObj(pod), "volumeName", podVolume.Name, "volumeSpecName", volumeSpec.Name())	//★
		}
		// sync reconstructed volume
		dswp.actualStateOfWorld.SyncReconstructedVolume(uniqueVolumeName, uniquePodName, podVolume.Name)

		if expandInUsePV {
			dswp.checkVolumeFSResize(pod, podVolume, pvc, volumeSpec,
				uniquePodName, mountedVolumesForPod, processedVolumesForFSResize)
		}
	}

	// some of the volume additions may have failed, should not mark this pod as fully processed
	if allVolumesAdded {
		dswp.markPodProcessed(uniquePodName)
		// New pod has been synced. Re-mount all volumes that need it
		// (e.g. DownwardAPI)
		dswp.actualStateOfWorld.MarkRemountRequired(uniquePodName)
		// Remove any stored errors for the pod, everything went well in this processPodVolumes
		dswp.desiredStateOfWorld.PopPodErrors(uniquePodName)
	} else if dswp.podHasBeenSeenOnce(uniquePodName) {
		// For the Pod which has been processed at least once, even though some volumes
		// may not have been reprocessed successfully this round, we still mark it as processed to avoid
		// processing it at a very high frequency. The pod will be reprocessed when volume manager calls
		// ReprocessPod() which is triggered by SyncPod.
		dswp.markPodProcessed(uniquePodName)
	}

}
```

```go
// createVolumeSpec creates and returns a mutable volume.Spec object for the
// specified volume. It dereference any PVC to get PV objects, if needed.
// Returns an error if unable to obtain the volume at this time.
func (dswp *desiredStateOfWorldPopulator) createVolumeSpec(
	podVolume v1.Volume, pod *v1.Pod, mounts, devices sets.String) (*v1.PersistentVolumeClaim, *volume.Spec, string, error) {
	pvcSource := podVolume.VolumeSource.PersistentVolumeClaim
	isEphemeral := pvcSource == nil && podVolume.VolumeSource.Ephemeral != nil
	if isEphemeral {
		// Generic ephemeral inline volumes are handled the
		// same way as a PVC reference. The only additional
		// constraint (checked below) is that the PVC must be
		// owned by the pod.
		pvcSource = &v1.PersistentVolumeClaimVolumeSource{
			ClaimName: ephemeral.VolumeClaimName(pod, &podVolume),
		}
	}
	if pvcSource != nil {                                                                           //PVCが指定されている場合
		klog.V(5).InfoS("Found PVC", "PVC", klog.KRef(pod.Namespace, pvcSource.ClaimName))			//★
		// If podVolume is a PVC, fetch the real PV behind the claim
		pvc, err := dswp.getPVCExtractPV(
			pod.Namespace, pvcSource.ClaimName)                                                     //Namespace+PVCNameからPVCの情報を取得
		if err != nil {
			return nil, nil, "", fmt.Errorf(
				"error processing PVC %s/%s: %v",
				pod.Namespace,
				pvcSource.ClaimName,
				err)
		}
		if isEphemeral {
			if err := ephemeral.VolumeIsForPod(pod, pvc); err != nil {
				return nil, nil, "", err
			}
		}
		pvName, pvcUID := pvc.Spec.VolumeName, pvc.UID                                              //PVC情報からPV名とPVCIDを取得
		klog.V(5).InfoS("Found bound PV for PVC", "PVC", klog.KRef(pod.Namespace, pvcSource.ClaimName), "PVCUID", pvcUID, "PVName", pvName)		//★
		// Fetch actual PV object
		volumeSpec, volumeGidValue, err :=
			dswp.getPVSpec(pvName, pvcSource.ReadOnly, pvcUID)                                      //PVの情報を取得
		if err != nil {
			return nil, nil, "", fmt.Errorf(
				"error processing PVC %s/%s: %v",
				pod.Namespace,
				pvcSource.ClaimName,
				err)
		}
		klog.V(5).InfoS("Extracted volumeSpec from bound PV and PVC", "PVC", klog.KRef(pod.Namespace, pvcSource.ClaimName), "PVCUID", pvcUID, "PVName", pvName, "volumeSpecName", volumeSpec.Name())				//★
		migratable, err := dswp.csiMigratedPluginManager.IsMigratable(volumeSpec)
		if err != nil {
			return nil, nil, "", err
		}
		if migratable {
			volumeSpec, err = csimigration.TranslateInTreeSpecToCSI(volumeSpec, pod.Namespace, dswp.intreeToCSITranslator)
			if err != nil {
				return nil, nil, "", err
			}
		}

		// TODO: replace this with util.GetVolumeMode() when features.BlockVolume is removed.
		// The function will return the right value then.
		volumeMode := v1.PersistentVolumeFilesystem
		if volumeSpec.PersistentVolume != nil && volumeSpec.PersistentVolume.Spec.VolumeMode != nil {
			volumeMode = *volumeSpec.PersistentVolume.Spec.VolumeMode
		}

		// TODO: remove features.BlockVolume checks / comments after no longer needed
		// Error if a container has volumeMounts but the volumeMode of PVC isn't Filesystem.
		// Do not check feature gate here to make sure even when the feature is disabled in kubelet,
		// because controller-manager / API server can already contain block PVs / PVCs.
		if mounts.Has(podVolume.Name) && volumeMode != v1.PersistentVolumeFilesystem {
			return nil, nil, "", fmt.Errorf(
				"volume %s has volumeMode %s, but is specified in volumeMounts",
				podVolume.Name,
				volumeMode)
		}
		// Error if a container has volumeDevices but the volumeMode of PVC isn't Block
		if devices.Has(podVolume.Name) && volumeMode != v1.PersistentVolumeBlock {
			return nil, nil, "", fmt.Errorf(
				"volume %s has volumeMode %s, but is specified in volumeDevices",
				podVolume.Name,
				volumeMode)
		}
		return pvc, volumeSpec, volumeGidValue, nil
	}

	// Do not return the original volume object, since the source could mutate it
	clonedPodVolume := podVolume.DeepCopy()

	spec := volume.NewSpecFromVolume(clonedPodVolume)
	migratable, err := dswp.csiMigratedPluginManager.IsMigratable(spec)
	if err != nil {
		return nil, nil, "", err
	}
	if migratable {
		spec, err = csimigration.TranslateInTreeSpecToCSI(spec, pod.Namespace, dswp.intreeToCSITranslator)
		if err != nil {
			return nil, nil, "", err
		}
	}
	return nil, spec, "", nil
}
```

**pkg\kubelet\volumemanager\reconciler\reconciler.go**

```text
reconciler.go:211] "Starting operationExecutor.VerifyControllerAttachedVolume for volume \"pvc-38c829ea-060b-482a-84e7-39fdecc173da\" (UniqueName: \"kubernetes.io/csi/nfs.csi.k8s.io^192.168.1.60/nfs-private/pvc-38c829ea-060b-482a-84e7-39fdecc173da\") pod \"pod2\" (UID: \"b9ba097d-5dff-4c9d-ab9e-8287801185d5\") "
reconciler.go:224] "operationExecutor.VerifyControllerAttachedVolume started for volume \"pvc-38c829ea-060b-482a-84e7-39fdecc173da\" (UniqueName: \"kubernetes.io/csi/nfs.csi.k8s.io^192.168.1.60/nfs-private/pvc-38c829ea-060b-482a-84e7-39fdecc173da\") pod \"pod2\" (UID: \"b9ba097d-5dff-4c9d-ab9e-8287801185d5\") "
reconciler.go:211] "Starting operationExecutor.VerifyControllerAttachedVolume for volume \"kube-api-access-vq2z9\" (UniqueName: \"kubernetes.io/projected/b9ba097d-5dff-4c9d-ab9e-8287801185d5-kube-api-access-vq2z9\") pod \"pod2\" (UID: \"b9ba097d-5dff-4c9d-ab9e-8287801185d5\") "
reconciler.go:224] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kube-api-access-vq2z9\" (UniqueName: \"kubernetes.io/projected/b9ba097d-5dff-4c9d-ab9e-8287801185d5-kube-api-access-vq2z9\") pod \"pod2\" (UID: \"b9ba097d-5dff-4c9d-ab9e-8287801185d5\") "
reconciler.go:254] "Starting operationExecutor.MountVolume for volume \"pvc-38c829ea-060b-482a-84e7-39fdecc173da\" (UniqueName: \"kubernetes.io/csi/nfs.csi.k8s.io^192.168.1.60/nfs-private/pvc-38c829ea-060b-482a-84e7-39fdecc173da\") pod \"pod2\" (UID: \"b9ba097d-5dff-4c9d-ab9e-8287801185d5\") "
reconciler.go:269] "operationExecutor.MountVolume started for volume \"pvc-38c829ea-060b-482a-84e7-39fdecc173da\" (UniqueName: \"kubernetes.io/csi/nfs.csi.k8s.io^192.168.1.60/nfs-private/pvc-38c829ea-060b-482a-84e7-39fdecc173da\") pod \"pod2\" (UID: \"b9ba097d-5dff-4c9d-ab9e-8287801185d5\") "
reconciler.go:254] "Starting operationExecutor.MountVolume for volume \"kube-api-access-vq2z9\" (UniqueName: \"kubernetes.io/projected/b9ba097d-5dff-4c9d-ab9e-8287801185d5-kube-api-access-vq2z9\") pod \"pod2\" (UID: \"b9ba097d-5dff-4c9d-ab9e-8287801185d5\") "
reconciler.go:269] "operationExecutor.MountVolume started for volume \"kube-api-access-vq2z9\" (UniqueName: \"kubernetes.io/projected/b9ba097d-5dff-4c9d-ab9e-8287801185d5-kube-api-access-vq2z9\") pod \"pod2\" (UID: \"b9ba097d-5dff-4c9d-ab9e-8287801185d5\") "

```

```
reconciler.Run()
    reconciler.reconciliationLoopFunc()
        reconciler.reconcile()
            reconciler.mountAttachVolumes()
```

```go
func (rc *reconciler) Run(stopCh <-chan struct{}) {
	wait.Until(rc.reconciliationLoopFunc(), rc.loopSleepDuration, stopCh)       //■
}

func (rc *reconciler) reconciliationLoopFunc() func() {
	return func() {
		rc.reconcile()                                      //■

		// Sync the state with the reality once after all existing pods are added to the desired state from all sources.
		// Otherwise, the reconstruct process may clean up pods' volumes that are still in use because
		// desired state of world does not contain a complete list of pods.
		if rc.populatorHasAddedPods() && !rc.StatesHasBeenSynced() {
			klog.InfoS("Reconciler: start to sync state")
			rc.sync()
		}
	}
}
```

```go
func (rc *reconciler) reconcile() {
	// Unmounts are triggered before mounts so that a volume that was
	// referenced by a pod that was deleted and is now referenced by another
	// pod is unmounted from the first pod before being mounted to the new
	// pod.
	rc.unmountVolumes()

	// Next we mount required volumes. This function could also trigger
	// attach if kubelet is responsible for attaching volumes.
	// If underlying PVC was resized while in-use then this function also handles volume
	// resizing.
	rc.mountAttachVolumes()                                 //■

	// Ensure devices that should be detached/unmounted are detached/unmounted.
	rc.unmountDetachDevices()
}
```

```go
func (rc *reconciler) mountAttachVolumes() {
	// Ensure volumes that should be attached/mounted are attached/mounted.
	for _, volumeToMount := range rc.desiredStateOfWorld.GetVolumesToMount() {
		volMounted, devicePath, err := rc.actualStateOfWorld.PodExistsInVolume(volumeToMount.PodName, volumeToMount.VolumeName)
		volumeToMount.DevicePath = devicePath
		if cache.IsVolumeNotAttachedError(err) {
			if rc.controllerAttachDetachEnabled || !volumeToMount.PluginIsAttachable {
				// Volume is not attached (or doesn't implement attacher), kubelet attach is disabled, wait
				// for controller to finish attaching volume.
				klog.V(5).InfoS(volumeToMount.GenerateMsgDetailed("Starting operationExecutor.VerifyControllerAttachedVolume", ""), "pod", klog.KObj(volumeToMount.Pod))        //★
				err := rc.operationExecutor.VerifyControllerAttachedVolume(
					volumeToMount.VolumeToMount,
					rc.nodeName,
					rc.actualStateOfWorld)
				if err != nil && !isExpectedError(err) {
					klog.ErrorS(err, volumeToMount.GenerateErrorDetailed(fmt.Sprintf("operationExecutor.VerifyControllerAttachedVolume failed (controllerAttachDetachEnabled %v)", rc.controllerAttachDetachEnabled), err).Error(), "pod", klog.KObj(volumeToMount.Pod))
				}
				if err == nil {
					klog.InfoS(volumeToMount.GenerateMsgDetailed("operationExecutor.VerifyControllerAttachedVolume started", ""), "pod", klog.KObj(volumeToMount.Pod))
				}       //★
			} else {
				// Volume is not attached to node, kubelet attach is enabled, volume implements an attacher,
				// so attach it
				volumeToAttach := operationexecutor.VolumeToAttach{
					VolumeName: volumeToMount.VolumeName,
					VolumeSpec: volumeToMount.VolumeSpec,
					NodeName:   rc.nodeName,
				}
				klog.V(5).InfoS(volumeToAttach.GenerateMsgDetailed("Starting operationExecutor.AttachVolume", ""), "pod", klog.KObj(volumeToMount.Pod))
				err := rc.operationExecutor.AttachVolume(volumeToAttach, rc.actualStateOfWorld)
				if err != nil && !isExpectedError(err) {
					klog.ErrorS(err, volumeToMount.GenerateErrorDetailed(fmt.Sprintf("operationExecutor.AttachVolume failed (controllerAttachDetachEnabled %v)", rc.controllerAttachDetachEnabled), err).Error(), "pod", klog.KObj(volumeToMount.Pod))
				}
				if err == nil {
					klog.InfoS(volumeToMount.GenerateMsgDetailed("operationExecutor.AttachVolume started", ""), "pod", klog.KObj(volumeToMount.Pod))
				}
			}
		} else if !volMounted || cache.IsRemountRequiredError(err) {
			// Volume is not mounted, or is already mounted, but requires remounting
			remountingLogStr := ""
			isRemount := cache.IsRemountRequiredError(err)
			if isRemount {
				remountingLogStr = "Volume is already mounted to pod, but remount was requested."
			}
			klog.V(4).InfoS(volumeToMount.GenerateMsgDetailed("Starting operationExecutor.MountVolume", remountingLogStr), "pod", klog.KObj(volumeToMount.Pod)) //★
			err := rc.operationExecutor.MountVolume(
				rc.waitForAttachTimeout,
				volumeToMount.VolumeToMount,
				rc.actualStateOfWorld,
				isRemount)
			if err != nil && !isExpectedError(err) {
				klog.ErrorS(err, volumeToMount.GenerateErrorDetailed(fmt.Sprintf("operationExecutor.MountVolume failed (controllerAttachDetachEnabled %v)", rc.controllerAttachDetachEnabled), err).Error(), "pod", klog.KObj(volumeToMount.Pod))
			}
			if err == nil {
				if remountingLogStr == "" {
					klog.V(1).InfoS(volumeToMount.GenerateMsgDetailed("operationExecutor.MountVolume started", remountingLogStr), "pod", klog.KObj(volumeToMount.Pod))  //★
				} else {
					klog.V(5).InfoS(volumeToMount.GenerateMsgDetailed("operationExecutor.MountVolume started", remountingLogStr), "pod", klog.KObj(volumeToMount.Pod))
				}
			}
		} else if cache.IsFSResizeRequiredError(err) &&
			utilfeature.DefaultFeatureGate.Enabled(features.ExpandInUsePersistentVolumes) {
			klog.V(4).InfoS(volumeToMount.GenerateMsgDetailed("Starting operationExecutor.ExpandInUseVolume", ""), "pod", klog.KObj(volumeToMount.Pod))
			err := rc.operationExecutor.ExpandInUseVolume(
				volumeToMount.VolumeToMount,
				rc.actualStateOfWorld)
			if err != nil && !isExpectedError(err) {
				klog.ErrorS(err, volumeToMount.GenerateErrorDetailed("operationExecutor.ExpandInUseVolume failed", err).Error(), "pod", klog.KObj(volumeToMount.Pod))
			}
			if err == nil {
				klog.V(4).InfoS(volumeToMount.GenerateMsgDetailed("operationExecutor.ExpandInUseVolume started", ""), "pod", klog.KObj(volumeToMount.Pod))
			}
		}
	}
}
```

```text
Dec 28 02:17:43 kind-worker2 kubelet[248]: I1228 02:17:43.984452     248 csi_mounter.go:87] kubernetes.io/csi: mounter.GetPath generated [/var/lib/kubelet/pods/b9ba097d-5dff-4c9d-ab9e-8287801185d5/volumes/kubernetes.io~csi/pvc-38c829ea-060b-482a-84e7-39fdecc173da/mount]
Dec 28 02:17:43 kind-worker2 kubelet[248]: I1228 02:17:43.984830     248 csi_plugin.go:438] kubernetes.io/csi: created path successfully [/var/lib/kubelet/pods/b9ba097d-5dff-4c9d-ab9e-8287801185d5/volumes/kubernetes.io~csi/pvc-38c829ea-060b-482a-84e7-39fdecc173da]
Dec 28 02:17:43 kind-worker2 kubelet[248]: I1228 02:17:43.984947     248 csi_util.go:63] kubernetes.io/csi: saving volume data file [/var/lib/kubelet/pods/b9ba097d-5dff-4c9d-ab9e-8287801185d5/volumes/kubernetes.io~csi/pvc-38c829ea-060b-482a-84e7-39fdecc173da/vol_data.json]
Dec 28 02:17:43 kind-worker2 kubelet[248]: I1228 02:17:43.985196     248 csi_util.go:72] kubernetes.io/csi: volume data file saved successfully [/var/lib/kubelet/pods/b9ba097d-5dff-4c9d-ab9e-8287801185d5/volumes/kubernetes.io~csi/pvc-38c829ea-060b-482a-84e7-39fdecc173da/vol_data.json]
Dec 28 02:17:43 kind-worker2 kubelet[248]: I1228 02:17:43.985224     248 csi_plugin.go:474] kubernetes.io/csi: mounter created successfully
```

**pkg\volume\csi\csi_plugin.go**

```go
func (p *csiPlugin) NewMounter(
	spec *volume.Spec,
	pod *api.Pod,
	_ volume.VolumeOptions) (volume.Mounter, error) {

	volSrc, pvSrc, err := getSourceFromSpec(spec)
	if err != nil {
		return nil, err
	}

	var (
		driverName   string
		volumeHandle string
		readOnly     bool
	)

	switch {
	case volSrc != nil && utilfeature.DefaultFeatureGate.Enabled(features.CSIInlineVolume):
		volumeHandle = makeVolumeHandle(string(pod.UID), spec.Name())
		driverName = volSrc.Driver
		if volSrc.ReadOnly != nil {
			readOnly = *volSrc.ReadOnly
		}
	case pvSrc != nil:
		driverName = pvSrc.Driver
		volumeHandle = pvSrc.VolumeHandle
		readOnly = spec.ReadOnly
	default:
		return nil, errors.New(log("volume source not found in volume.Spec"))
	}

	volumeLifecycleMode, err := p.getVolumeLifecycleMode(spec)
	if err != nil {
		return nil, err
	}

	// Check CSIDriver.Spec.Mode to ensure that the CSI driver
	// supports the current volumeLifecycleMode.
	if err := p.supportsVolumeLifecycleMode(driverName, volumeLifecycleMode); err != nil {
		return nil, err
	}

	fsGroupPolicy, err := p.getFSGroupPolicy(driverName)
	if err != nil {
		return nil, err
	}

	k8s := p.host.GetKubeClient()
	if k8s == nil {
		return nil, errors.New(log("failed to get a kubernetes client"))
	}

	kvh, ok := p.host.(volume.KubeletVolumeHost)
	if !ok {
		return nil, errors.New(log("cast from VolumeHost to KubeletVolumeHost failed"))
	}

	mounter := &csiMountMgr{
		plugin:              p,
		k8s:                 k8s,
		spec:                spec,
		pod:                 pod,
		podUID:              pod.UID,
		driverName:          csiDriverName(driverName),
		volumeLifecycleMode: volumeLifecycleMode,
		fsGroupPolicy:       fsGroupPolicy,
		volumeID:            volumeHandle,
		specVolumeID:        spec.Name(),
		readOnly:            readOnly,
		kubeVolHost:         kvh,
	}
	mounter.csiClientGetter.driverName = csiDriverName(driverName)

	// Save volume info in pod dir
	dir := mounter.GetPath()                                                                //■
	dataDir := filepath.Dir(dir) // dropoff /mount at end

	if err := os.MkdirAll(dataDir, 0750); err != nil {
		return nil, errors.New(log("failed to create dir %#v:  %v", dataDir, err))
	}
	klog.V(4).Info(log("created path successfully [%s]", dataDir))                          //★

	mounter.MetricsProvider = NewMetricsCsi(volumeHandle, dir, csiDriverName(driverName))

	// persist volume info data for teardown
	node := string(p.host.GetNodeName())
	volData := map[string]string{
		volDataKey.specVolID:           spec.Name(),
		volDataKey.volHandle:           volumeHandle,
		volDataKey.driverName:          driverName,
		volDataKey.nodeName:            node,
		volDataKey.volumeLifecycleMode: string(volumeLifecycleMode),
	}

	attachID := getAttachmentName(volumeHandle, driverName, node)
	volData[volDataKey.attachmentID] = attachID

	err = saveVolumeData(dataDir, volDataFileName, volData)                                 //■
	defer func() {
		// Only if there was an error and volume operation was considered
		// finished, we should remove the directory.
		if err != nil && volumetypes.IsOperationFinishedError(err) {
			// attempt to cleanup volume mount dir.
			if err = removeMountDir(p, dir); err != nil {
				klog.Error(log("attacher.MountDevice failed to remove mount dir after error [%s]: %v", dir, err))
			}
		}
	}()

	if err != nil {
		errorMsg := log("csi.NewMounter failed to save volume info data: %v", err)
		klog.Error(errorMsg)

		return nil, errors.New(errorMsg)
	}

	klog.V(4).Info(log("mounter created successfully"))                                     //★

	return mounter, nil
}
```

**pkg\volume\csi\csi_mounter.go**

```go
func (c *csiMountMgr) GetPath() string {
	dir := GetCSIMounterPath(filepath.Join(getTargetPath(c.podUID, c.specVolumeID, c.plugin.host)))
	klog.V(4).Info(log("mounter.GetPath generated [%s]", dir))              //★
	return dir
}
```

**pkg\volume\csi\csi_util.go**

```go
// saveVolumeData persists parameter data as json file at the provided location
func saveVolumeData(dir string, fileName string, data map[string]string) error {
	dataFilePath := filepath.Join(dir, fileName)
	klog.V(4).Info(log("saving volume data file [%s]", dataFilePath))                       //★
	file, err := os.Create(dataFilePath)
	if err != nil {
		return errors.New(log("failed to save volume data file %s: %v", dataFilePath, err))
	}
	defer file.Close()
	if err := json.NewEncoder(file).Encode(data); err != nil {
		return errors.New(log("failed to save volume data file %s: %v", dataFilePath, err))
	}
	klog.V(4).Info(log("volume data file saved successfully [%s]", dataFilePath))           //★
	return nil
}
```

```text
csi_attacher.go:260] kubernetes.io/csi: attacher.GetDeviceMountPath(&{nil &PersistentVolume{ObjectMeta:{pvc-38c829ea-060b-482a-84e7-39fdecc173da    f67e2772-d93e-4fa0-a61c-9047388de840 170286 0 2021-12-27 23:50:46 +0000 UTC <nil> <nil> map[] map[pv.kubernetes.io/provisioned-by:nfs.csi.k8s.io] [] [kubernetes.io/pv-protection]  [{csi-provisioner Update v1 2021-12-27 23:50:46 +0000 UTC FieldsV1 {"f:metadata":{"f:annotations":{".":{},"f:pv.kubernetes.io/provisioned-by":{}}},"f:spec":{"f:accessModes":{},"f:capacity":{".":{},"f:storage":{}},"f:claimRef":{},"f:csi":{".":{},"f:driver":{},"f:volumeAttributes":{".":{},"f:server":{},"f:share":{},"f:storage.kubernetes.io/csiProvisionerIdentity":{}},"f:volumeHandle":{}},"f:mountOptions":{},"f:persistentVolumeReclaimPolicy":{},"f:storageClassName":{},"f:volumeMode":{}}} } {kube-controller-manager Update v1 2021-12-27 23:50:46 +0000 UTC FieldsV1 {"f:status":{"f:phase":{}}} status}]},Spec:PersistentVolumeSpec{Capacity:ResourceList{storage: {{1073741824 0} {<nil>} 1Gi BinarySI},},PersistentVolumeSource:PersistentVolumeSource{GCEPersistentDisk:nil,AWSElasticBlockStore:nil,HostPath:nil,Glusterfs:nil,NFS:nil,RBD:nil,ISCSI:nil,Cinder:nil,CephFS:nil,FC:nil,Flocker:nil,FlexVolume:nil,AzureFile:nil,VsphereVolume:nil,Quobyte:nil,AzureDisk:nil,PhotonPersistentDisk:nil,PortworxVolume:nil,ScaleIO:nil,Local:nil,StorageOS:nil,CSI:&CSIPersistentVolumeSource{Driver:nfs.csi.k8s.io,VolumeHandle:192.168.1.60/nfs-private/pvc-38c829ea-060b-482a-84e7-39fdecc173da,ReadOnly:false,FSType:,VolumeAttributes:map[string]string{server: 192.168.1.60,share: /nfs-private/pvc-38c829ea-060b-482a-84e7-39fdecc173da,storage.kubernetes.io/csiProvisionerIdentity: 1640570259025-8081-nfs.csi.k8s.io,},ControllerPublishSecretRef:nil,NodeStageSecretRef:nil,NodePublishSecretRef:nil,ControllerExpandSecretRef:nil,},},AccessModes:[ReadWriteOnce],ClaimRef:&ObjectReference{Kind:PersistentVolumeClaim,Namespace:default,Name:pvc1,UID:38c829ea-060b-482a-84e7-39fdecc173da,APIVersion:v1,ResourceVersion:170280,FieldPath:,},PersistentVolumeReclaimPolicy:Delete,StorageClassName:nfs-csi,MountOptions:[hard nfsvers=4.1],VolumeMode:*Filesystem,NodeAffinity:nil,},Status:PersistentVolumeStatus{Phase:Bound,Message:,Reason:,},} false false false})

csi_attacher.go:265] attacher.GetDeviceMountPath succeeded, deviceMountPath: /var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-38c829ea-060b-482a-84e7-39fdecc173da/globalmount

csi_attacher.go:270] kubernetes.io/csi: attacher.MountDevice(, /var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-38c829ea-060b-482a-84e7-39fdecc173da/globalmount)
csi_client.go:667] kubernetes.io/csi: calling NodeGetCapabilities rpc to determine if the node service has STAGE_UNSTAGE_VOLUME capability
csi_client.go:531] kubernetes.io/csi: creating new gRPC connection for [unix:///var/lib/kubelet/plugins/csi-nfsplugin/csi.sock]
～この間のログを大幅に消去した～
csi_attacher.go:327] kubernetes.io/csi: created target path successfully [/var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-38c829ea-060b-482a-84e7-39fdecc173da/globalmount]
csi_util.go:63] kubernetes.io/csi: saving volume data file [/var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-38c829ea-060b-482a-84e7-39fdecc173da/vol_data.json]
csi_util.go:72] kubernetes.io/csi: volume data file saved successfully [/var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-38c829ea-060b-482a-84e7-39fdecc173da/vol_data.json]
csi_attacher.go:354] kubernetes.io/csi: attacher.MountDevice STAGE_UNSTAGE_VOLUME capability not set. Skipping MountDevice...
```

csiAttacher.GetDeviceMountPath()                どこから呼ばれる？
csiAttacher.MountDevice()                       どこから呼ばれる？
    csiDriverClient.NodeSupportsStageUnstage()
        csiDriverClient.nodeSupportsCapability()

nodeV1ClientCreator() = newV1NodeClient()       どこから呼ばれる？
    newGrpcConn()

**pkg\volume\csi\csi_attacher.go**

```go
func (c *csiAttacher) GetDeviceMountPath(spec *volume.Spec) (string, error) {
	klog.V(4).Info(log("attacher.GetDeviceMountPath(%v)", spec))                            //★
	deviceMountPath, err := makeDeviceMountPath(c.plugin, spec)
	if err != nil {
		return "", errors.New(log("attacher.GetDeviceMountPath failed to make device mount path: %v", err))
	}
	klog.V(4).Infof("attacher.GetDeviceMountPath succeeded, deviceMountPath: %s", deviceMountPath)  //★
	return deviceMountPath, nil
}
```

```go
func (c *csiAttacher) MountDevice(spec *volume.Spec, devicePath string, deviceMountPath string, deviceMounterArgs volume.DeviceMounterArgs) error {
	klog.V(4).Infof(log("attacher.MountDevice(%s, %s)", devicePath, deviceMountPath))               //★

	if deviceMountPath == "" {
		return errors.New(log("attacher.MountDevice failed, deviceMountPath is empty"))
	}

	// Setup
	if spec == nil {
		return errors.New(log("attacher.MountDevice failed, spec is nil"))
	}
	csiSource, err := getPVSourceFromSpec(spec)
	if err != nil {
		return errors.New(log("attacher.MountDevice failed to get CSIPersistentVolumeSource: %v", err))
	}

	// lets check if node/unstage is supported
	if c.csiClient == nil {
		c.csiClient, err = newCsiDriverClient(csiDriverName(csiSource.Driver))
		if err != nil {
			return errors.New(log("attacher.MountDevice failed to create newCsiDriverClient: %v", err))
		}
	}
	csi := c.csiClient

	ctx, cancel := createCSIOperationContext(spec, c.watchTimeout)
	defer cancel()
	// Check whether "STAGE_UNSTAGE_VOLUME" is set
	stageUnstageSet, err := csi.NodeSupportsStageUnstage(ctx)                               //★
	if err != nil {
		return err
	}

	// Get secrets and publish context required for mountDevice
	nodeName := string(c.plugin.host.GetNodeName())
	publishContext, err := c.plugin.getPublishContext(c.k8s, csiSource.VolumeHandle, csiSource.Driver, nodeName)

	if err != nil {
		return volumetypes.NewTransientOperationFailure(err.Error())
	}

	nodeStageSecrets := map[string]string{}
	// we only require secrets if csiSource has them and volume has NodeStage capability
	if csiSource.NodeStageSecretRef != nil && stageUnstageSet {
		nodeStageSecrets, err = getCredentialsFromSecret(c.k8s, csiSource.NodeStageSecretRef)
		if err != nil {
			err = fmt.Errorf("fetching NodeStageSecretRef %s/%s failed: %v",
				csiSource.NodeStageSecretRef.Namespace, csiSource.NodeStageSecretRef.Name, err)
			// if we failed to fetch secret then that could be a transient error
			return volumetypes.NewTransientOperationFailure(err.Error())
		}
	}

	// Store volume metadata for UnmountDevice. Keep it around even if the
	// driver does not support NodeStage, UnmountDevice still needs it.
	if err = os.MkdirAll(deviceMountPath, 0750); err != nil {
		return errors.New(log("attacher.MountDevice failed to create dir %#v:  %v", deviceMountPath, err))
	}
	klog.V(4).Info(log("created target path successfully [%s]", deviceMountPath))                               //★
	dataDir := filepath.Dir(deviceMountPath)
	data := map[string]string{
		volDataKey.volHandle:  csiSource.VolumeHandle,
		volDataKey.driverName: csiSource.Driver,
	}

	err = saveVolumeData(dataDir, volDataFileName, data)                                                        //■
	defer func() {
		// Only if there was an error and volume operation was considered
		// finished, we should remove the directory.
		if err != nil && volumetypes.IsOperationFinishedError(err) {
			// clean up metadata
			klog.Errorf(log("attacher.MountDevice failed: %v", err))
			if err := removeMountDir(c.plugin, deviceMountPath); err != nil {
				klog.Error(log("attacher.MountDevice failed to remove mount dir after error [%s]: %v", deviceMountPath, err))
			}
		}
	}()

	if err != nil {
		errMsg := log("failed to save volume info data: %v", err)
		klog.Error(errMsg)
		return errors.New(errMsg)
	}

	if !stageUnstageSet {
		klog.Infof(log("attacher.MountDevice STAGE_UNSTAGE_VOLUME capability not set. Skipping MountDevice..."))        //★
		// defer does *not* remove the metadata file and it's correct - UnmountDevice needs it there.
		return nil
	}

	//TODO (vladimirvivien) implement better AccessModes mapping between k8s and CSI
	accessMode := v1.ReadWriteOnce
	if spec.PersistentVolume.Spec.AccessModes != nil {
		accessMode = spec.PersistentVolume.Spec.AccessModes[0]
	}

	var mountOptions []string
	if spec.PersistentVolume != nil && spec.PersistentVolume.Spec.MountOptions != nil {
		mountOptions = spec.PersistentVolume.Spec.MountOptions
	}

	var nodeStageFSGroupArg *int64
	if utilfeature.DefaultFeatureGate.Enabled(features.DelegateFSGroupToCSIDriver) {
		driverSupportsCSIVolumeMountGroup, err := csi.NodeSupportsVolumeMountGroup(ctx)                 //■ここからNodeSupportsCapabilityへ？
		if err != nil {
			return volumetypes.NewTransientOperationFailure(log("attacher.MountDevice failed to determine if the node service has VOLUME_MOUNT_GROUP capability: %v", err))
		}

		if driverSupportsCSIVolumeMountGroup {
			klog.V(3).Infof("Driver %s supports applying FSGroup (has VOLUME_MOUNT_GROUP node capability). Delegating FSGroup application to the driver through NodeStageVolume.", csiSource.Driver)
			nodeStageFSGroupArg = deviceMounterArgs.FsGroup
		}
	}

	fsType := csiSource.FSType
	err = csi.NodeStageVolume(ctx,
		csiSource.VolumeHandle,
		publishContext,
		deviceMountPath,
		fsType,
		accessMode,
		nodeStageSecrets,
		csiSource.VolumeAttributes,
		mountOptions,
		nodeStageFSGroupArg)

	if err != nil {
		return err
	}

	klog.V(4).Infof(log("attacher.MountDevice successfully requested NodeStageVolume [%s]", deviceMountPath))
	return err
}
```

**pkg\volume\csi\csi_client.go**

```go
func (c *csiDriverClient) NodeSupportsStageUnstage(ctx context.Context) (bool, error) {
	return c.nodeSupportsCapability(ctx, csipbv1.NodeServiceCapability_RPC_STAGE_UNSTAGE_VOLUME)
}
```

```go
func (c *csiDriverClient) nodeSupportsCapability(ctx context.Context, capabilityType csipbv1.NodeServiceCapability_RPC_Type) (bool, error) {
	klog.V(4).Info(log("calling NodeGetCapabilities rpc to determine if the node service has %s capability", capabilityType))   //★
	capabilities, err := c.nodeGetCapabilities(ctx)
	if err != nil {
		return false, err
	}

	for _, capability := range capabilities {
		if capability == nil || capability.GetRpc() == nil {
			continue
		}
		if capability.GetRpc().GetType() == capabilityType {
			return true, nil
		}
	}
	return false, nil
}
```

```go
func newCsiDriverClient(driverName csiDriverName) (*csiDriverClient, error) {
	if driverName == "" {
		return nil, fmt.Errorf("driver name is empty")
	}

	existingDriver, driverExists := csiDrivers.Get(string(driverName))
	if !driverExists {
		return nil, fmt.Errorf("driver name %s not found in the list of registered CSI drivers", driverName)
	}

	nodeV1ClientCreator := newV1NodeClient                      //■
	return &csiDriverClient{
		driverName:          driverName,
		addr:                csiAddr(existingDriver.endpoint),
		nodeV1ClientCreator: nodeV1ClientCreator,
		metricsManager:      NewCSIMetricsManager(string(driverName)),
	}, nil
}

// newV1NodeClient creates a new NodeClient with the internally used gRPC
// connection set up. It also returns a closer which must to be called to close
// the gRPC connection when the NodeClient is not used anymore.
// This is the default implementation for the nodeV1ClientCreator, used in
// newCsiDriverClient.
func newV1NodeClient(addr csiAddr, metricsManager *MetricsManager) (nodeClient csipbv1.NodeClient, closer io.Closer, err error) {
	var conn *grpc.ClientConn
	conn, err = newGrpcConn(addr, metricsManager)                           //■
	if err != nil {
		return nil, nil, err
	}

	nodeClient = csipbv1.NewNodeClient(conn)
	return nodeClient, conn, nil
}

func newGrpcConn(addr csiAddr, metricsManager *MetricsManager) (*grpc.ClientConn, error) {
	network := "unix"
	klog.V(4).Infof(log("creating new gRPC connection for [%s://%s]", network, addr))           //★

	return grpc.Dial(
		string(addr),
		grpc.WithInsecure(),
		grpc.WithContextDialer(func(ctx context.Context, target string) (net.Conn, error) {
			return (&net.Dialer{}).DialContext(ctx, network, target)
		}),
		grpc.WithChainUnaryInterceptor(metricsManager.RecordMetricsInterceptor),
	)
}
```

**vendor\google.golang.org\grpc\clientconn.go**

```go
// DialContext creates a client connection to the given target. By default, it's
// a non-blocking dial (the function won't wait for connections to be
// established, and connecting happens in the background). To make it a blocking
// dial, use WithBlock() dial option.
//
// In the non-blocking case, the ctx does not act against the connection. It
// only controls the setup steps.
//
// In the blocking case, ctx can be used to cancel or expire the pending
// connection. Once this function returns, the cancellation and expiration of
// ctx will be noop. Users should call ClientConn.Close to terminate all the
// pending operations after this function returns.
//
// The target name syntax is defined in
// https://github.com/grpc/grpc/blob/master/doc/naming.md.
// e.g. to use dns resolver, a "dns:///" prefix should be applied to the target.
func DialContext(ctx context.Context, target string, opts ...DialOption) (conn *ClientConn, err error) {
	cc := &ClientConn{
		target:            target,
		csMgr:             &connectivityStateManager{},
		conns:             make(map[*addrConn]struct{}),
		dopts:             defaultDialOptions(),
		blockingpicker:    newPickerWrapper(),
		czData:            new(channelzData),
		firstResolveEvent: grpcsync.NewEvent(),
	}
	cc.retryThrottler.Store((*retryThrottler)(nil))
	cc.safeConfigSelector.UpdateConfigSelector(&defaultConfigSelector{nil})
	cc.ctx, cc.cancel = context.WithCancel(context.Background())

	for _, opt := range opts {
		opt.apply(&cc.dopts)
	}

	chainUnaryClientInterceptors(cc)
	chainStreamClientInterceptors(cc)

	defer func() {
		if err != nil {
			cc.Close()
		}
	}()

	if channelz.IsOn() {
		if cc.dopts.channelzParentID != 0 {
			cc.channelzID = channelz.RegisterChannel(&channelzChannel{cc}, cc.dopts.channelzParentID, target)
			channelz.AddTraceEvent(logger, cc.channelzID, 0, &channelz.TraceEventDesc{
				Desc:     "Channel Created",
				Severity: channelz.CtInfo,
				Parent: &channelz.TraceEventDesc{
					Desc:     fmt.Sprintf("Nested Channel(id:%d) created", cc.channelzID),
					Severity: channelz.CtInfo,
				},
			})
		} else {
			cc.channelzID = channelz.RegisterChannel(&channelzChannel{cc}, 0, target)
			channelz.Info(logger, cc.channelzID, "Channel Created")
		}
		cc.csMgr.channelzID = cc.channelzID
	}

	if !cc.dopts.insecure {
		if cc.dopts.copts.TransportCredentials == nil && cc.dopts.copts.CredsBundle == nil {
			return nil, errNoTransportSecurity
		}
		if cc.dopts.copts.TransportCredentials != nil && cc.dopts.copts.CredsBundle != nil {
			return nil, errTransportCredsAndBundle
		}
	} else {
		if cc.dopts.copts.TransportCredentials != nil || cc.dopts.copts.CredsBundle != nil {
			return nil, errCredentialsConflict
		}
		for _, cd := range cc.dopts.copts.PerRPCCredentials {
			if cd.RequireTransportSecurity() {
				return nil, errTransportCredentialsMissing
			}
		}
	}

	if cc.dopts.defaultServiceConfigRawJSON != nil {
		scpr := parseServiceConfig(*cc.dopts.defaultServiceConfigRawJSON)
		if scpr.Err != nil {
			return nil, fmt.Errorf("%s: %v", invalidDefaultServiceConfigErrPrefix, scpr.Err)
		}
		cc.dopts.defaultServiceConfig, _ = scpr.Config.(*ServiceConfig)
	}
	cc.mkp = cc.dopts.copts.KeepaliveParams

	if cc.dopts.copts.UserAgent != "" {
		cc.dopts.copts.UserAgent += " " + grpcUA
	} else {
		cc.dopts.copts.UserAgent = grpcUA
	}

	if cc.dopts.timeout > 0 {
		var cancel context.CancelFunc
		ctx, cancel = context.WithTimeout(ctx, cc.dopts.timeout)
		defer cancel()
	}
	defer func() {
		select {
		case <-ctx.Done():
			switch {
			case ctx.Err() == err:
				conn = nil
			case err == nil || !cc.dopts.returnLastError:
				conn, err = nil, ctx.Err()
			default:
				conn, err = nil, fmt.Errorf("%v: %v", ctx.Err(), err)
			}
		default:
		}
	}()

	scSet := false
	if cc.dopts.scChan != nil {
		// Try to get an initial service config.
		select {
		case sc, ok := <-cc.dopts.scChan:
			if ok {
				cc.sc = &sc
				cc.safeConfigSelector.UpdateConfigSelector(&defaultConfigSelector{&sc})
				scSet = true
			}
		default:
		}
	}
	if cc.dopts.bs == nil {
		cc.dopts.bs = backoff.DefaultExponential
	}

	// Determine the resolver to use.
	cc.parsedTarget = grpcutil.ParseTarget(cc.target, cc.dopts.copts.Dialer != nil)
	channelz.Infof(logger, cc.channelzID, "parsed scheme: %q", cc.parsedTarget.Scheme)
	resolverBuilder := cc.getResolver(cc.parsedTarget.Scheme)
	if resolverBuilder == nil {
		// If resolver builder is still nil, the parsed target's scheme is
		// not registered. Fallback to default resolver and set Endpoint to
		// the original target.
		channelz.Infof(logger, cc.channelzID, "scheme %q not registered, fallback to default scheme", cc.parsedTarget.Scheme)   //★
		cc.parsedTarget = resolver.Target{
			Scheme:   resolver.GetDefaultScheme(),
			Endpoint: target,
		}
		resolverBuilder = cc.getResolver(cc.parsedTarget.Scheme)
		if resolverBuilder == nil {
			return nil, fmt.Errorf("could not get resolver for default scheme: %q", cc.parsedTarget.Scheme)
		}
	}

	creds := cc.dopts.copts.TransportCredentials
	if creds != nil && creds.Info().ServerName != "" {
		cc.authority = creds.Info().ServerName
	} else if cc.dopts.insecure && cc.dopts.authority != "" {
		cc.authority = cc.dopts.authority
	} else if strings.HasPrefix(cc.target, "unix:") || strings.HasPrefix(cc.target, "unix-abstract:") {
		cc.authority = "localhost"
	} else if strings.HasPrefix(cc.parsedTarget.Endpoint, ":") {
		cc.authority = "localhost" + cc.parsedTarget.Endpoint
	} else {
		// Use endpoint from "scheme://authority/endpoint" as the default
		// authority for ClientConn.
		cc.authority = cc.parsedTarget.Endpoint
	}

	if cc.dopts.scChan != nil && !scSet {
		// Blocking wait for the initial service config.
		select {
		case sc, ok := <-cc.dopts.scChan:
			if ok {
				cc.sc = &sc
				cc.safeConfigSelector.UpdateConfigSelector(&defaultConfigSelector{&sc})
			}
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}
	if cc.dopts.scChan != nil {
		go cc.scWatcher()
	}

	var credsClone credentials.TransportCredentials
	if creds := cc.dopts.copts.TransportCredentials; creds != nil {
		credsClone = creds.Clone()
	}
	cc.balancerBuildOpts = balancer.BuildOptions{
		DialCreds:        credsClone,
		CredsBundle:      cc.dopts.copts.CredsBundle,
		Dialer:           cc.dopts.copts.Dialer,
		CustomUserAgent:  cc.dopts.copts.UserAgent,
		ChannelzParentID: cc.channelzID,
		Target:           cc.parsedTarget,
	}

	// Build the resolver.
	rWrapper, err := newCCResolverWrapper(cc, resolverBuilder)
	if err != nil {
		return nil, fmt.Errorf("failed to build resolver: %v", err)
	}
	cc.mu.Lock()
	cc.resolverWrapper = rWrapper
	cc.mu.Unlock()

	// A blocking dial blocks until the clientConn is ready.
	if cc.dopts.block {
		for {
			s := cc.GetState()
			if s == connectivity.Ready {
				break
			} else if cc.dopts.copts.FailOnNonTempDialError && s == connectivity.TransientFailure {
				if err = cc.connectionError(); err != nil {
					terr, ok := err.(interface {
						Temporary() bool
					})
					if ok && !terr.Temporary() {
						return nil, err
					}
				}
			}
			if !cc.WaitForStateChange(ctx, s) {
				// ctx got timeout or canceled.
				if err = cc.connectionError(); err != nil && cc.dopts.returnLastError {
					return nil, err
				}
				return nil, ctx.Err()
			}
		}
	}

	return cc, nil
}
```




**pkg\volume\util\operationexecutor\operation_generator.go**

```go
func (og *operationGenerator) GenerateMountVolumeFunc(
	waitForAttachTimeout time.Duration,
	volumeToMount VolumeToMount,
	actualStateOfWorld ActualStateOfWorldMounterUpdater,
	isRemount bool) volumetypes.GeneratedOperations {

	volumePluginName := unknownVolumePlugin
	volumePlugin, err :=
		og.volumePluginMgr.FindPluginBySpec(volumeToMount.VolumeSpec)
	if err == nil && volumePlugin != nil {
		volumePluginName = volumePlugin.GetPluginName()
	}

	mountVolumeFunc := func() volumetypes.OperationContext {
		// Get mounter plugin
		volumePlugin, err := og.volumePluginMgr.FindPluginBySpec(volumeToMount.VolumeSpec)

		migrated := getMigratedStatusBySpec(volumeToMount.VolumeSpec)

		if err != nil || volumePlugin == nil {
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.FindPluginBySpec failed", err)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		affinityErr := checkNodeAffinity(og, volumeToMount)
		if affinityErr != nil {
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.NodeAffinity check failed", affinityErr)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		volumeMounter, newMounterErr := volumePlugin.NewMounter(
			volumeToMount.VolumeSpec,
			volumeToMount.Pod,
			volume.VolumeOptions{})
		if newMounterErr != nil {
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.NewMounter initialization failed", newMounterErr)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		mountCheckError := checkMountOptionSupport(og, volumeToMount, volumePlugin)
		if mountCheckError != nil {
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.MountOptionSupport check failed", mountCheckError)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		// Enforce ReadWriteOncePod access mode if it is the only one present. This is also enforced during scheduling.
		if utilfeature.DefaultFeatureGate.Enabled(features.ReadWriteOncePod) &&
			actualStateOfWorld.IsVolumeMountedElsewhere(volumeToMount.VolumeName, volumeToMount.PodName) &&
			// Because we do not know what access mode the pod intends to use if there are multiple.
			len(volumeToMount.VolumeSpec.PersistentVolume.Spec.AccessModes) == 1 &&
			v1helper.ContainsAccessMode(volumeToMount.VolumeSpec.PersistentVolume.Spec.AccessModes, v1.ReadWriteOncePod) {

			err = goerrors.New("volume uses the ReadWriteOncePod access mode and is already in use by another pod")
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.SetUp failed", err)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		// Get attacher, if possible
		attachableVolumePlugin, _ :=
			og.volumePluginMgr.FindAttachablePluginBySpec(volumeToMount.VolumeSpec)
		var volumeAttacher volume.Attacher
		if attachableVolumePlugin != nil {
			volumeAttacher, _ = attachableVolumePlugin.NewAttacher()
		}

		// get deviceMounter, if possible
		deviceMountableVolumePlugin, _ := og.volumePluginMgr.FindDeviceMountablePluginBySpec(volumeToMount.VolumeSpec)
		var volumeDeviceMounter volume.DeviceMounter
		if deviceMountableVolumePlugin != nil {
			volumeDeviceMounter, _ = deviceMountableVolumePlugin.NewDeviceMounter()
		}

		var fsGroup *int64
		var fsGroupChangePolicy *v1.PodFSGroupChangePolicy
		if podSc := volumeToMount.Pod.Spec.SecurityContext; podSc != nil {
			if podSc.FSGroup != nil {
				fsGroup = podSc.FSGroup
			}
			if podSc.FSGroupChangePolicy != nil {
				fsGroupChangePolicy = podSc.FSGroupChangePolicy
			}
		}

		devicePath := volumeToMount.DevicePath
		if volumeAttacher != nil {
			// Wait for attachable volumes to finish attaching
			klog.InfoS(volumeToMount.GenerateMsgDetailed("MountVolume.WaitForAttach entering", fmt.Sprintf("DevicePath %q", volumeToMount.DevicePath)), "pod", klog.KObj(volumeToMount.Pod))

			devicePath, err = volumeAttacher.WaitForAttach(
				volumeToMount.VolumeSpec, devicePath, volumeToMount.Pod, waitForAttachTimeout)
			if err != nil {
				// On failure, return error. Caller will log and retry.
				eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.WaitForAttach failed", err)
				return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
			}

			klog.InfoS(volumeToMount.GenerateMsgDetailed("MountVolume.WaitForAttach succeeded", fmt.Sprintf("DevicePath %q", devicePath)), "pod", klog.KObj(volumeToMount.Pod))
		}

		var resizeDone bool
		var resizeError error
		resizeOptions := volume.NodeResizeOptions{
			DevicePath: devicePath,
		}

		if volumeDeviceMounter != nil && actualStateOfWorld.GetDeviceMountState(volumeToMount.VolumeName) != DeviceGloballyMounted {
			deviceMountPath, err :=
				volumeDeviceMounter.GetDeviceMountPath(volumeToMount.VolumeSpec)
			if err != nil {
				// On failure, return error. Caller will log and retry.
				eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.GetDeviceMountPath failed", err)
				return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
			}

			// Mount device to global mount path
			err = volumeDeviceMounter.MountDevice(
				volumeToMount.VolumeSpec,
				devicePath,
				deviceMountPath,
				volume.DeviceMounterArgs{FsGroup: fsGroup},
			)
			if err != nil {
				og.checkForFailedMount(volumeToMount, err)
				og.markDeviceErrorState(volumeToMount, devicePath, deviceMountPath, err, actualStateOfWorld)
				// On failure, return error. Caller will log and retry.
				eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.MountDevice failed", err)
				return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
			}

			klog.InfoS(volumeToMount.GenerateMsgDetailed("MountVolume.MountDevice succeeded", fmt.Sprintf("device mount path %q", deviceMountPath)), "pod", klog.KObj(volumeToMount.Pod))

			// Update actual state of world to reflect volume is globally mounted
			markDeviceMountedErr := actualStateOfWorld.MarkDeviceAsMounted(
				volumeToMount.VolumeName, devicePath, deviceMountPath)
			if markDeviceMountedErr != nil {
				// On failure, return error. Caller will log and retry.
				eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.MarkDeviceAsMounted failed", markDeviceMountedErr)
				return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
			}

			// If volume expansion is performed after MountDevice but before SetUp then
			// deviceMountPath and deviceStagePath is going to be the same.
			// Deprecation: Calling NodeExpandVolume after NodeStage/MountDevice will be deprecated
			// in a future version of k8s.
			resizeOptions.DeviceMountPath = deviceMountPath
			resizeOptions.DeviceStagePath = deviceMountPath
			resizeOptions.CSIVolumePhase = volume.CSIVolumeStaged

			// NodeExpandVolume will resize the file system if user has requested a resize of
			// underlying persistent volume and is allowed to do so.
			resizeDone, resizeError = og.nodeExpandVolume(volumeToMount, actualStateOfWorld, resizeOptions)

			if resizeError != nil {
				klog.Errorf("MountVolume.NodeExpandVolume failed with %v", resizeError)

				// Resize failed. To make sure NodeExpand is re-tried again on the next attempt
				// *before* SetUp(), mark the mounted device as uncertain.
				markDeviceUncertainErr := actualStateOfWorld.MarkDeviceAsUncertain(
					volumeToMount.VolumeName, devicePath, deviceMountPath)
				if markDeviceUncertainErr != nil {
					// just log, return the resizeError error instead
					klog.InfoS(volumeToMount.GenerateMsgDetailed(
						"MountVolume.MountDevice failed to mark volume as uncertain",
						markDeviceUncertainErr.Error()), "pod", klog.KObj(volumeToMount.Pod))
				}
				eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.MountDevice failed while expanding volume", resizeError)
				return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
			}
		}

		if og.checkNodeCapabilitiesBeforeMount {
			if canMountErr := volumeMounter.CanMount(); canMountErr != nil {
				err = fmt.Errorf(
					"verify that your node machine has the required components before attempting to mount this volume type. %s",
					canMountErr)
				eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.CanMount failed", err)
				return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
			}
		}

		// Execute mount
		mountErr := volumeMounter.SetUp(volume.MounterArgs{
			FsUser:              util.FsUserFrom(volumeToMount.Pod),
			FsGroup:             fsGroup,
			DesiredSize:         volumeToMount.DesiredSizeLimit,
			FSGroupChangePolicy: fsGroupChangePolicy,
		})
		// Update actual state of world
		markOpts := MarkVolumeOpts{
			PodName:             volumeToMount.PodName,
			PodUID:              volumeToMount.Pod.UID,
			VolumeName:          volumeToMount.VolumeName,
			Mounter:             volumeMounter,
			OuterVolumeSpecName: volumeToMount.OuterVolumeSpecName,
			VolumeGidVolume:     volumeToMount.VolumeGidValue,
			VolumeSpec:          volumeToMount.VolumeSpec,
			VolumeMountState:    VolumeMounted,
		}
		if mountErr != nil {
			og.checkForFailedMount(volumeToMount, mountErr)
			og.markVolumeErrorState(volumeToMount, markOpts, mountErr, actualStateOfWorld)
			// On failure, return error. Caller will log and retry.
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.SetUp failed", mountErr)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		detailedMsg := volumeToMount.GenerateMsgDetailed("MountVolume.SetUp succeeded", "")
		verbosity := klog.Level(1)
		if isRemount {
			verbosity = klog.Level(4)
		}
		klog.V(verbosity).InfoS(detailedMsg, "pod", klog.KObj(volumeToMount.Pod))
		resizeOptions.DeviceMountPath = volumeMounter.GetPath()
		resizeOptions.CSIVolumePhase = volume.CSIVolumePublished

		// We need to call resizing here again in case resizing was not done during device mount. There could be
		// two reasons of that:
		//	- Volume does not support DeviceMounter interface.
		//	- In case of CSI the volume does not have node stage_unstage capability.
		if !resizeDone {
			_, resizeError = og.nodeExpandVolume(volumeToMount, actualStateOfWorld, resizeOptions)
			if resizeError != nil {
				klog.Errorf("MountVolume.NodeExpandVolume failed with %v", resizeError)
				eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.Setup failed while expanding volume", resizeError)
				// At this point, MountVolume.Setup already succeeded, we should add volume into actual state
				// so that reconciler can clean up volume when needed. However, volume resize failed,
				// we should not mark the volume as mounted to avoid pod starts using it.
				// Considering the above situations, we mark volume as uncertain here so that reconciler will tigger
				// volume tear down when pod is deleted, and also makes sure pod will not start using it.
				if err := actualStateOfWorld.MarkVolumeMountAsUncertain(markOpts); err != nil {
					klog.Errorf(volumeToMount.GenerateErrorDetailed("MountVolume.MarkVolumeMountAsUncertain failed", err).Error())
				}
				return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
			}
		}

		markVolMountedErr := actualStateOfWorld.MarkVolumeAsMounted(markOpts)
		if markVolMountedErr != nil {
			// On failure, return error. Caller will log and retry.
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.MarkVolumeAsMounted failed", markVolMountedErr)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}
		return volumetypes.NewOperationContext(nil, nil, migrated)
	}

	eventRecorderFunc := func(err *error) {
		if *err != nil {
			og.recorder.Eventf(volumeToMount.Pod, v1.EventTypeWarning, kevents.FailedMountVolume, (*err).Error())
		}
	}

	return volumetypes.GeneratedOperations{
		OperationName:     "volume_mount",
		OperationFunc:     mountVolumeFunc,
		EventRecorderFunc: eventRecorderFunc,
		CompleteFunc:      util.OperationCompleteHook(util.GetFullQualifiedPluginNameForVolume(volumePluginName, volumeToMount.VolumeSpec), "volume_mount"),
	}
}
```


operation_generator.go:630] MountVolume.MountDevice succeeded for volume "pvc-38c829ea-060b-482a-84e7-39fdecc173da" (UniqueName: "kubernetes.io/csi/nfs.csi.k8s.io^192.168.1.60/nfs-private/pvc-38c829ea-060b-482a-84e7-39fdecc173da") pod "pod2" (UID: "b9ba097d-5dff-4c9d-ab9e-8287801185d5") device mount path "/var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-38c829ea-060b-482a-84e7-39fdecc173da/globalmount"

Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.014552     248 csi_mounter.go:87] kubernetes.io/csi: mounter.GetPath generated [/var/lib/kubelet/pods/b9ba097d-5dff-4c9d-ab9e-8287801185d5/volumes/kubernetes.io~csi/pvc-38c829ea-060b-482a-84e7-39fdecc173da/mount]
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.014578     248 csi_mounter.go:108] kubernetes.io/csi: Mounter.SetUpAt(/var/lib/kubelet/pods/b9ba097d-5dff-4c9d-ab9e-8287801185d5/volumes/kubernetes.io~csi/pvc-38c829ea-060b-482a-84e7-39fdecc173da/mount)
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.014607     248 csi_client.go:667] kubernetes.io/csi: calling NodeGetCapabilities rpc to determine if the node service has STAGE_UNSTAGE_VOLUME capability
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.014622     248 csi_client.go:531] kubernetes.io/csi: creating new gRPC connection for [unix:///var/lib/kubelet/plugins/csi-nfsplugin/csi.sock]
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.014660     248 clientconn.go:252] [core] parsed scheme: ""
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.014682     248 clientconn.go:258] [core] scheme "" not registered, fallback to default scheme
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.014716     248 resolver_conn_wrapper.go:96] [core] ccResolverWrapper: sending update to cc: {[{/var/lib/kubelet/plugins/csi-nfsplugin/csi.sock  <nil> 0 <nil>}] <nil> <nil>}
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.014732     248 clientconn.go:708] [core] ClientConn switching balancer to "pick_first"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.014747     248 clientconn.go:723] [core] Channel switches to new LB policy "pick_first"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.014793     248 clientconn.go:1114] [core] Subchannel Connectivity change to CONNECTING
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.014835     248 picker_wrapper.go:161] [core] blockingPicker: the picked transport is not ready, loop back to repick
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.014870     248 clientconn.go:1251] [core] Subchannel picks a new address "/var/lib/kubelet/plugins/csi-nfsplugin/csi.sock" to connect
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.014891     248 pickfirst.go:95] [core] pickfirstBalancer: UpdateSubConnState: 0xc001473eb0, {CONNECTING <nil>}
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.014920     248 clientconn.go:435] [core] Channel Connectivity change to CONNECTING
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.015321     248 clientconn.go:1114] [core] Subchannel Connectivity change to READY
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.015400     248 pickfirst.go:95] [core] pickfirstBalancer: UpdateSubConnState: 0xc001473eb0, {READY <nil>}
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.015463     248 clientconn.go:435] [core] Channel Connectivity change to READY
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.015957     248 clientconn.go:435] [core] Channel Connectivity change to SHUTDOWN
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.015988     248 clientconn.go:1114] [core] Subchannel Connectivity change to SHUTDOWN
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016015     248 shared_informer.go:270] caches populated
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016061     248 csi_mounter.go:210] kubernetes.io/csi: created target path successfully [/var/lib/kubelet/pods/b9ba097d-5dff-4c9d-ab9e-8287801185d5/volumes/kubernetes.io~csi/pvc-38c829ea-060b-482a-84e7-39fdecc173da]
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016075     248 shared_informer.go:270] caches populated
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016086     248 controlbuf.go:521] [transport] transport: loopyWriter.run returning. connection error: desc = "transport is closing"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016088     248 csi_plugin.go:989] kubernetes.io/csi: CSIDriver "nfs.csi.k8s.io" does not require pod information
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016184     248 csi_client.go:227] kubernetes.io/csi: calling NodePublishVolume rpc [volid=192.168.1.60/nfs-private/pvc-38c829ea-060b-482a-84e7-39fdecc173da,target_path=/var/lib/kubelet/pods/b9ba097d-5dff-4c9d-ab9e-8287801185d5/volumes/kubernetes.io~csi/pvc-38c829ea-060b-482a-84e7-39fdecc173da/mount]
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016196     248 csi_client.go:667] kubernetes.io/csi: calling NodeGetCapabilities rpc to determine if the node service has SINGLE_NODE_MULTI_WRITER capability
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016204     248 csi_client.go:531] kubernetes.io/csi: creating new gRPC connection for [unix:///var/lib/kubelet/plugins/csi-nfsplugin/csi.sock]
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016225     248 clientconn.go:252] [core] parsed scheme: ""
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016239     248 clientconn.go:258] [core] scheme "" not registered, fallback to default scheme
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016258     248 resolver_conn_wrapper.go:96] [core] ccResolverWrapper: sending update to cc: {[{/var/lib/kubelet/plugins/csi-nfsplugin/csi.sock  <nil> 0 <nil>}] <nil> <nil>}
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016269     248 clientconn.go:708] [core] ClientConn switching balancer to "pick_first"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016277     248 clientconn.go:723] [core] Channel switches to new LB policy "pick_first"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016303     248 clientconn.go:1114] [core] Subchannel Connectivity change to CONNECTING
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016328     248 picker_wrapper.go:161] [core] blockingPicker: the picked transport is not ready, loop back to repick
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016345     248 clientconn.go:1251] [core] Subchannel picks a new address "/var/lib/kubelet/plugins/csi-nfsplugin/csi.sock" to connect
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016351     248 pickfirst.go:95] [core] pickfirstBalancer: UpdateSubConnState: 0xc0014fb170, {CONNECTING <nil>}
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016370     248 clientconn.go:435] [core] Channel Connectivity change to CONNECTING
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016758     248 clientconn.go:1114] [core] Subchannel Connectivity change to READY
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016783     248 pickfirst.go:95] [core] pickfirstBalancer: UpdateSubConnState: 0xc0014fb170, {READY <nil>}
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.016809     248 clientconn.go:435] [core] Channel Connectivity change to READY
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.017257     248 clientconn.go:435] [core] Channel Connectivity change to SHUTDOWN
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.017278     248 clientconn.go:1114] [core] Subchannel Connectivity change to SHUTDOWN
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.017292     248 csi_client.go:531] kubernetes.io/csi: creating new gRPC connection for [unix:///var/lib/kubelet/plugins/csi-nfsplugin/csi.sock]
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.017313     248 clientconn.go:252] [core] parsed scheme: ""
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.017318     248 controlbuf.go:521] [transport] transport: loopyWriter.run returning. connection error: desc = "transport is closing"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.017327     248 clientconn.go:258] [core] scheme "" not registered, fallback to default scheme
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.017348     248 resolver_conn_wrapper.go:96] [core] ccResolverWrapper: sending update to cc: {[{/var/lib/kubelet/plugins/csi-nfsplugin/csi.sock  <nil> 0 <nil>}] <nil> <nil>}
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.017357     248 clientconn.go:708] [core] ClientConn switching balancer to "pick_first"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.017404     248 clientconn.go:723] [core] Channel switches to new LB policy "pick_first"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.017437     248 clientconn.go:1114] [core] Subchannel Connectivity change to CONNECTING
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.017466     248 picker_wrapper.go:161] [core] blockingPicker: the picked transport is not ready, loop back to repick
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.017485     248 clientconn.go:1251] [core] Subchannel picks a new address "/var/lib/kubelet/plugins/csi-nfsplugin/csi.sock" to connect
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.017550     248 pickfirst.go:95] [core] pickfirstBalancer: UpdateSubConnState: 0xc000bd57a0, {CONNECTING <nil>}
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.017569     248 clientconn.go:435] [core] Channel Connectivity change to CONNECTING
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.017724     248 clientconn.go:1114] [core] Subchannel Connectivity change to READY
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.017745     248 pickfirst.go:95] [core] pickfirstBalancer: UpdateSubConnState: 0xc000bd57a0, {READY <nil>}
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.017759     248 clientconn.go:435] [core] Channel Connectivity change to READY
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.085513     248 reconciler.go:254] "Starting operationExecutor.MountVolume for volume \"pvc-38c829ea-060b-482a-84e7-39fdecc173da\" (UniqueName: \"kubernetes.io/csi/nfs.csi.k8s.io^192.168.1.60/nfs-private/pvc-38c829ea-060b-482a-84e7-39fdecc173da\") pod \"pod2\" (UID: \"b9ba097d-5dff-4c9d-ab9e-8287801185d5\") "
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.111202     248 clientconn.go:435] [core] Channel Connectivity change to SHUTDOWN
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.111285     248 clientconn.go:1114] [core] Subchannel Connectivity change to SHUTDOWN
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.111508     248 controlbuf.go:521] [transport] transport: loopyWriter.run returning. connection error: desc = "transport is closing"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.114157     248 csi_mounter.go:297] kubernetes.io/csi: mounter.SetUp successfully requested NodePublish [/var/lib/kubelet/pods/b9ba097d-5dff-4c9d-ab9e-8287801185d5/volumes/kubernetes.io~csi/pvc-38c829ea-060b-482a-84e7-39fdecc173da/mount]
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.114248     248 operation_generator.go:712] MountVolume.SetUp succeeded for volume "pvc-38c829ea-060b-482a-84e7-39fdecc173da" (UniqueName: "kubernetes.io/csi/nfs.csi.k8s.io^192.168.1.60/nfs-private/pvc-38c829ea-060b-482a-84e7-39fdecc173da") pod "pod2" (UID: "b9ba097d-5dff-4c9d-ab9e-8287801185d5")
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.114279     248 csi_mounter.go:87] kubernetes.io/csi: mounter.GetPath generated [/var/lib/kubelet/pods/b9ba097d-5dff-4c9d-ab9e-8287801185d5/volumes/kubernetes.io~csi/pvc-38c829ea-060b-482a-84e7-39fdecc173da/mount]
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.117101     248 volume_manager.go:437] "All volumes are attached and mounted for pod" pod="default/pod2"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.117150     248 kuberuntime_manager.go:539] "Syncing Pod" pod="default/pod2"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.117216     248 kuberuntime_manager.go:482] "No sandbox for pod can be found. Need to start a new one" pod="default/pod2"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.117298     248 kuberuntime_manager.go:727] "computePodActions got for pod" podActions={KillPod:true CreateSandbox:true SandboxID: Attempt:0 NextInitContainerToStart:nil ContainersToStart:[0] ContainersToKill:map[] EphemeralContainersToStart:[]} pod="default/pod2"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.117363     248 kuberuntime_manager.go:736] "SyncPod received new pod, will create a sandbox for it" pod="default/pod2"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.117386     248 kuberuntime_manager.go:743] "Stopping PodSandbox for pod, will start new one" pod="default/pod2"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.117421     248 kuberuntime_manager.go:798] "Creating PodSandbox for pod" pod="default/pod2"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.117844     248 remote_runtime.go:106] "[RemoteRuntimeService] RunPodSandbox" config="&PodSandboxConfig{Metadata:&PodSandboxMetadata{Name:pod2,Uid:b9ba097d-5dff-4c9d-ab9e-8287801185d5,Namespace:default,Attempt:0,},Hostname:pod2,LogDirectory:/var/log/pods/default_pod2_b9ba097d-5dff-4c9d-ab9e-8287801185d5,DnsConfig:&DNSConfig{Servers:[10.96.0.10],Searches:[default.svc.cluster.local svc.cluster.local cluster.local],Options:[ndots:5],},PortMappings:[]*PortMapping{},Labels:map[string]string{io.kubernetes.pod.name: pod2,io.kubernetes.pod.namespace: default,io.kubernetes.pod.uid: b9ba097d-5dff-4c9d-ab9e-8287801185d5,run: pod2,},Annotations:map[string]string{kubectl.kubernetes.io/last-applied-configuration: {\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"run\":\"pod2\"},\"name\":\"pod2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"sleep\",\"infinity\"],\"image\":\"centos:7\",\"name\":\"pod2\",\"volumeMounts\":[{\"mountPath\":\"/mnt/data\",\"name\":\"myvolume\"}]}],\"nodeSelector\":{\"kubernetes.io/hostname\":\"kind-worker2\"},\"volumes\":[{\"name\":\"myvolume\",\"persistentVolumeClaim\":{\"claimName\":\"pvc1\"}}]}}\n,kubernetes.io/config.seen: 2021-12-28T02:17:43.793731871Z,kubernetes.io/config.source: api,},Linux:&LinuxPodSandboxConfig{CgroupParent:/kubelet/kubepods/besteffort/podb9ba097d-5dff-4c9d-ab9e-8287801185d5,SecurityContext:&LinuxSandboxSecurityContext{NamespaceOptions:&NamespaceOption{Network:POD,Pid:CONTAINER,Ipc:POD,TargetId:,},SelinuxOptions:nil,RunAsUser:nil,ReadonlyRootfs:false,SupplementalGroups:[],Privileged:false,SeccompProfilePath:runtime/default,RunAsGroup:nil,Seccomp:&SecurityProfile{ProfileType:RuntimeDefault,LocalhostRef:,},Apparmor:nil,},Sysctls:map[string]string{},},Windows:nil,}" runtimeHandler="" timeout="4m0s"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.175022     248 generic.go:191] "GenericPLEG: Relisting"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.175179     248 remote_runtime.go:199] "[RemoteRuntimeService] ListPodSandbox" filter="nil" timeout="2m0s"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.176974     248 remote_runtime.go:211] "[RemoteRuntimeService] ListPodSandbox Response" filter="nil" items=[&PodSandbox{Id:8aa1fc56e1b975e136fcc911208ce136f7836ac2afc770b3df1e4e82d0ca4cc9,Metadata:&PodSandboxMetadata{Name:kube-proxy-4kxp2,Uid:f7fd2241-6d84-45fe-bb45-5e4ff7c3dc90,Namespace:kube-system,Attempt:0,},State:SANDBOX_READY,CreatedAt:1640567744146327483,Labels:map[string]string{controller-revision-hash: 7c76476f94,io.kubernetes.pod.name: kube-proxy-4kxp2,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: f7fd2241-6d84-45fe-bb45-5e4ff7c3dc90,k8s-app: kube-proxy,pod-template-generation: 1,},Annotations:map[string]string{kubernetes.io/config.seen: 2021-12-27T01:15:42.145279082Z,kubernetes.io/config.source: api,},RuntimeHandler:,} &PodSandbox{Id:6c9e2abee1d3cc91a4b4236373965bc5b2910971019c150c1de6eb6e8127975c,Metadata:&PodSandboxMetadata{Name:kindnet-kcz76,Uid:4d6edc85-1e87-49c6-ae52-7528ef6d853a,Namespace:kube-system,Attempt:0,},State:SANDBOX_READY,CreatedAt:1640567744068106433,Labels:map[string]string{app: kindnet,controller-revision-hash: 6445d85c5c,io.kubernetes.pod.name: kindnet-kcz76,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 4d6edc85-1e87-49c6-ae52-7528ef6d853a,k8s-app: kindnet,pod-template-generation: 1,tier: node,},Annotations:map[string]string{kubernetes.io/config.seen: 2021-12-27T01:15:42.145252924Z,kubernetes.io/config.source: api,},RuntimeHandler:,} &PodSandbox{Id:2162817c2f36c0a60b71e3de5d8a32cf971691280167860b6e8925cfc74c9aad,Metadata:&PodSandboxMetadata{Name:csi-nfs-controller-d775b4db9-6kj9p,Uid:7f344931-7918-40df-a833-0eaa83d1e412,Namespace:kube-system,Attempt:0,},State:SANDBOX_READY,CreatedAt:1640569715859038684,Labels:map[string]string{app: csi-nfs-controller,io.kubernetes.pod.name: csi-nfs-controller-d775b4db9-6kj9p,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 7f344931-7918-40df-a833-0eaa83d1e412,pod-template-hash: d775b4db9,},Annotations:map[string]string{kubernetes.io/config.seen: 2021-12-27T01:48:35.522768731Z,kubernetes.io/config.source: api,},RuntimeHandler:,} &PodSandbox{Id:b81eb4cfa1555e549df9678a54141707d464354d0799acdf54330b79adafa3a3,Metadata:&PodSandboxMetadata{Name:csi-nfs-node-hs4q9,Uid:78e34d6c-417b-408b-bf2a-12ceb26bb982,Namespace:kube-system,Attempt:0,},State:SANDBOX_READY,CreatedAt:1640569716385141797,Labels:map[string]string{app: csi-nfs-node,controller-revision-hash: d5b5cc545,io.kubernetes.pod.name: csi-nfs-node-hs4q9,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 78e34d6c-417b-408b-bf2a-12ceb26bb982,pod-template-generation: 1,},Annotations:map[string]string{kubernetes.io/config.seen: 2021-12-27T01:48:35.767112649Z,kubernetes.io/config.source: api,},RuntimeHandler:,}]
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.177088     248 remote_runtime.go:306] "[RemoteRuntimeService] ListContainers" filter="&ContainerFilter{Id:,State:nil,PodSandboxId:,LabelSelector:map[string]string{},}" timeout="2m0s"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.178740     248 remote_runtime.go:317] "[RemoteRuntimeService] ListContainers Response" filter="&ContainerFilter{Id:,State:nil,PodSandboxId:,LabelSelector:map[string]string{},}" containers=[&Container{Id:92ec9f558b6ee6189686322bfe9519daed872fa3102006f0dacb0bec97a4a33b,PodSandboxId:8aa1fc56e1b975e136fcc911208ce136f7836ac2afc770b3df1e4e82d0ca4cc9,Metadata:&ContainerMetadata{Name:kube-proxy,Attempt:0,},Image:&ImageSpec{Image:sha256:d34e62e43dded439d130408f771232696905ff6b6a900027c289ee55b4df605e,Annotations:map[string]string{},},ImageRef:sha256:d34e62e43dded439d130408f771232696905ff6b6a900027c289ee55b4df605e,State:CONTAINER_RUNNING,CreatedAt:1640567751617687909,Labels:map[string]string{io.kubernetes.container.name: kube-proxy,io.kubernetes.pod.name: kube-proxy-4kxp2,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: f7fd2241-6d84-45fe-bb45-5e4ff7c3dc90,},Annotations:map[string]string{io.kubernetes.container.hash: 123908ae,io.kubernetes.container.restartCount: 0,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:32e6f3235cccd00041058a61c787a5adda69e7bf7ce40dcaee24749ede04061e,PodSandboxId:2162817c2f36c0a60b71e3de5d8a32cf971691280167860b6e8925cfc74c9aad,Metadata:&ContainerMetadata{Name:csi-provisioner,Attempt:0,},Image:&ImageSpec{Image:sha256:e18077242e6d7990e7f18bf5dfb2184c2d57124ae62b6ba2764b71741ed2f4c2,Annotations:map[string]string{},},ImageRef:sha256:e18077242e6d7990e7f18bf5dfb2184c2d57124ae62b6ba2764b71741ed2f4c2,State:CONTAINER_EXITED,CreatedAt:1640570235809622978,Labels:map[string]string{io.kubernetes.container.name: csi-provisioner,io.kubernetes.pod.name: csi-nfs-controller-d775b4db9-6kj9p,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 7f344931-7918-40df-a833-0eaa83d1e412,},Annotations:map[string]string{io.kubernetes.container.hash: 29d2c2b3,io.kubernetes.container.restartCount: 0,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:ae4e5a6803a4a9a2d5808c0418b52bdca909da3cc06d9ee872bcdf9acfc4ed65,PodSandboxId:b81eb4cfa1555e549df9678a54141707d464354d0799acdf54330b79adafa3a3,Metadata:&ContainerMetadata{Name:nfs,Attempt:3,},Image:&ImageSpec{Image:sha256:e193d2eaa1633ba12a6c0111c9a97566e07e4c98a0c3e5b794fa78858fd474d5,Annotations:map[string]string{},},ImageRef:sha256:e193d2eaa1633ba12a6c0111c9a97566e07e4c98a0c3e5b794fa78858fd474d5,State:CONTAINER_RUNNING,CreatedAt:1640570256874312561,Labels:map[string]string{io.kubernetes.container.name: nfs,io.kubernetes.pod.name: csi-nfs-node-hs4q9,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 78e34d6c-417b-408b-bf2a-12ceb26bb982,},Annotations:map[string]string{io.kubernetes.container.hash: f82f63bc,io.kubernetes.container.ports: [{"name":"healthz","hostPort":29653,"containerPort":29653,"protocol":"TCP"}],io.kubernetes.container.restartCount: 3,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:e01eecf7217dba4707fa1cdbd0b3725d0d3ee4a86146f7b44fad4a3fe8ad18ab,PodSandboxId:2162817c2f36c0a60b71e3de5d8a32cf971691280167860b6e8925cfc74c9aad,Metadata:&ContainerMetadata{Name:csi-provisioner,Attempt:1,},Image:&ImageSpec{Image:sha256:e18077242e6d7990e7f18bf5dfb2184c2d57124ae62b6ba2764b71741ed2f4c2,Annotations:map[string]string{},},ImageRef:sha256:e18077242e6d7990e7f18bf5dfb2184c2d57124ae62b6ba2764b71741ed2f4c2,State:CONTAINER_RUNNING,CreatedAt:1640570258701719590,Labels:map[string]string{io.kubernetes.container.name: csi-provisioner,io.kubernetes.pod.name: csi-nfs-controller-d775b4db9-6kj9p,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 7f344931-7918-40df-a833-0eaa83d1e412,},Annotations:map[string]string{io.kubernetes.container.hash: 29d2c2b3,io.kubernetes.container.restartCount: 1,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:2890ad62c49835f8b455ed03e434f7a39dcd2adeeb1afdcf173faa1d239a0674,PodSandboxId:b81eb4cfa1555e549df9678a54141707d464354d0799acdf54330b79adafa3a3,Metadata:&ContainerMetadata{Name:liveness-probe,Attempt:0,},Image:&ImageSpec{Image:sha256:a347a725763c086b9dc98abda1cdaae8c9f6c62bd10e8db33a2205919a9cf8ed,Annotations:map[string]string{},},ImageRef:sha256:a347a725763c086b9dc98abda1cdaae8c9f6c62bd10e8db33a2205919a9cf8ed,State:CONTAINER_RUNNING,CreatedAt:1640570273640601518,Labels:map[string]string{io.kubernetes.container.name: liveness-probe,io.kubernetes.pod.name: csi-nfs-node-hs4q9,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 78e34d6c-417b-408b-bf2a-12ceb26bb982,},Annotations:map[string]string{io.kubernetes.container.hash: 661a0ca1,io.kubernetes.container.restartCount: 0,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:91236bbddf09a83b23b4c50a821c727c10f770ae272e403ea68a111a07c12c26,PodSandboxId:b81eb4cfa1555e549df9678a54141707d464354d0799acdf54330b79adafa3a3,Metadata:&ContainerMetadata{Name:node-driver-registrar,Attempt:0,},Image:&ImageSpec{Image:sha256:f45c8a305a0bb15ff256a32686d56356be69e1b8d469e90a247d279ad6702382,Annotations:map[string]string{},},ImageRef:sha256:f45c8a305a0bb15ff256a32686d56356be69e1b8d469e90a247d279ad6702382,State:CONTAINER_RUNNING,CreatedAt:1640570316368173914,Labels:map[string]string{io.kubernetes.container.name: node-driver-registrar,io.kubernetes.pod.name: csi-nfs-node-hs4q9,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 78e34d6c-417b-408b-bf2a-12ceb26bb982,},Annotations:map[string]string{io.kubernetes.container.hash: 385cd5c0,io.kubernetes.container.restartCount: 0,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:815e473c76e85a6b98e06040c2f33c314c9a3c18bbaa2632312f9237cfecde94,PodSandboxId:6c9e2abee1d3cc91a4b4236373965bc5b2910971019c150c1de6eb6e8127975c,Metadata:&ContainerMetadata{Name:kindnet-cni,Attempt:0,},Image:&ImageSpec{Image:sha256:3105b3ff910efceee092aa79ef80f8cbc8a776b7c02325b586c466a6079c0ccf,Annotations:map[string]string{},},ImageRef:sha256:3105b3ff910efceee092aa79ef80f8cbc8a776b7c02325b586c466a6079c0ccf,State:CONTAINER_RUNNING,CreatedAt:1640567751616629285,Labels:map[string]string{io.kubernetes.container.name: kindnet-cni,io.kubernetes.pod.name: kindnet-kcz76,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 4d6edc85-1e87-49c6-ae52-7528ef6d853a,},Annotations:map[string]string{io.kubernetes.container.hash: 34ee6d28,io.kubernetes.container.restartCount: 0,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:beb4ec0092c2893ef2bb559ac2f18542ef15bf2e1b0175f6c30965629a5bde1b,PodSandboxId:b81eb4cfa1555e549df9678a54141707d464354d0799acdf54330b79adafa3a3,Metadata:&ContainerMetadata{Name:nfs,Attempt:2,},Image:&ImageSpec{Image:sha256:e193d2eaa1633ba12a6c0111c9a97566e07e4c98a0c3e5b794fa78858fd474d5,Annotations:map[string]string{},},ImageRef:sha256:e193d2eaa1633ba12a6c0111c9a97566e07e4c98a0c3e5b794fa78858fd474d5,State:CONTAINER_EXITED,CreatedAt:1640570076820223815,Labels:map[string]string{io.kubernetes.container.name: nfs,io.kubernetes.pod.name: csi-nfs-node-hs4q9,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 78e34d6c-417b-408b-bf2a-12ceb26bb982,},Annotations:map[string]string{io.kubernetes.container.hash: f82f63bc,io.kubernetes.container.ports: [{"name":"healthz","hostPort":29653,"containerPort":29653,"protocol":"TCP"}],io.kubernetes.container.restartCount: 2,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:3e03cc01f742f797920859ee23676a239ffcadec1008456fb929bd7e74f78c4a,PodSandboxId:2162817c2f36c0a60b71e3de5d8a32cf971691280167860b6e8925cfc74c9aad,Metadata:&ContainerMetadata{Name:nfs,Attempt:3,},Image:&ImageSpec{Image:sha256:e193d2eaa1633ba12a6c0111c9a97566e07e4c98a0c3e5b794fa78858fd474d5,Annotations:map[string]string{},},ImageRef:sha256:e193d2eaa1633ba12a6c0111c9a97566e07e4c98a0c3e5b794fa78858fd474d5,State:CONTAINER_RUNNING,CreatedAt:1640570256641136071,Labels:map[string]string{io.kubernetes.container.name: nfs,io.kubernetes.pod.name: csi-nfs-controller-d775b4db9-6kj9p,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 7f344931-7918-40df-a833-0eaa83d1e412,},Annotations:map[string]string{io.kubernetes.container.hash: 3a748bea,io.kubernetes.container.ports: [{"name":"healthz","hostPort":29652,"containerPort":29652,"protocol":"TCP"}],io.kubernetes.container.restartCount: 3,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:43b7ee31c304a7698a4ad0b96f3ec7efb251ea214e22f61e40358d5bf78b3521,PodSandboxId:2162817c2f36c0a60b71e3de5d8a32cf971691280167860b6e8925cfc74c9aad,Metadata:&ContainerMetadata{Name:nfs,Attempt:2,},Image:&ImageSpec{Image:sha256:e193d2eaa1633ba12a6c0111c9a97566e07e4c98a0c3e5b794fa78858fd474d5,Annotations:map[string]string{},},ImageRef:sha256:e193d2eaa1633ba12a6c0111c9a97566e07e4c98a0c3e5b794fa78858fd474d5,State:CONTAINER_EXITED,CreatedAt:1640570076058729754,Labels:map[string]string{io.kubernetes.container.name: nfs,io.kubernetes.pod.name: csi-nfs-controller-d775b4db9-6kj9p,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 7f344931-7918-40df-a833-0eaa83d1e412,},Annotations:map[string]string{io.kubernetes.container.hash: 3a748bea,io.kubernetes.container.ports: [{"name":"healthz","hostPort":29652,"containerPort":29652,"protocol":"TCP"}],io.kubernetes.container.restartCount: 2,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:0b0ab98174cae147a5824a61ca22b3ed981a813ce602a599825a002825519c8a,PodSandboxId:2162817c2f36c0a60b71e3de5d8a32cf971691280167860b6e8925cfc74c9aad,Metadata:&ContainerMetadata{Name:liveness-probe,Attempt:0,},Image:&ImageSpec{Image:sha256:a347a725763c086b9dc98abda1cdaae8c9f6c62bd10e8db33a2205919a9cf8ed,Annotations:map[string]string{},},ImageRef:sha256:a347a725763c086b9dc98abda1cdaae8c9f6c62bd10e8db33a2205919a9cf8ed,State:CONTAINER_RUNNING,CreatedAt:1640570263501601096,Labels:map[string]string{io.kubernetes.container.name: liveness-probe,io.kubernetes.pod.name: csi-nfs-controller-d775b4db9-6kj9p,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 7f344931-7918-40df-a833-0eaa83d1e412,},Annotations:map[string]string{io.kubernetes.container.hash: 51a44179,io.kubernetes.container.restartCount: 0,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},}]
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.297209     248 factory.go:216] Using factory "containerd" for container "/system.slice/docker-ebade756ae235ae7e656d8f79c95f3c11b9b734c2c0b8ae1d3ad714fc915481e.scope/kubelet/kubepods/besteffort/podb9ba097d-5dff-4c9d-ab9e-8287801185d5/0d6e2be2d4d6fb6a9b4f68c6ea1b0b28088f1d88e3e1056834ca69a406ab8014"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.365494     248 config.go:97] "Looking for sources, have seen" sources=[api file] seenSources=map[api:{} file:{}]
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.365597     248 kubelet.go:2141] "SyncLoop (housekeeping)"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.369769     248 remote_runtime.go:127] "[RemoteRuntimeService] RunPodSandbox Response" podSandboxID="0d6e2be2d4d6fb6a9b4f68c6ea1b0b28088f1d88e3e1056834ca69a406ab8014"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.369839     248 kuberuntime_manager.go:823] "Created PodSandbox for pod" podSandboxID="0d6e2be2d4d6fb6a9b4f68c6ea1b0b28088f1d88e3e1056834ca69a406ab8014" pod="default/pod2"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.369858     248 remote_runtime.go:175] "[RemoteRuntimeService] PodSandboxStatus" podSandboxID="0d6e2be2d4d6fb6a9b4f68c6ea1b0b28088f1d88e3e1056834ca69a406ab8014" timeout="2m0s"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.370467     248 remote_runtime.go:186] "[RemoteRuntimeService] PodSandboxStatus Response" podSandboxID="0d6e2be2d4d6fb6a9b4f68c6ea1b0b28088f1d88e3e1056834ca69a406ab8014" status="&PodSandboxStatus{Id:0d6e2be2d4d6fb6a9b4f68c6ea1b0b28088f1d88e3e1056834ca69a406ab8014,Metadata:&PodSandboxMetadata{Name:pod2,Uid:b9ba097d-5dff-4c9d-ab9e-8287801185d5,Namespace:default,Attempt:0,},State:SANDBOX_READY,CreatedAt:1640657864247120195,Network:&PodSandboxNetworkStatus{Ip:10.244.2.2,AdditionalIps:[]*PodIP{},},Linux:&LinuxPodSandboxStatus{Namespaces:&Namespace{Options:&NamespaceOption{Network:POD,Pid:CONTAINER,Ipc:POD,TargetId:,},},},Labels:map[string]string{io.kubernetes.pod.name: pod2,io.kubernetes.pod.namespace: default,io.kubernetes.pod.uid: b9ba097d-5dff-4c9d-ab9e-8287801185d5,run: pod2,},Annotations:map[string]string{kubectl.kubernetes.io/last-applied-configuration: {\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"run\":\"pod2\"},\"name\":\"pod2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"sleep\",\"infinity\"],\"image\":\"centos:7\",\"name\":\"pod2\",\"volumeMounts\":[{\"mountPath\":\"/mnt/data\",\"name\":\"myvolume\"}]}],\"nodeSelector\":{\"kubernetes.io/hostname\":\"kind-worker2\"},\"volumes\":[{\"name\":\"myvolume\",\"persistentVolumeClaim\":{\"claimName\":\"pvc1\"}}]}}\n,kubernetes.io/config.seen: 2021-12-28T02:17:43.793731871Z,kubernetes.io/config.source: api,},RuntimeHandler:,}"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.370526     248 kuberuntime_manager.go:842] "Determined the ip for pod after sandbox changed" IPs=[10.244.2.2] pod="default/pod2"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.370679     248 kuberuntime_manager.go:882] "Creating container in pod" containerType="container" container="&Container{Name:pod2,Image:centos:7,Command:[sleep infinity],Args:[],WorkingDir:,Ports:[]ContainerPort{},Env:[]EnvVar{},Resources:ResourceRequirements{Limits:ResourceList{},Requests:ResourceList{},},VolumeMounts:[]VolumeMount{VolumeMount{Name:myvolume,ReadOnly:false,MountPath:/mnt/data,SubPath:,MountPropagation:nil,SubPathExpr:,},VolumeMount{Name:kube-api-access-vq2z9,ReadOnly:true,MountPath:/var/run/secrets/kubernetes.io/serviceaccount,SubPath:,MountPropagation:nil,SubPathExpr:,},},LivenessProbe:nil,ReadinessProbe:nil,Lifecycle:nil,TerminationMessagePath:/dev/termination-log,ImagePullPolicy:IfNotPresent,SecurityContext:nil,Stdin:false,StdinOnce:false,TTY:false,EnvFrom:[]EnvFromSource{},TerminationMessagePolicy:File,VolumeDevices:[]VolumeDevice{},StartupProbe:nil,}" pod="default/pod2"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.371335     248 event.go:291] "Event occurred" object="default/pod2" kind="Pod" apiVersion="v1" type="Normal" reason="Pulling" message="Pulling image \"centos:7\""
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.371376     248 request.go:1179] Request Body:
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.371507     248 round_trippers.go:435] curl -v -XPOST  -H "Accept: application/vnd.kubernetes.protobuf,application/json" -H "Content-Type: application/vnd.kubernetes.protobuf" -H "User-Agent: kubelet/v1.22.0 (linux/amd64) kubernetes/c2b5237" 'https://kind-control-plane:6443/api/v1/namespaces/default/events'
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.373013     248 kubelet_pods.go:1046] "Clean up pod workers for terminated pods"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.373042     248 kubelet_pods.go:1072] "Clean up probes for terminating and terminated pods"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.373070     248 remote_runtime.go:199] "[RemoteRuntimeService] ListPodSandbox" filter="&PodSandboxFilter{Id:,State:&PodSandboxStateValue{State:SANDBOX_READY,},LabelSelector:map[string]string{},}" timeout="2m0s"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.373593     248 remote_runtime.go:211] "[RemoteRuntimeService] ListPodSandbox Response" filter="&PodSandboxFilter{Id:,State:&PodSandboxStateValue{State:SANDBOX_READY,},LabelSelector:map[string]string{},}" items=[&PodSandbox{Id:b81eb4cfa1555e549df9678a54141707d464354d0799acdf54330b79adafa3a3,Metadata:&PodSandboxMetadata{Name:csi-nfs-node-hs4q9,Uid:78e34d6c-417b-408b-bf2a-12ceb26bb982,Namespace:kube-system,Attempt:0,},State:SANDBOX_READY,CreatedAt:1640569716385141797,Labels:map[string]string{app: csi-nfs-node,controller-revision-hash: d5b5cc545,io.kubernetes.pod.name: csi-nfs-node-hs4q9,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 78e34d6c-417b-408b-bf2a-12ceb26bb982,pod-template-generation: 1,},Annotations:map[string]string{kubernetes.io/config.seen: 2021-12-27T01:48:35.767112649Z,kubernetes.io/config.source: api,},RuntimeHandler:,} &PodSandbox{Id:0d6e2be2d4d6fb6a9b4f68c6ea1b0b28088f1d88e3e1056834ca69a406ab8014,Metadata:&PodSandboxMetadata{Name:pod2,Uid:b9ba097d-5dff-4c9d-ab9e-8287801185d5,Namespace:default,Attempt:0,},State:SANDBOX_READY,CreatedAt:1640657864247120195,Labels:map[string]string{io.kubernetes.pod.name: pod2,io.kubernetes.pod.namespace: default,io.kubernetes.pod.uid: b9ba097d-5dff-4c9d-ab9e-8287801185d5,run: pod2,},Annotations:map[string]string{kubectl.kubernetes.io/last-applied-configuration: {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"labels":{"run":"pod2"},"name":"pod2","namespace":"default"},"spec":{"containers":[{"command":["sleep","infinity"],"image":"centos:7","name":"pod2","volumeMounts":[{"mountPath":"/mnt/data","name":"myvolume"}]}],"nodeSelector":{"kubernetes.io/hostname":"kind-worker2"},"volumes":[{"name":"myvolume","persistentVolumeClaim":{"claimName":"pvc1"}}]}}
Dec 28 02:17:44 kind-worker2 kubelet[248]: ,kubernetes.io/config.seen: 2021-12-28T02:17:43.793731871Z,kubernetes.io/config.source: api,},RuntimeHandler:,} &PodSandbox{Id:8aa1fc56e1b975e136fcc911208ce136f7836ac2afc770b3df1e4e82d0ca4cc9,Metadata:&PodSandboxMetadata{Name:kube-proxy-4kxp2,Uid:f7fd2241-6d84-45fe-bb45-5e4ff7c3dc90,Namespace:kube-system,Attempt:0,},State:SANDBOX_READY,CreatedAt:1640567744146327483,Labels:map[string]string{controller-revision-hash: 7c76476f94,io.kubernetes.pod.name: kube-proxy-4kxp2,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: f7fd2241-6d84-45fe-bb45-5e4ff7c3dc90,k8s-app: kube-proxy,pod-template-generation: 1,},Annotations:map[string]string{kubernetes.io/config.seen: 2021-12-27T01:15:42.145279082Z,kubernetes.io/config.source: api,},RuntimeHandler:,} &PodSandbox{Id:6c9e2abee1d3cc91a4b4236373965bc5b2910971019c150c1de6eb6e8127975c,Metadata:&PodSandboxMetadata{Name:kindnet-kcz76,Uid:4d6edc85-1e87-49c6-ae52-7528ef6d853a,Namespace:kube-system,Attempt:0,},State:SANDBOX_READY,CreatedAt:1640567744068106433,Labels:map[string]string{app: kindnet,controller-revision-hash: 6445d85c5c,io.kubernetes.pod.name: kindnet-kcz76,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 4d6edc85-1e87-49c6-ae52-7528ef6d853a,k8s-app: kindnet,pod-template-generation: 1,tier: node,},Annotations:map[string]string{kubernetes.io/config.seen: 2021-12-27T01:15:42.145252924Z,kubernetes.io/config.source: api,},RuntimeHandler:,} &PodSandbox{Id:2162817c2f36c0a60b71e3de5d8a32cf971691280167860b6e8925cfc74c9aad,Metadata:&PodSandboxMetadata{Name:csi-nfs-controller-d775b4db9-6kj9p,Uid:7f344931-7918-40df-a833-0eaa83d1e412,Namespace:kube-system,Attempt:0,},State:SANDBOX_READY,CreatedAt:1640569715859038684,Labels:map[string]string{app: csi-nfs-controller,io.kubernetes.pod.name: csi-nfs-controller-d775b4db9-6kj9p,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 7f344931-7918-40df-a833-0eaa83d1e412,pod-template-hash: d775b4db9,},Annotations:map[string]string{kubernetes.io/config.seen: 2021-12-27T01:48:35.522768731Z,kubernetes.io/config.source: api,},RuntimeHandler:,}]
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.373646     248 remote_runtime.go:306] "[RemoteRuntimeService] ListContainers" filter="&ContainerFilter{Id:,State:&ContainerStateValue{State:CONTAINER_RUNNING,},PodSandboxId:,LabelSelector:map[string]string{},}" timeout="2m0s"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.374175     248 remote_runtime.go:317] "[RemoteRuntimeService] ListContainers Response" filter="&ContainerFilter{Id:,State:&ContainerStateValue{State:CONTAINER_RUNNING,},PodSandboxId:,LabelSelector:map[string]string{},}" containers=[&Container{Id:92ec9f558b6ee6189686322bfe9519daed872fa3102006f0dacb0bec97a4a33b,PodSandboxId:8aa1fc56e1b975e136fcc911208ce136f7836ac2afc770b3df1e4e82d0ca4cc9,Metadata:&ContainerMetadata{Name:kube-proxy,Attempt:0,},Image:&ImageSpec{Image:sha256:d34e62e43dded439d130408f771232696905ff6b6a900027c289ee55b4df605e,Annotations:map[string]string{},},ImageRef:sha256:d34e62e43dded439d130408f771232696905ff6b6a900027c289ee55b4df605e,State:CONTAINER_RUNNING,CreatedAt:1640567751617687909,Labels:map[string]string{io.kubernetes.container.name: kube-proxy,io.kubernetes.pod.name: kube-proxy-4kxp2,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: f7fd2241-6d84-45fe-bb45-5e4ff7c3dc90,},Annotations:map[string]string{io.kubernetes.container.hash: 123908ae,io.kubernetes.container.restartCount: 0,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:ae4e5a6803a4a9a2d5808c0418b52bdca909da3cc06d9ee872bcdf9acfc4ed65,PodSandboxId:b81eb4cfa1555e549df9678a54141707d464354d0799acdf54330b79adafa3a3,Metadata:&ContainerMetadata{Name:nfs,Attempt:3,},Image:&ImageSpec{Image:sha256:e193d2eaa1633ba12a6c0111c9a97566e07e4c98a0c3e5b794fa78858fd474d5,Annotations:map[string]string{},},ImageRef:sha256:e193d2eaa1633ba12a6c0111c9a97566e07e4c98a0c3e5b794fa78858fd474d5,State:CONTAINER_RUNNING,CreatedAt:1640570256874312561,Labels:map[string]string{io.kubernetes.container.name: nfs,io.kubernetes.pod.name: csi-nfs-node-hs4q9,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 78e34d6c-417b-408b-bf2a-12ceb26bb982,},Annotations:map[string]string{io.kubernetes.container.hash: f82f63bc,io.kubernetes.container.ports: [{"name":"healthz","hostPort":29653,"containerPort":29653,"protocol":"TCP"}],io.kubernetes.container.restartCount: 3,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:e01eecf7217dba4707fa1cdbd0b3725d0d3ee4a86146f7b44fad4a3fe8ad18ab,PodSandboxId:2162817c2f36c0a60b71e3de5d8a32cf971691280167860b6e8925cfc74c9aad,Metadata:&ContainerMetadata{Name:csi-provisioner,Attempt:1,},Image:&ImageSpec{Image:sha256:e18077242e6d7990e7f18bf5dfb2184c2d57124ae62b6ba2764b71741ed2f4c2,Annotations:map[string]string{},},ImageRef:sha256:e18077242e6d7990e7f18bf5dfb2184c2d57124ae62b6ba2764b71741ed2f4c2,State:CONTAINER_RUNNING,CreatedAt:1640570258701719590,Labels:map[string]string{io.kubernetes.container.name: csi-provisioner,io.kubernetes.pod.name: csi-nfs-controller-d775b4db9-6kj9p,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 7f344931-7918-40df-a833-0eaa83d1e412,},Annotations:map[string]string{io.kubernetes.container.hash: 29d2c2b3,io.kubernetes.container.restartCount: 1,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:2890ad62c49835f8b455ed03e434f7a39dcd2adeeb1afdcf173faa1d239a0674,PodSandboxId:b81eb4cfa1555e549df9678a54141707d464354d0799acdf54330b79adafa3a3,Metadata:&ContainerMetadata{Name:liveness-probe,Attempt:0,},Image:&ImageSpec{Image:sha256:a347a725763c086b9dc98abda1cdaae8c9f6c62bd10e8db33a2205919a9cf8ed,Annotations:map[string]string{},},ImageRef:sha256:a347a725763c086b9dc98abda1cdaae8c9f6c62bd10e8db33a2205919a9cf8ed,State:CONTAINER_RUNNING,CreatedAt:1640570273640601518,Labels:map[string]string{io.kubernetes.container.name: liveness-probe,io.kubernetes.pod.name: csi-nfs-node-hs4q9,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 78e34d6c-417b-408b-bf2a-12ceb26bb982,},Annotations:map[string]string{io.kubernetes.container.hash: 661a0ca1,io.kubernetes.container.restartCount: 0,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:91236bbddf09a83b23b4c50a821c727c10f770ae272e403ea68a111a07c12c26,PodSandboxId:b81eb4cfa1555e549df9678a54141707d464354d0799acdf54330b79adafa3a3,Metadata:&ContainerMetadata{Name:node-driver-registrar,Attempt:0,},Image:&ImageSpec{Image:sha256:f45c8a305a0bb15ff256a32686d56356be69e1b8d469e90a247d279ad6702382,Annotations:map[string]string{},},ImageRef:sha256:f45c8a305a0bb15ff256a32686d56356be69e1b8d469e90a247d279ad6702382,State:CONTAINER_RUNNING,CreatedAt:1640570316368173914,Labels:map[string]string{io.kubernetes.container.name: node-driver-registrar,io.kubernetes.pod.name: csi-nfs-node-hs4q9,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 78e34d6c-417b-408b-bf2a-12ceb26bb982,},Annotations:map[string]string{io.kubernetes.container.hash: 385cd5c0,io.kubernetes.container.restartCount: 0,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:815e473c76e85a6b98e06040c2f33c314c9a3c18bbaa2632312f9237cfecde94,PodSandboxId:6c9e2abee1d3cc91a4b4236373965bc5b2910971019c150c1de6eb6e8127975c,Metadata:&ContainerMetadata{Name:kindnet-cni,Attempt:0,},Image:&ImageSpec{Image:sha256:3105b3ff910efceee092aa79ef80f8cbc8a776b7c02325b586c466a6079c0ccf,Annotations:map[string]string{},},ImageRef:sha256:3105b3ff910efceee092aa79ef80f8cbc8a776b7c02325b586c466a6079c0ccf,State:CONTAINER_RUNNING,CreatedAt:1640567751616629285,Labels:map[string]string{io.kubernetes.container.name: kindnet-cni,io.kubernetes.pod.name: kindnet-kcz76,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 4d6edc85-1e87-49c6-ae52-7528ef6d853a,},Annotations:map[string]string{io.kubernetes.container.hash: 34ee6d28,io.kubernetes.container.restartCount: 0,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:3e03cc01f742f797920859ee23676a239ffcadec1008456fb929bd7e74f78c4a,PodSandboxId:2162817c2f36c0a60b71e3de5d8a32cf971691280167860b6e8925cfc74c9aad,Metadata:&ContainerMetadata{Name:nfs,Attempt:3,},Image:&ImageSpec{Image:sha256:e193d2eaa1633ba12a6c0111c9a97566e07e4c98a0c3e5b794fa78858fd474d5,Annotations:map[string]string{},},ImageRef:sha256:e193d2eaa1633ba12a6c0111c9a97566e07e4c98a0c3e5b794fa78858fd474d5,State:CONTAINER_RUNNING,CreatedAt:1640570256641136071,Labels:map[string]string{io.kubernetes.container.name: nfs,io.kubernetes.pod.name: csi-nfs-controller-d775b4db9-6kj9p,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 7f344931-7918-40df-a833-0eaa83d1e412,},Annotations:map[string]string{io.kubernetes.container.hash: 3a748bea,io.kubernetes.container.ports: [{"name":"healthz","hostPort":29652,"containerPort":29652,"protocol":"TCP"}],io.kubernetes.container.restartCount: 3,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:0b0ab98174cae147a5824a61ca22b3ed981a813ce602a599825a002825519c8a,PodSandboxId:2162817c2f36c0a60b71e3de5d8a32cf971691280167860b6e8925cfc74c9aad,Metadata:&ContainerMetadata{Name:liveness-probe,Attempt:0,},Image:&ImageSpec{Image:sha256:a347a725763c086b9dc98abda1cdaae8c9f6c62bd10e8db33a2205919a9cf8ed,Annotations:map[string]string{},},ImageRef:sha256:a347a725763c086b9dc98abda1cdaae8c9f6c62bd10e8db33a2205919a9cf8ed,State:CONTAINER_RUNNING,CreatedAt:1640570263501601096,Labels:map[string]string{io.kubernetes.container.name: liveness-probe,io.kubernetes.pod.name: csi-nfs-controller-d775b4db9-6kj9p,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 7f344931-7918-40df-a833-0eaa83d1e412,},Annotations:map[string]string{io.kubernetes.container.hash: 51a44179,io.kubernetes.container.restartCount: 0,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},}]
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.374303     248 kubelet_pods.go:1109] "Clean up orphaned pod statuses"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.374340     248 remote_runtime.go:199] "[RemoteRuntimeService] ListPodSandbox" filter="&PodSandboxFilter{Id:,State:&PodSandboxStateValue{State:SANDBOX_READY,},LabelSelector:map[string]string{},}" timeout="2m0s"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.374759     248 remote_runtime.go:211] "[RemoteRuntimeService] ListPodSandbox Response" filter="&PodSandboxFilter{Id:,State:&PodSandboxStateValue{State:SANDBOX_READY,},LabelSelector:map[string]string{},}" items=[&PodSandbox{Id:0d6e2be2d4d6fb6a9b4f68c6ea1b0b28088f1d88e3e1056834ca69a406ab8014,Metadata:&PodSandboxMetadata{Name:pod2,Uid:b9ba097d-5dff-4c9d-ab9e-8287801185d5,Namespace:default,Attempt:0,},State:SANDBOX_READY,CreatedAt:1640657864247120195,Labels:map[string]string{io.kubernetes.pod.name: pod2,io.kubernetes.pod.namespace: default,io.kubernetes.pod.uid: b9ba097d-5dff-4c9d-ab9e-8287801185d5,run: pod2,},Annotations:map[string]string{kubectl.kubernetes.io/last-applied-configuration: {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"labels":{"run":"pod2"},"name":"pod2","namespace":"default"},"spec":{"containers":[{"command":["sleep","infinity"],"image":"centos:7","name":"pod2","volumeMounts":[{"mountPath":"/mnt/data","name":"myvolume"}]}],"nodeSelector":{"kubernetes.io/hostname":"kind-worker2"},"volumes":[{"name":"myvolume","persistentVolumeClaim":{"claimName":"pvc1"}}]}}
Dec 28 02:17:44 kind-worker2 kubelet[248]: ,kubernetes.io/config.seen: 2021-12-28T02:17:43.793731871Z,kubernetes.io/config.source: api,},RuntimeHandler:,} &PodSandbox{Id:8aa1fc56e1b975e136fcc911208ce136f7836ac2afc770b3df1e4e82d0ca4cc9,Metadata:&PodSandboxMetadata{Name:kube-proxy-4kxp2,Uid:f7fd2241-6d84-45fe-bb45-5e4ff7c3dc90,Namespace:kube-system,Attempt:0,},State:SANDBOX_READY,CreatedAt:1640567744146327483,Labels:map[string]string{controller-revision-hash: 7c76476f94,io.kubernetes.pod.name: kube-proxy-4kxp2,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: f7fd2241-6d84-45fe-bb45-5e4ff7c3dc90,k8s-app: kube-proxy,pod-template-generation: 1,},Annotations:map[string]string{kubernetes.io/config.seen: 2021-12-27T01:15:42.145279082Z,kubernetes.io/config.source: api,},RuntimeHandler:,} &PodSandbox{Id:6c9e2abee1d3cc91a4b4236373965bc5b2910971019c150c1de6eb6e8127975c,Metadata:&PodSandboxMetadata{Name:kindnet-kcz76,Uid:4d6edc85-1e87-49c6-ae52-7528ef6d853a,Namespace:kube-system,Attempt:0,},State:SANDBOX_READY,CreatedAt:1640567744068106433,Labels:map[string]string{app: kindnet,controller-revision-hash: 6445d85c5c,io.kubernetes.pod.name: kindnet-kcz76,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 4d6edc85-1e87-49c6-ae52-7528ef6d853a,k8s-app: kindnet,pod-template-generation: 1,tier: node,},Annotations:map[string]string{kubernetes.io/config.seen: 2021-12-27T01:15:42.145252924Z,kubernetes.io/config.source: api,},RuntimeHandler:,} &PodSandbox{Id:2162817c2f36c0a60b71e3de5d8a32cf971691280167860b6e8925cfc74c9aad,Metadata:&PodSandboxMetadata{Name:csi-nfs-controller-d775b4db9-6kj9p,Uid:7f344931-7918-40df-a833-0eaa83d1e412,Namespace:kube-system,Attempt:0,},State:SANDBOX_READY,CreatedAt:1640569715859038684,Labels:map[string]string{app: csi-nfs-controller,io.kubernetes.pod.name: csi-nfs-controller-d775b4db9-6kj9p,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 7f344931-7918-40df-a833-0eaa83d1e412,pod-template-hash: d775b4db9,},Annotations:map[string]string{kubernetes.io/config.seen: 2021-12-27T01:48:35.522768731Z,kubernetes.io/config.source: api,},RuntimeHandler:,} &PodSandbox{Id:b81eb4cfa1555e549df9678a54141707d464354d0799acdf54330b79adafa3a3,Metadata:&PodSandboxMetadata{Name:csi-nfs-node-hs4q9,Uid:78e34d6c-417b-408b-bf2a-12ceb26bb982,Namespace:kube-system,Attempt:0,},State:SANDBOX_READY,CreatedAt:1640569716385141797,Labels:map[string]string{app: csi-nfs-node,controller-revision-hash: d5b5cc545,io.kubernetes.pod.name: csi-nfs-node-hs4q9,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 78e34d6c-417b-408b-bf2a-12ceb26bb982,pod-template-generation: 1,},Annotations:map[string]string{kubernetes.io/config.seen: 2021-12-27T01:48:35.767112649Z,kubernetes.io/config.source: api,},RuntimeHandler:,}]
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.374799     248 remote_runtime.go:306] "[RemoteRuntimeService] ListContainers" filter="&ContainerFilter{Id:,State:&ContainerStateValue{State:CONTAINER_RUNNING,},PodSandboxId:,LabelSelector:map[string]string{},}" timeout="2m0s"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.375684     248 remote_runtime.go:317] "[RemoteRuntimeService] ListContainers Response" filter="&ContainerFilter{Id:,State:&ContainerStateValue{State:CONTAINER_RUNNING,},PodSandboxId:,LabelSelector:map[string]string{},}" containers=[&Container{Id:ae4e5a6803a4a9a2d5808c0418b52bdca909da3cc06d9ee872bcdf9acfc4ed65,PodSandboxId:b81eb4cfa1555e549df9678a54141707d464354d0799acdf54330b79adafa3a3,Metadata:&ContainerMetadata{Name:nfs,Attempt:3,},Image:&ImageSpec{Image:sha256:e193d2eaa1633ba12a6c0111c9a97566e07e4c98a0c3e5b794fa78858fd474d5,Annotations:map[string]string{},},ImageRef:sha256:e193d2eaa1633ba12a6c0111c9a97566e07e4c98a0c3e5b794fa78858fd474d5,State:CONTAINER_RUNNING,CreatedAt:1640570256874312561,Labels:map[string]string{io.kubernetes.container.name: nfs,io.kubernetes.pod.name: csi-nfs-node-hs4q9,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 78e34d6c-417b-408b-bf2a-12ceb26bb982,},Annotations:map[string]string{io.kubernetes.container.hash: f82f63bc,io.kubernetes.container.ports: [{"name":"healthz","hostPort":29653,"containerPort":29653,"protocol":"TCP"}],io.kubernetes.container.restartCount: 3,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:e01eecf7217dba4707fa1cdbd0b3725d0d3ee4a86146f7b44fad4a3fe8ad18ab,PodSandboxId:2162817c2f36c0a60b71e3de5d8a32cf971691280167860b6e8925cfc74c9aad,Metadata:&ContainerMetadata{Name:csi-provisioner,Attempt:1,},Image:&ImageSpec{Image:sha256:e18077242e6d7990e7f18bf5dfb2184c2d57124ae62b6ba2764b71741ed2f4c2,Annotations:map[string]string{},},ImageRef:sha256:e18077242e6d7990e7f18bf5dfb2184c2d57124ae62b6ba2764b71741ed2f4c2,State:CONTAINER_RUNNING,CreatedAt:1640570258701719590,Labels:map[string]string{io.kubernetes.container.name: csi-provisioner,io.kubernetes.pod.name: csi-nfs-controller-d775b4db9-6kj9p,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 7f344931-7918-40df-a833-0eaa83d1e412,},Annotations:map[string]string{io.kubernetes.container.hash: 29d2c2b3,io.kubernetes.container.restartCount: 1,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:2890ad62c49835f8b455ed03e434f7a39dcd2adeeb1afdcf173faa1d239a0674,PodSandboxId:b81eb4cfa1555e549df9678a54141707d464354d0799acdf54330b79adafa3a3,Metadata:&ContainerMetadata{Name:liveness-probe,Attempt:0,},Image:&ImageSpec{Image:sha256:a347a725763c086b9dc98abda1cdaae8c9f6c62bd10e8db33a2205919a9cf8ed,Annotations:map[string]string{},},ImageRef:sha256:a347a725763c086b9dc98abda1cdaae8c9f6c62bd10e8db33a2205919a9cf8ed,State:CONTAINER_RUNNING,CreatedAt:1640570273640601518,Labels:map[string]string{io.kubernetes.container.name: liveness-probe,io.kubernetes.pod.name: csi-nfs-node-hs4q9,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 78e34d6c-417b-408b-bf2a-12ceb26bb982,},Annotations:map[string]string{io.kubernetes.container.hash: 661a0ca1,io.kubernetes.container.restartCount: 0,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:92ec9f558b6ee6189686322bfe9519daed872fa3102006f0dacb0bec97a4a33b,PodSandboxId:8aa1fc56e1b975e136fcc911208ce136f7836ac2afc770b3df1e4e82d0ca4cc9,Metadata:&ContainerMetadata{Name:kube-proxy,Attempt:0,},Image:&ImageSpec{Image:sha256:d34e62e43dded439d130408f771232696905ff6b6a900027c289ee55b4df605e,Annotations:map[string]string{},},ImageRef:sha256:d34e62e43dded439d130408f771232696905ff6b6a900027c289ee55b4df605e,State:CONTAINER_RUNNING,CreatedAt:1640567751617687909,Labels:map[string]string{io.kubernetes.container.name: kube-proxy,io.kubernetes.pod.name: kube-proxy-4kxp2,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: f7fd2241-6d84-45fe-bb45-5e4ff7c3dc90,},Annotations:map[string]string{io.kubernetes.container.hash: 123908ae,io.kubernetes.container.restartCount: 0,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:3e03cc01f742f797920859ee23676a239ffcadec1008456fb929bd7e74f78c4a,PodSandboxId:2162817c2f36c0a60b71e3de5d8a32cf971691280167860b6e8925cfc74c9aad,Metadata:&ContainerMetadata{Name:nfs,Attempt:3,},Image:&ImageSpec{Image:sha256:e193d2eaa1633ba12a6c0111c9a97566e07e4c98a0c3e5b794fa78858fd474d5,Annotations:map[string]string{},},ImageRef:sha256:e193d2eaa1633ba12a6c0111c9a97566e07e4c98a0c3e5b794fa78858fd474d5,State:CONTAINER_RUNNING,CreatedAt:1640570256641136071,Labels:map[string]string{io.kubernetes.container.name: nfs,io.kubernetes.pod.name: csi-nfs-controller-d775b4db9-6kj9p,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 7f344931-7918-40df-a833-0eaa83d1e412,},Annotations:map[string]string{io.kubernetes.container.hash: 3a748bea,io.kubernetes.container.ports: [{"name":"healthz","hostPort":29652,"containerPort":29652,"protocol":"TCP"}],io.kubernetes.container.restartCount: 3,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:0b0ab98174cae147a5824a61ca22b3ed981a813ce602a599825a002825519c8a,PodSandboxId:2162817c2f36c0a60b71e3de5d8a32cf971691280167860b6e8925cfc74c9aad,Metadata:&ContainerMetadata{Name:liveness-probe,Attempt:0,},Image:&ImageSpec{Image:sha256:a347a725763c086b9dc98abda1cdaae8c9f6c62bd10e8db33a2205919a9cf8ed,Annotations:map[string]string{},},ImageRef:sha256:a347a725763c086b9dc98abda1cdaae8c9f6c62bd10e8db33a2205919a9cf8ed,State:CONTAINER_RUNNING,CreatedAt:1640570263501601096,Labels:map[string]string{io.kubernetes.container.name: liveness-probe,io.kubernetes.pod.name: csi-nfs-controller-d775b4db9-6kj9p,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 7f344931-7918-40df-a833-0eaa83d1e412,},Annotations:map[string]string{io.kubernetes.container.hash: 51a44179,io.kubernetes.container.restartCount: 0,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:91236bbddf09a83b23b4c50a821c727c10f770ae272e403ea68a111a07c12c26,PodSandboxId:b81eb4cfa1555e549df9678a54141707d464354d0799acdf54330b79adafa3a3,Metadata:&ContainerMetadata{Name:node-driver-registrar,Attempt:0,},Image:&ImageSpec{Image:sha256:f45c8a305a0bb15ff256a32686d56356be69e1b8d469e90a247d279ad6702382,Annotations:map[string]string{},},ImageRef:sha256:f45c8a305a0bb15ff256a32686d56356be69e1b8d469e90a247d279ad6702382,State:CONTAINER_RUNNING,CreatedAt:1640570316368173914,Labels:map[string]string{io.kubernetes.container.name: node-driver-registrar,io.kubernetes.pod.name: csi-nfs-node-hs4q9,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 78e34d6c-417b-408b-bf2a-12ceb26bb982,},Annotations:map[string]string{io.kubernetes.container.hash: 385cd5c0,io.kubernetes.container.restartCount: 0,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},} &Container{Id:815e473c76e85a6b98e06040c2f33c314c9a3c18bbaa2632312f9237cfecde94,PodSandboxId:6c9e2abee1d3cc91a4b4236373965bc5b2910971019c150c1de6eb6e8127975c,Metadata:&ContainerMetadata{Name:kindnet-cni,Attempt:0,},Image:&ImageSpec{Image:sha256:3105b3ff910efceee092aa79ef80f8cbc8a776b7c02325b586c466a6079c0ccf,Annotations:map[string]string{},},ImageRef:sha256:3105b3ff910efceee092aa79ef80f8cbc8a776b7c02325b586c466a6079c0ccf,State:CONTAINER_RUNNING,CreatedAt:1640567751616629285,Labels:map[string]string{io.kubernetes.container.name: kindnet-cni,io.kubernetes.pod.name: kindnet-kcz76,io.kubernetes.pod.namespace: kube-system,io.kubernetes.pod.uid: 4d6edc85-1e87-49c6-ae52-7528ef6d853a,},Annotations:map[string]string{io.kubernetes.container.hash: 34ee6d28,io.kubernetes.container.restartCount: 0,io.kubernetes.container.terminationMessagePath: /dev/termination-log,io.kubernetes.container.terminationMessagePolicy: File,io.kubernetes.pod.terminationGracePeriod: 30,},}]
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.375752     248 kubelet_pods.go:1128] "Clean up orphaned pod directories"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.375836     248 kubelet_pods.go:1139] "Clean up orphaned mirror pods"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.375848     248 kubelet_pods.go:1146] "Clean up orphaned pod cgroups"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.375902     248 kubelet.go:2149] "SyncLoop (housekeeping) end"
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.381250     248 round_trippers.go:454] POST https://kind-control-plane:6443/api/v1/namespaces/default/events 201 Created in 9 milliseconds
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.381265     248 round_trippers.go:460] Response Headers:
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.381272     248 round_trippers.go:463]     X-Kubernetes-Pf-Flowschema-Uid: f969d278-8444-4416-9dd5-1502f71cced1
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.381277     248 round_trippers.go:463]     X-Kubernetes-Pf-Prioritylevel-Uid: 31aad832-271e-4b47-939b-f576268dbc44
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.381280     248 round_trippers.go:463]     Content-Length: 531
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.381284     248 round_trippers.go:463]     Date: Tue, 28 Dec 2021 02:17:44 GMT
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.381287     248 round_trippers.go:463]     Audit-Id: ab060e7f-8b6d-48f0-afab-e014e41adf43
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.381291     248 round_trippers.go:463]     Cache-Control: no-cache, private
Dec 28 02:17:44 kind-worker2 kubelet[248]: I1228 02:17:44.381295     248 round_trippers.go:463]     Content-Type: application/vnd.kubernetes.protobuf








