# k8s-setup-nfs

在 K8S 的群集中的 Pod 重啟時狀態也會還原, 會遇到如何將資料在群集中儲存下來的課題, 於是乎 Kubernetes 就抽象出 `Volume` 的概念作為儲存的容器:

* 將 Pod 指定一個 Volume 做資料的保存.
* 而這個 Volume 不僅可以保留 Pod 應用的資料也能夠讓 Pod 中的容器做資料的共享.
* Pod 可以使用任意數目的 Volume.
* `Persistent Volume` 的存活週期比 Pod 更長, 即使 Pod 不存在時, Kubernetes 也不會銷毀 Persistent Volume.

![setup-persistent-data-pic-1](https://godleon.github.io/blog/images/kubernetes/dynamic-volume-provision.jpg)

* ___PersistentVolume (PV)___ : 由 K8S 群集管理者規劃儲存方案, 提供 PV 給使用者, PV 也是一個抽象的物件, 使用者只要知道對於儲存容器有怎樣的需求而不用去實作細節. PV 可分為靜態產生或是由 `Storage Classes` 動態的產生.

* ___StorageClass___ : 是一個由群集管理者定義儲存的方式也就是一種儲存的類別, 不一樣的 StorageClass 通常對應不同的服務等級, 備份策略, 回收規範.

* ___PersistentVolumeClaim (PVC)___ : 使用者對於 PV 的聲明, 聲明會指出要求的資源等級, ex. CPU & Memory 存取的權限, 甚至指定 StorageClass 或是綁定特定的 PV.

## Use Network File System (NFS) as PersistentVolume

現階段我是使用 Network File System (NFS) 做為群集中儲存資料的容器, NFS 上的資料相較於其他資料儲存方案是比較簡單的做法, 建議資料要定期的備份, 以下會分兩個步驟說明:

1. Install nfs package on nodes of cluster.
2. Deploy [NFS Subdir External Provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner) to cluster

## Install nfs package on nodes of cluster

使用 NFS Server 做 Kubernetes 但前提是群集中的節點和 NFS Server 要能網路連通.

* Install NFS Server on single node of cluster

```
apt-get install nfs-kernel-server nfs-common
```

* Create directory for NFS Server

```bat
mkdir /data/nfs-share
sudo chmod -R 777 /data/nfs-share
```

* Modify insecure option in /etc/exports

```bat
echo "/data/nfs-share 10.88.26.0/24(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
```

* Re-export the share

```bat
sudo exportfs -rav
```

* 重啟 nfs-kernel-server

```bat
/etc/init.d/nfs-kernel-server restart
```

* Install NFS common on other nodes of cluster

```
apt-get install nfs-common
```

## Deploy NFS Subdir External Provisioner

我們只要將群集中的 NFS Server 設定好後, 接著就可以將 NFS subdir external provisioner 佈署到群集中做 automatic provisioner.
這麼一來當 StorageClass 接受到 PVC 後就會向 provisioner 要求動態的產生 PV.

* 預設的資料夾命名規則為 `${namespace}-${pvcName}-${pvName}`.
* PersistentVolPVume 回收的时候預設的命名規則為 `archieved-${namespace}-${pvcName}-${pvName}`.

* 先建立一個 namespace

```
kubectl create namespace test-pvc
```

* 佈署 [建立 Service Account 以及給予相關的權限]](<https://gitlab01.mic.com.tw/sfcs/planet/kubernetes/k8s-setup-nfs/-/blob/main/nfs-deploy/rbac.yaml>).

```
kubectl apply -f nfs-deploy/rbac.yaml
```

* 佈署 [NFS Subdir External Provisioner 服務](https://gitlab01.mic.com.tw/sfcs/planet/kubernetes/k8s-setup-nfs/-/blob/DEV/nfs-deploy/deployment.yaml).

```
kubectl apply -f nfs-deploy/deployment.yaml
```

到這邊我們就完成了 NFS Subdir External Provisioner 的佈署了, 接下來有兩種方式來提供 PV:

* Static : 群集管理者會先建立數個 PV, 由他們來決定這些 PV 背後真實的儲存細節.

* Dynamic : 當群集管理者所建立的靜態 PV 都無法匹配使用者的 PVC 時, 群集將會動態的提供 PV 給 PVC. 但這是基於 `StorageClass` 的前提, 也就是說 PVC 需要向 StorageClass 取得那些動態配置出來的 PV.

如果 PVC 不要取得動態產生的 PV, 那 StorageClass 屬性須設定為空字串 "".

# PVC binding to PV

PVC 綁定 PV 是一對一的對應關係, 另外 PV 可以使用 ClaimRef 指定要保留給哪個 PVC 使用, 這樣一來就達到了雙向綁定的關係. 另外只要群集中要是沒有 PV 能滿足 PVC 的要求, 那 PVC 會一直是 Unbound 的狀態直到有滿足需求的 PV 出現.

# Dynamic Provisioning and Storage Classes in Kubernetes

向先前所提到的需要透過 StorageClass 才能動態的產生 PV, 所以我們需要聲明一個 StorageClass, 接著測試看看是否動態產生 PV.

* 聲明一個使用 NFS Subdir External Provisioner 的 StorageClass.

```
kubectl apply -f dynamic-case/class.yaml
```

* 聲明一個 PVC 去綁定 NFS StorageClass 產生的 PV (動態產生).

```
kubectl apply -f dynamic-case/test-pvc.yaml
```

在創建 PVC 後, Control Plane 會查找滿足 Resource Request 的 PV, 如果 Control Plane 在指定的 StorageClass 中有合適的 PV, 則會將 PVC 綁定到 PV 上.

我們可以確認剛剛聲明的 PVC `STATUS 是否為 Bound`

```
kubectl get pvc
```

此時也可以查看 /data/nfs-share 下產生了一個資料夾 預設的資料夾命名規則為 `${namespace}-${pvcName}-${pvName}`.

```
test-pvc-test-claim-pvc-def74e30-ff5e-46be-9560-c5fe69b7ee27
```

* 佈署一個 Pod 可以指定一個 PVC 而不是直接指定 PV, 還記得先前提到的抽象化嗎? 使用者只要去聲明自己對於儲存條件的要求, 至於最後實際使用的 PV 是不用去關心的.

```
kubectl apply -f dynamic-case/test-pod.yaml
```

## Remove PersistentVolume

* 當 PV 與 PVC 進行了綁定後. 如果 PVC 沒有被刪除是無法直接刪除 PV, 如果 Pod 已經聲明使用了 PVC, 同樣在 Pod 沒被刪除 PVC 也不會被刪除.
* 預設的情況下當 test-claim 被刪除後, 當初所綁定的 PV 也會一同被刪除, 但是由於我們在聲明 StorageClass.reclaimPolicy 使用的是 Retain 所以 PV 會保留下來. 但是即使再次聲明 dynamic-case/test-pvc.yaml, 先前的 PV 也不會再次被綁定了.

## Reserving a PersistentVolume

接下來要提到的是 static 的作法, 情境是不論 PVC 是否重新聲明, 所指定的 PV 是固定可以重複使用的. 這樣群集管理者必須事先建立好 PV, 讓 Pod 透過 PVC 指定使用.

所以在 static-case/test-pvc.yaml 可以看到 spec.volumeName 直接去指定了 test-pv , 但是這樣並不保證 test-pv 只給予我們的 PVC 使用, 因為其他的 PVC 仍然也可以指定它, 既然如此我們必須使用 `pre-bind`, 透過 `claimRef` 去指定同 namespace 下的 test-pvc, 這麼一來就作了雙向綁定了.

* 佈署一個使用 NFS Server 的 PV 並且預先綁定給 test-pvc.

```
kubectl apply -f static-case/test-pv.yaml
```

* 佈署一個使用 test-pv 的 PVC.

```
kubectl apply -f static-case/test-pvc.yaml
```

* 佈署一個使用 test-claim 的 Pod.

```
kubectl apply -f static-case/test-pod.yaml
```

接著我們就可以看到出現了以下的資料夾綁定到了 Pod

```
/data/nfs-share/static-pv
```

## Reference Link

* [Configure a Pod to Use a PersistentVolume for Storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)
* [利用NFS动态提供Kubernetes后端存储卷](https://jimmysong.io/kubernetes-handbook/practice/using-nfs-for-persistent-storage.html) by jimmysong.io
* [Kubernetes 中部署 NFS-Subdir-External-Provisioner 为 NFS 提供动态分配卷](http://www.mydlq.club/article/109/) by 超级小豆丁

__Enjoy!__
