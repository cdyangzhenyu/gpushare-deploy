## 参考

[GPUSHARE](https://github.com/AliyunContainerService/gpushare-scheduler-extender)
gpushare是阿里云开源的k8s device plugin，实现了将单个GPU的共享给多个k8s pod使用，通过显存来控制调度。
可以使gpu的调度粒度细化，提高gpu利用率。

## 依赖

- Kubernetes 1.11+
- golang 1.10+
- NVIDIA drivers ~= 361.93
- Nvidia-docker version > 2.0 (see how to [install](https://github.com/NVIDIA/nvidia-docker) and it's [prerequisites](https://github.com/nvidia/nvidia-docker/wiki/Installation-\(version-2.0\)#prerequisites))
- Docker configured with Nvidia as the [default runtime](https://github.com/NVIDIA/nvidia-docker/wiki/Advanced-topics#default-runtime).

## 安装依赖环境

### 宿主机安装nvidia drivers, cuda等

```bash
cat <<EOF> /etc/modprobe.d/blacklist-nouveau.conf    
blacklist nouveau
options nouveau modeset=0
EOF
dracut --force

#centos7
wget https://developer.download.nvidia.cn/compute/cuda/repos/rhel7/x86_64/cuda-repo-rhel7-10.1.168-1.x86_64.rpm
rpm -ivh cuda-repo-*.rpm
yum install cuda-drivers

reboot
```

### 检查是否安装成功
```bash
[root@node1 ~]# lsmod | grep nvidia
nvidia_drm             40960  0 
nvidia_modeset       1093632  1 nvidia_drm
nvidia_uvm            790528  0 
nvidia              17895424  14 nvidia_uvm,nvidia_modeset
ipmi_msghandler       102400  2 ipmi_devintf,nvidia
drm_kms_helper        167936  2 bochs_drm,nvidia_drm
drm                   458752  5 drm_kms_helper,bochs_drm,nvidia_drm,ttm

[root@node1 gpushare-deploy]# nvidia-smi
Tue Jun 18 14:34:55 2019       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 418.67       Driver Version: 418.67       CUDA Version: 10.1     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 108...  Off  | 00000000:00:09.0 Off |                  N/A |
| 23%   38C    P8    10W / 250W |      0MiB / 11178MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

### 安装nvidia docker2

```bash
# If you have nvidia-docker 1.0 installed: we need to remove it and all existing GPU containers
docker volume ls -q -f driver=nvidia-docker | xargs -r -I{} -n1 docker ps -q -a -f volume={} | xargs -r docker rm -f
sudo yum remove nvidia-docker

# Add the package repositories
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.repo | \
  sudo tee /etc/yum.repos.d/nvidia-docker.repo

# Install nvidia-docker2 and reload the Docker daemon configuration
sudo yum install -y nvidia-docker2
sudo pkill -SIGHUP dockerd

# Test nvidia-smi with the latest official CUDA image
docker run --runtime=nvidia --rm nvidia/cuda:9.0-base nvidia-smi
```

### 设置nvidia为docker默认runtime
```bash
[root@node1 ~]# cat /etc/docker/daemon.json 
{
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "runtimeArgs": []
    }
  },
  "default-runtime": "nvidia"
}

[root@node1 ~]# systemctl restart docker
```

## 安装gpushare

### 部署gpushare调度扩展组件
```bash
cp scheduler-policy-config.json /etc/kubernetes/
kubectl create -f gpushare-schd-extender.yaml
```

### 修改k8s的scheduler配置
```bash
vi /etc/kubernetes/manifests/kube-scheduler.yaml
...
- command:
    - kube-scheduler
+   - --policy-config-file=/etc/kubernetes/scheduler-policy-config.json
...
    volumeMounts:
    - mountPath: /etc/kubernetes/scheduler.conf
      name: kubeconfig
      readOnly: true
+    - mountPath: /etc/kubernetes/scheduler-policy-config.json
+      name: scheduler-policy-config
+      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/scheduler.conf
      type: FileOrCreate
    name: kubeconfig
+  - hostPath:
+      path: /etc/kubernetes/scheduler-policy-config.json
+      type: FileOrCreate
+    name: scheduler-policy-config
...
```

### 部署device plugin
```bash
kubectl create -f device-plugin-rbac.yaml
kubectl create -f device-plugin-ds.yaml
```

### 为需要gpu share的节点打标签
```bash
kubectl label node mynode gpushare=true
```
此时device plugin的pod会启动（该pod使用daemonset，利用标签选择节点）

### 确保kubectl客户端为1.12以上，否则安装
```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.12.1/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/bin/kubectl
```

### 测试安装
```bash
[root@node1 gpushare-deploy]# kubectl inspect gpushare -d

NAME:       node1
IPADDRESS:  10.1.32.199

NAME         NAMESPACE  GPU0(Allocated)  
Allocated :  0 (0%)     
Total :      10         
----------


Allocated/Total GPU Memory In Cluster:  0/10 (0%) 

```

## gpushare功能测试

上面所有服务都正常后，进行如下测试：

```bash
kubectl create -f test-gpushare.yaml
```
等待测试pod起来后

```bash
[root@node1 gpushare-deploy]# nvidia-smi 
Tue Jun 18 14:27:55 2019       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 418.67       Driver Version: 418.67       CUDA Version: 10.1     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 108...  Off  | 00000000:00:09.0 Off |                  N/A |
| 23%   42C    P2    56W / 250W |   3769MiB / 11178MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    0      5476      C   python                                      1253MiB |
|    0      5477      C   python                                      1253MiB |
|    0      5493      C   python                                      1253MiB |
+-----------------------------------------------------------------------------+

[root@node1 gpushare-deploy]# kubectl inspect gpushare -d

NAME:       node1
IPADDRESS:  10.1.32.199

NAME         NAMESPACE  GPU0(Allocated)
binpack-1-0  default    1
binpack-1-1  default    1
binpack-1-2  default    1
Allocated :  3 (30%)
Total :      10
-------------------------------------------------------------------------------


Allocated/Total GPU Memory In Cluster:  3/10 (30%)
```
- 可以看到一块1080ti的gpu卡已经被多个进程同时使用，每个占用显存1253MB。
- 另外如果申请的资源不够会调度失败（总资源够，单个卡资源不够也会失败，类似虚拟机内存调度）

