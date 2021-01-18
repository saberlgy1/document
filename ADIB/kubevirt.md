
## 集成部署

- 内核：`5.10.2-1.el7.elrepo.x86_64`

- 操作系统：`CentOS Linux release 7.9.2009 (Core)`

- docker：`20.10.1`
    
- kubernetes：`v1.18.6`
    
- KubeVirt：`v0.35.0`

- harbor：`v2.1.1`

- harbor domain：`harbor.cloud.com`

### 依赖检测

> 检测宿主是否满足虚拟化条件

安装`libvirt-client`

    yum install -y libvirt-client
    
检测

    [root@node3 ~]# virt-host-validate qemu
      QEMU: Checking for hardware virtualization                                 : PASS
      QEMU: Checking if device /dev/kvm exists                                   : PASS
      QEMU: Checking if device /dev/kvm is accessible                            : PASS
      QEMU: Checking if device /dev/vhost-net exists                             : PASS
      QEMU: Checking if device /dev/net/tun exists                               : PASS
      QEMU: Checking for cgroup 'memory' controller support                      : PASS
      QEMU: Checking for cgroup 'memory' controller mount-point                  : PASS
      QEMU: Checking for cgroup 'cpu' controller support                         : PASS
      QEMU: Checking for cgroup 'cpu' controller mount-point                     : PASS
      QEMU: Checking for cgroup 'cpuacct' controller support                     : PASS
      QEMU: Checking for cgroup 'cpuacct' controller mount-point                 : PASS
      QEMU: Checking for cgroup 'cpuset' controller support                      : PASS
      QEMU: Checking for cgroup 'cpuset' controller mount-point                  : PASS
      QEMU: Checking for cgroup 'devices' controller support                     : PASS
      QEMU: Checking for cgroup 'devices' controller mount-point                 : PASS
      QEMU: Checking for cgroup 'blkio' controller support                       : PASS
      QEMU: Checking for cgroup 'blkio' controller mount-point                   : PASS
      QEMU: Checking for device assignment IOMMU support                         : PASS
      QEMU: Checking if IOMMU is enabled by kernel                               : WARN (IOMMU appears to be disabled in kernel. Add intel_iommu=on to kernel cmdline arguments)
      
> 处理`IOMMU`告警

- 方法一

修改`GRUB_CMDLINE_LINUX`添加`intel_iommu=on`
    
修改前

    [root@node3 ~]# cat /etc/default/grub
    GRUB_TIMEOUT=5
    GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
    GRUB_DEFAULT=saved
    GRUB_DISABLE_SUBMENU=true
    GRUB_TERMINAL_OUTPUT="console"
    GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos00/root rhgb quiet"
    GRUB_DISABLE_RECOVERY="true"
    
修改后

    GRUB_TIMEOUT=5
    GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
    GRUB_DEFAULT=saved
    GRUB_DISABLE_SUBMENU=true
    GRUB_TERMINAL_OUTPUT="console"
    GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos00/root rhgb quiet intel_iommu=on"
    GRUB_DISABLE_RECOVERY="true"

重建内核引导文件

    grub2-mkconfig -o /boot/grub2/grub.cfg
    dracut --regenerate-all --force
    
重启

    reboot
    
验证

    cat /proc/cmdline |grep intel_iommu=on
    
如返回为空，尝试方法二
    
- 方法二

查询引导文件

    [root@node3 ~]# find / -name "grub.cfg"
    /boot/efi/EFI/centos/grub.cfg
    /boot/grub2/grub.cfg
    
修改`/boot/efi/EFI/centos/grub.cfg`文件内容

修改前

    linuxefi /vmlinuz-5.10.2-1.el7.elrepo.x86_64 root=/dev/mapper/centos00-root ro crashkernel=auto rd.lvm.lv=centos00/root rhgb quiet LANG=en_US.UTF-8
    
修改后

    linuxefi /vmlinuz-5.10.2-1.el7.elrepo.x86_64 root=/dev/mapper/centos00-root ro crashkernel=auto rd.lvm.lv=centos00/root rhgb quiet intel_iommu=on LANG=en_US.UTF-8

重建内核引导文件

    grub2-mkconfig -o /boot/grub2/grub.cfg
    dracut --regenerate-all --force
    
重启

    reboot
    
验证

    cat /proc/cmdline |grep intel_iommu=on

### 部署KubeVirt operator

> 下载`kubevirt-operator.yaml`

- [v0.35.0版本下载链接](https://github.com/kubevirt/kubevirt/releases/download/v0.35.0/kubevirt-operator.yaml)

> 下载`kubevirt operator`所需镜像

镜像列表

    kubevirt/virt-operator:v0.35.0
    kubevirt/virt-api:v0.35.0
    kubevirt/virt-controller:v0.35.0
    kubevirt/virt-handler:v0.35.0
    kubevirt/virt-launcher:v0.35.0

修改镜像tag，修改后如下

    harbor.cloud.com/kubevirt/virt-operator:v0.35.0
    harbor.cloud.com/kubevirt/virt-api:v0.35.0
    harbor.cloud.com/kubevirt/virt-controller:v0.35.0
    harbor.cloud.com/kubevirt/virt-handler:v0.35.0
    harbor.cloud.com/kubevirt/virt-launcher:v0.35.0
    
推送至私有仓库

    docker push harbor.cloud.com/kubevirt/virt-operator:v0.35.0
    docker push harbor.cloud.com/kubevirt/virt-api:v0.35.0
    docker push harbor.cloud.com/kubevirt/virt-controller:v0.35.0
    docker push harbor.cloud.com/kubevirt/virt-handler:v0.35.0
    docker push harbor.cloud.com/kubevirt/virt-launcher:v0.35.0
    
> 上传`kubevirt-operator.yaml`并调整镜像tag

    ...
    containers:
        - command:
        - virt-operator
        - --port
        - "8443"
        - -v
        - "2"
        env:
        - name: OPERATOR_IMAGE
          value: harbor.cloud.com/kubevirt/virt-operator:v0.35.0
        - name: WATCH_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['olm.targetNamespaces']
        - name: KUBEVIRT_VERSION
          value: v0.35.0
        - name: VIRT_API_SHASUM
          value: sha256:bf38c1997f3c60a71d53b956f235973834d37c0c604b5711084b2a7ef8cd3c7b
        - name: VIRT_CONTROLLER_SHASUM
          value: sha256:7b81c59034df51c1a1f54d3180e0df678469790b6a8ac4fcc5fcaa615b1ca84c
        - name: VIRT_HANDLER_SHASUM
          value: sha256:14b4bd6d62b585ef2f4dbacafc75a66a6c575c64ed630835cf4ef6c0f77d40d1
        - name: VIRT_LAUNCHER_SHASUM
          value: sha256:a6d9f1dada1d33a218ba9ed0494d2e2cd09f5596eff5eb5b8d70bfe1fd4f8812
        image: harbor.cloud.com/kubevirt/virt-operator:v0.35.0
    ...
    
上传k8s节点发布

    kubectl apply -f kubevirt-operator.yaml

### 部署KubeVirt CR

> 下载`kubevirt-cr.yaml`

- [v0.35.0版本下载链接](https://github.com/kubevirt/kubevirt/releases/download/v0.35/kubevirt-cr.yaml)
    
> 添加热迁移功能

修改`kubevirt-cr.yaml`前

    ---
    apiVersion: kubevirt.io/v1alpha3
    kind: KubeVirt
    metadata:
      name: kubevirt
      namespace: kubevirt
    spec:
      certificateRotateStrategy: {}
      configuration: {}
      customizeComponents: {}
      imagePullPolicy: IfNotPresent
      
修改`kubevirt-cr.yaml`后

    ---
    apiVersion: kubevirt.io/v1alpha3
    kind: KubeVirt
    metadata:
      name: kubevirt
      namespace: kubevirt
    spec:
      certificateRotateStrategy: {}
      configuration: {}
      customizeComponents: {}
      imagePullPolicy: IfNotPresent
      configuration:
        developerConfiguration:
          featureGates:
            - "DataVolumes"
            - "LiveMigration"
            
> 上传k8s节点发布

    kubectl apply -f kubevirt-cr.yaml
    
> 查看组件启动情况

    [root@node3 kubevirt]# kubectl get pod -n kubevirt
    NAME                               READY   STATUS    RESTARTS   AGE
    virt-api-bb7f45d97-526fx           1/1     Running   0          10m
    virt-api-bb7f45d97-vm8sl           1/1     Running   0          10m
    virt-controller-5b6bd8d48c-dw29f   1/1     Running   0          10m
    virt-controller-5b6bd8d48c-ft5sj   1/1     Running   0          10m
    virt-handler-d4vx9                 1/1     Running   0          10m
    virt-handler-wwtmg                 1/1     Running   0          10s
    virt-handler-zxbmp                 1/1     Running   0          10m
    virt-operator-66b9d8fbf8-gcdfs     1/1     Running   0          11m
    virt-operator-66b9d8fbf8-knxss     1/1     Running   0          11m
    
> k8s物理机节点打上标签

    kubectl label nodes node3 HostMachine=physical
    kubectl label nodes node4 HostMachine=physical
    kubectl label nodes node5 HostMachine=physical
    
> 调整`virt-handler DaemonSet`节点至物理机节点

    kubectl patch ds/virt-handler -n kubevirt -p '{"spec": {"template": {"spec": {"nodeSelector": {"HostMachine": "physical"}}}}}'

查看组件运行情况

    [root@node3 kubevirt]# kubectl get pod -n kubevirt
    NAME                               READY   STATUS    RESTARTS   AGE
    virt-api-bb7f45d97-526fx           1/1     Running   0          12m
    virt-api-bb7f45d97-vm8sl           1/1     Running   0          12m
    virt-controller-5b6bd8d48c-dw29f   1/1     Running   0          11m
    virt-controller-5b6bd8d48c-ft5sj   1/1     Running   0          11m
    virt-handler-4pm5c                 1/1     Running   0          16s
    virt-handler-k2qcr                 1/1     Running   0          52s
    virt-handler-wwtmg                 1/1     Running   0          81s
    virt-operator-66b9d8fbf8-gcdfs     1/1     Running   0          12m
    virt-operator-66b9d8fbf8-knxss     1/1     Running   0          12m
    
### 部署virtctl

为什么安装`virtctl`?

对`VirtualMachineInstance`基础操作可以通过`kubectl`进行管理。然而，virtctl二进制有以下高级特性：

- 串行和图形控制台访问

`virtctl`还提供了方便的命令:

- 启/停虚拟机实例

- 实时迁移虚拟机实例

- 上传虚拟机磁盘镜像

> 下载`virtctl`

[virtctl 0.35.0](https://github.com/kubevirt/kubevirt/releases/download/v0.35.0/virtctl-v0.35.0-linux-amd64)

> 上传安装

    mv virtctl-*-linux-amd64 /usr/bin/virtctl
    chmod +x /usr/bin/virtctl
    
### 安装CDI

> 下载`cdi-operator.yaml`与`cdi-cr.yaml`

- [cdi-operator.yaml](https://github.com/kubevirt/containerized-data-importer/releases/download/v1.28.0/cdi-operator.yaml)
- [cdi-cr.yaml](https://github.com/kubevirt/containerized-data-importer/releases/download/v1.28.0/cdi-cr.yaml)

> 下载镜像导入本地镜像库

    kubevirt/cdi-controller:v1.28.0
    kubevirt/cdi-importer:v1.28.0
    kubevirt/cdi-cloner:v1.28.0
    kubevirt/cdi-apiserver:v1.28.0
    kubevirt/cdi-uploadserver:v1.28.0
    kubevirt/cdi-uploadproxy:v1.28.0
    kubevirt/cdi-operator:v1.28.0
            
修改`cdi-operator.yaml`镜像tag

    sed -i 's#kubevirt/cdi-#harbor.cloud.com/kubevirt/cdi-#g' cdi-operator.yaml
    
> 创建发布

    kubectl create -f cdi-operator.yaml
    kubectl create -f cdi-cr.yaml
    
> 查看组件运行情况

    [root@node3 kubevirt]# kubectl get pod -n cdi
    NAME                               READY   STATUS    RESTARTS   AGE
    cdi-apiserver-d6448b7f5-d5c2z      1/1     Running   0          64s
    cdi-deployment-6f7974947c-59bh9    1/1     Running   0          57s
    cdi-operator-7d5c8cdcb-wc8hk       1/1     Running   0          94s
    cdi-uploadproxy-6bc6cfbf98-fsbt9   1/1     Running   0          54s
    

### 创建虚拟机

> 下载`CentOS7 Cloud`镜像

**相比iso 云镜像(Cloud Images)有以下优势：**

- OS已经预安装：云镜像一般以qcow2, vhd, ovf的虚拟磁盘格式发布，包含了一个完整的系统快照，不再需要从头安装。
- 通过cloud-init配置：cloud-init是AWS首先推出的，用于对一个镜像做个性化配置，主要是用来注入登录的用户名/密码，SSH密钥对，甚至还可以用来对磁盘进行扩容等。目前cloud-init已经被几乎所有的OS厂商集成到其云镜像中作为镜像初始化的标准。

[CentOS-7-x86_64-GenericCloud-2009.qcow2下载链接](https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-2009.qcow2)

> 获取`cdi-uploadproxy`地址

    [root@node3 kubevirt]# kubectl get svc -n cdi
    NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
    cdi-api                  ClusterIP   10.233.47.29   <none>        443/TCP   54m
    cdi-prometheus-metrics   ClusterIP   10.233.6.228   <none>        443/TCP   54m
    cdi-uploadproxy          ClusterIP   10.233.20.73   <none>        443/TCP   54m

> 上传系统镜像

    [root@node3 netdata]# virtctl image-upload pvc centos-vm-disk --size=10240Mi --image-path=/cloud/kubevirt/CentOS-7-x86_64-GenericCloud-2009.qcow2 --uploadproxy-url=https://10.233.20.73:443 --insecure
    Using existing PVC default/centos-vm-disk
    Uploading data to https://10.233.20.73:443
    
     847.81 MiB / 847.81 MiB [=====================================================================================================] 100.00% 3s
    
    Uploading data completed successfully, waiting for processing to complete, you can hit ctrl-c without interrupting the progress
    Processing completed successfully
    Uploading /cloud/kubevirt/CentOS-7-x86_64-GenericCloud-2009.qcow2 completed successfully
    You have new mail in /var/spool/mail/root

> 创建虚拟机

配额：

- 内存：16Gi


    cat <<EOF | kubectl apply -f -
    apiVersion: kubevirt.io/v1alpha3
    kind: VirtualMachineInstance
    metadata:
      name: centos7-vm1
    spec:
      domain:
        cpu:
          sockets: 1
          cores: 2
          threads: 2
        resources:
          requests:
            memory: "1Gi"
            cpu: "2"
        devices:
          disks:
          - name: mypvcdisk
            # This makes it a disk
            disk: {}
        machine:
          type: ""
      terminationGracePeriodSeconds: 0
      volumes:
        - name: mypvcdisk
          persistentVolumeClaim:
            claimName: centos7-vm1-disk-pvc
    status: {}
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: centos7-vm1-disk-pvc
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 100Gi
    EOF

> 

### 清理KubeVirt

其他清理方式可导致k8s集群异常（`namespace Terminating`）

    kubectl delete -n kubevirt kubevirt kubevirt --wait=true # --wait=true should anyway be default
    kubectl delete apiservices v1alpha3.subresources.kubevirt.io # this needs to be deleted to avoid stuck terminating namespaces
    kubectl delete mutatingwebhookconfigurations virt-api-mutator # not blocking but would be left over
    kubectl delete validatingwebhookconfigurations virt-api-validator # not blocking but would be left over
    kubectl delete -f kubevirt-operator.yaml --wait=false
    kubectl -n kubevirt patch kv kubevirt --type=json -p '[{ "op": "remove", "path": "/metadata/finalizers" }]'
