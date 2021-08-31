# Kubevirt with Anthos BM

We will explore running a Windows VM on Anthos BM cluster using Kubevirt. This documentation is based the article [here](https://kubevirt.io/2020/KubeVirt-installing_Microsoft_Windows_from_an_iso.html)

## Prerequisites
* Install an Anthos BM cluster as explained [here](../standalone/readme.md)

* A client configured to access this Anthos BM cluster

* Docker should be running on the client machine

* Client machine needs GUI and [VNC viewer](https://www.realvnc.com/en/connect/download/viewer/)

## Enable Kubevirt on Anthos BM cluster

AnthosBM cluster configuration is a custom resource of type `cluster` stored in the cluster-CLUSTERNAME namespace. In order to find the CR, run the following command, substituting your cluster name.

```
kubectl get cluster -n cluster-CLUSTERNAME
```

You will see a result similar to the following

```
NAME          AGE
CLUSTERNAME   6d23h
```

Edit this CR to enable kubevirt by running `kubectl edit cluster CLUSTERNAME -n cluster-CLUSTERNAME` substituting CLUSTERNAME with the value of the name of your BM cluster.

and add `spec.kubevirt.useEmulation: true`

```
spec:
  kubevirt:
    useEmulation: true
```
and save the CR and exit.

This will turn on Kubevirt on your bare metal clsuter.

Verify running `kubectl get po -n kubevirt` and you should observe a list of pods as below:

```
NAME                               READY   STATUS    RESTARTS   AGE
virt-api-7dcd85dd74-kjm89          1/1     Running   5          6d23h
virt-api-7dcd85dd74-wv2zb          1/1     Running   4          5d6h
virt-controller-66d7c5bf65-f4w6z   1/1     Running   5          6d23h
virt-controller-66d7c5bf65-lhjjc   1/1     Running   4          5d6h
virt-handler-hzvpk                 1/1     Running   5          6d23h
virt-handler-m4p87                 1/1     Running   5          6d23h
virt-handler-wdq58                 1/1     Running   6          6d23h
virt-operator-6d448f5fb-s5s6h      1/1     Running   4          5d6h
```

Also verify that CDI is also running `kubectl get po -n cdi`. Output will be similar to

```
NAME                               READY   STATUS    RESTARTS   AGE
cdi-apiserver-68b49b85dd-cnsj7     1/1     Running   4          5d6h
cdi-deployment-766665ffb8-pnq99    1/1     Running   5          6d23h
cdi-operator-6555b6cd76-hpgk2      1/1     Running   4          5d6h
cdi-uploadproxy-5b4f875675-hstn6   1/1     Running   4          5d6h
```


## Install Virt tooling on your client

* Set up Krew following the instructions [here](https://krew.sigs.k8s.io/docs/user-guide/setup/install/)

* Run `kubectl krew` to check the installation

* Install `virt` plugin using krew

```
kubectl krew install virt
```

* Verify running `kubectl virt`

## Download Windows 

* Download 64 bit Windows Server ISO from the [Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2019) and save it locally

* Note the location of the image

## Upload image using CDI to persistent storage

The ISO image downloaded earlier will be uploaded next to a persistent volume so that we can stand up a VM using that ISO.

* Get the CDI Upload Proxy URL 

```
export PROXY_IP=$(kubectl get services cdi-uploadproxy -n cdi -o jsonpath='{.spec.loadBalancerIP}')
```

* Run the following command to upload the ISO by substituting the PATHTOISOFILE. Choose the pvc name of your choice.

```
kubectl virt image-upload \
--image-path=PATHTOISOFILE \
--pvc-name=iso-win2k12 \
--access-mode=ReadWriteOnce \
--pvc-size=10G \
--storage-class=longhorn \
--uploadproxy-url=https://${PROXY_IP}:443 \
--insecure \
--wait-secs=240
```

This command will run a few mins and while the image is uploaded

## Pull virtio container image

* Pull virtio container image to the local docker by running

```
docker pull kubevirt/virtio-container-disk
```

## Install VM

* Create a persistent volume file for the Windows VM named `winhd-pvc.yaml` with the following content

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: winhd
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 15Gi
  storageClassName: hostpath
```

* Create a VM custom resource with the following content that references the persistent volume from where the ISO is pulled. Save this into a file named `windoes-vm.yaml`

```
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: win2k12-iso
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/domain: win2k12-iso
    spec:
      domain:
        cpu:
          cores: 4
        devices:
          disks:
          - bootOrder: 1
            cdrom:
              bus: sata
            name: cdromiso
          - disk:
              bus: virtio
            name: harddrive
          - cdrom:
              bus: sata
            name: virtiocontainerdisk
        machine:
          type: q35
        resources:
          requests:
            memory: 8G
      volumes:
      - name: cdromiso
        persistentVolumeClaim:
          claimName: iso-win2k12
      - name: harddrive
        persistentVolumeClaim:
          claimName: winhd
      - containerDisk:
          image: kubevirt/virtio-container-disk
        name: virtiocontainerdisk
```

* Instantiate persistent volume claim and the VM

```
kubectl apply -f winhd-pvc.yaml
kubectl apply -f windoes-vm.yaml
```
output similar to

```
virtualmachine.kubevirt.io/win2k12-iso configured
persistentvolumeclaim/winhd created
```

Verify running `kubectl get vm` 

## Test

* Start the virtual machine running

```
kubectl virt start win2k12-iso
```

output

```
VM win2k12-iso was scheduled to start
```

* It takes a few mins to go from `Scheduled` to `Running`. Verify running `kubectl get vmi`

output similar to

```
NAME          AGE   PHASE     IP           NODENAME
win2k12-iso   13h   Running   10.0.4.118   worker2
```

* Once VM instance is running, you can VNC to the same using

```
kubectl virt vnc win2k12-iso
```

* Once you remote into this pod with VNC, continue windows installatioa and start using the VM










