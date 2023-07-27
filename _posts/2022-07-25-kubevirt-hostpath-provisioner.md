---
layout: post
title: Hostpath provisioner for kubevirt
tags: kubernetes
---

The below assumes kubevirt is up and running on a k8s cluster. 


### Create hostpath provisioner for CDI uploads 
```
$ kubectl create -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.1/cert-manager.yaml
$ kubectl wait --for=condition=Available -n cert-manager --timeout=120s --all deployments
$ kubectl create -f https://raw.githubusercontent.com/kubevirt/hostpath-provisioner-operator/main/deploy/namespace.yaml
$ kubectl create -f https://raw.githubusercontent.com/kubevirt/hostpath-provisioner-operator/main/deploy/webhook.yaml -n hostpath-provisioner
$ kubectl create -f https://raw.githubusercontent.com/kubevirt/hostpath-provisioner-operator/main/deploy/operator.yaml -n hostpath-provisioner
```

### Verify
```
root@k8s-master:/home/aprabh/kubevirt-manifests/ubuntu_vm/CDI/vm1# kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5b6d4f8d44-ttgq5              1/1     Running   0          35m
cert-manager-cainjector-747cfdfd87-hdwcl   1/1     Running   0          35m
cert-manager-webhook-67cb765ff6-9cnpl      1/1     Running   0          35m
root@k8s-master:/home/aprabh/kubevirt-manifests/ubuntu_vm/CDI/vm1# kubectl get pods -n hostpath-provisioner
NAME                                             READY   STATUS    RESTARTS   AGE
hostpath-provisioner-csi-kvxtl                   4/4     Running   0          30m
hostpath-provisioner-operator-86bf8c88b6-wg8fm   1/1     Running   0          35m
```

### Create Storage pool

Save to file storage-pool.yaml
```
apiVersion: hostpathprovisioner.kubevirt.io/v1beta1
kind: HostPathProvisioner
metadata:
  name: hostpath-provisioner
spec:
  imagePullPolicy: Always
  storagePools:
    - name: "local"
      path: "/var/hpvolumes"
  workload:
    nodeSelector:
      kubernetes.io/hostname: k8s-worker2
```

### Create Storage profile 
save to file storage-provisioner.yaml
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hostpath-csi
provisioner: kubevirt.io.hostpath-provisioner
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
parameters:
  storagePool: local
```

### Apply the above
```
kubectl apply -f storage-pool.yaml
kubectl apply -f storage-provisioner.yaml
```

### Create PV and DV

Create PV and save to file pv.yaml

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-u1
spec:
  storageClassName: hostpath-csi
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 200Gi
  hostPath:
    path: /opt/aprabh/kubevirt/ubuntu/pv-u1
```

Create DV and save to file dv.yaml

```
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: dv-ubuntu1
spec:
  source:
      upload: {}
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 200Gi
    storageClassName: hostpath-csi
```

### Apply PV and DV

```
kubectl apply -f pv.yaml
kubectl apply -f dv.yaml 
```

### Verify 
```
root@k8s-master:/home/aprabh/kubevirt-manifests/ubuntu_vm/CDI/vm1# kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS   REASON   AGE
pv-u1    200Gi      RWO            Retain           Bound    default/dv-ubuntu1-scratch   hostpath-csi            27m
pv-u12   200Gi      RWO            Retain           Bound    default/dv-ubuntu1           hostpath-csi            26m

root@k8s-master:/home/aprabh/kubevirt-manifests/ubuntu_vm/CDI/vm1# kubectl get dv
NAME         PHASE         PROGRESS   RESTARTS   AGE
dv-ubuntu1   UploadReady   N/A                   7m47s

root@k8s-master:/home/aprabh/kubevirt-manifests/ubuntu_vm/CDI/vm1# kubectl get pvc
NAME                 STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
dv-ubuntu1           Bound    pv-u12   200Gi      RWO            hostpath-csi   7m48s
dv-ubuntu1-scratch   Bound    pv-u1    200Gi      RWO            hostpath-csi   7m48s
```

Make sure PVs are bound. DV mentioning upload-ready indicates now image can be uploaded 

## References 
- [https://github.com/kubevirt/hostpath-provisioner-operator/blob/main/README.md](https://github.com/kubevirt/hostpath-provisioner-operator/blob/main/README.md)
- [https://developers.redhat.com/blog/2020/11/18/using-multus-and-datavolume-in-kubevirt#using_multus_in_kubevirt](https://developers.redhat.com/blog/2020/11/18/using-multus-and-datavolume-in-kubevirt#using_multus_in_kubevirt)
