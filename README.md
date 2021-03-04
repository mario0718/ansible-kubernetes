# Ansible安装二进制高可用kubernetes-1.17.9
------
***Author:Mario***
***Notice: 以下统一在k8s-master01主机执行;MASTER节点固定3台,ETCD版本3.2.30，可执行2N+1台，实际仅有三台实施;WORK节点可任意台***
---
###  一.安装配置ansible

**1)安装**
  ```
  # yum install ansible -y
  ```
**2)安装自己的情况修改**
  ```
  # vim ansible-kubernetes/hosts
  [k8s-all]
  k8s-master01 ansible_ssh_host=192.168.20.40
  k8s-master02 ansible_ssh_host=192.168.20.30
  k8s-master03 ansible_ssh_host=192.168.20.20
  k8s-node01  ansible_ssh_host=192.168.20.39
  k8s-node02  ansible_ssh_host=192.168.20.29
  k8s-node03  ansible_ssh_host=192.168.20.19
  .....
  [k8s-master]
  k8s-master01 ansible_ssh_host=192.168.20.40
  k8s-master02 ansible_ssh_host=192.168.20.30
  k8s-master03 ansible_ssh_host=192.168.20.20
  [k8s-node]
  k8s-node01  ansible_ssh_host=192.168.20.39
  k8s-node02  ansible_ssh_host=192.168.20.29
  k8s-node03  ansible_ssh_host=192.168.20.19
  ......
  [etcd-all]
  etcd-node1  ansible_ssh_host=192.168.20.40
  etcd-node2  ansible_ssh_host=192.168.20.30
  etcd-node3  ansible_ssh_host=192.168.20.20
  ......
  [k8s-all:vars]
  #统一登录密码
  ansible_ssh_user=root 
  ansible_ssh_pass=xxxxxx
  ansible_ssh_port=22
  ```
**3)配置免密钥**

  *生产密钥*
   **在一台k8s-master01主节点上执行: **
```
   # ssh-keygen -t rsa
```
   *配置*
```
   # vim /etc/ansible/ansible.cfg

   #去掉以下个配置的#号
    host_key_checking = False
	command_warnings = False

   # sed -i 's#^\#\(host_key_checking =\).*#\1 False#' /etc/ansible/ansible.cfg
   # sed -i 's#^\#\(command_warnings =\).*#\1 False#' /etc/ansible/ansible.cfg
```
**4)分发公钥证书**

```  
   # ansible -i hosts k8s-all -m  authorized_key -a "user=root key='{{lookup('file','/root/.ssh/id_rsa.pub')}}'"
``` 
   
##  二.安装kubernetes

> **修改kubernetes-ansible/group_vars/all.yaml参数**

>> 1、修改K8S主节点的3台主机名和IP地址;
-----
>> 2、修改ETCD集群所有主机名和IP地址;
-----
>> 3、修改ETCD_URL。修改成自己规划的IP即可;
-----
>> 4、修改ETCD_ENDPOINTS。修改成自己规划的IP即可;
-----
>> 5、修改IFACE。既网卡的名称;
---
###  三.开始运行安装(主控机:k8s-master01)

**1、这里所有主机系统初始化**
  ```
  # ansible-playbook -i hosts setup.yaml
  ```
**2、k8s工作节点初始化升级内核**
  ```
  # ansible-playbook -i hosts k8s-node.yaml  -e kernel=ture
  ```
**3、在k8s-master01上创建我们所需要的所有集群证书**
  ```
  # ansible-playbook -i hosts ca.yaml
  ```
**4、创建kubectl**
  ```
  # ansible-playbook -i hosts kubectl.yaml -e download="true"
  ```
**5、创建etcd集群**
  ```
  # ansible-playbook -i hosts etcd.yaml
  ```
***5.1 查看状态***
 ```
  # ETCDCTL_API=3  etcdctl -w table --cacert=/opt/kubernetes/ssl/ca.pem --cert=/opt/kubernetes/ssl/etcd.pem --key=/opt/kubernetes/ssl/etcd-key.pem --endpoints=https://172.16.2.198:2379,https://172.16.2.79:2379,https://172.16.2.41:2379 endpoint status
  +--------------------------+------------------+---------+---------+-----------+-----------+------------+
  |         ENDPOINT         |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
  +--------------------------+------------------+---------+---------+-----------+-----------+------------+
  | https://10.10.3.181:2379 | 29fd839438b13490 |  3.3.12 |   20 kB |     false |         2 |          8 |
  | https://10.10.3.182:2379 | 15e01ae39b43687d |  3.3.12 |   20 kB |     false |         2 |          8 |
  | https://10.10.3.183:2379 | deaefeb573250ff1 |  3.3.12 |   20 kB |      true |         2 |          8 |
  +--------------------------+------------------+---------+---------+-----------+-----------+------------+
  ```
**6、创建flannel网络**
  ```
   # ansible-playbook -i hosts flannel.yaml
  ```   
***6.1 查看结果***
  ```
   # ansible -i hosts k8s-all -m shell -a "/usr/sbin/ip addr show flannel.1|grep -w inet"
    10.10.3.183 | CHANGED | rc=0 >>
      inet 172.30.216.0/32 scope global flannel.1

    10.10.3.184 | CHANGED | rc=0 >>
      inet 172.30.176.0/32 scope global flannel.1

    10.10.3.182 | CHANGED | rc=0 >>
      inet 172.30.240.0/32 scope global flannel.1

    10.10.3.181 | CHANGED | rc=0 >>
      inet 172.30.88.0/32 scope global flannel.1
  ```
**7、安装kube-apiserver的负载均衡lb,nginx提供四层负载功能**

```
    upstream kube-apiservers {
    hash $remote_addr consistent;
    server {{ MASTER01_HOSTNAME }}:6443 weight=5 max_fails=1 fail_timeout=15s;
    server {{ MASTER02_HOSTNAME }}:6443 weight=5 max_fails=1 fail_timeout=15s;
    server {{ MASTER03_HOSTNAME }}:6443 weight=5 max_fails=1 fail_timeout=15s;
      }
 
    # ansible-playbook - hosts nginx.yaml
```	

**8、安装高可用版api-server**

```
    #ansible-playbook -i hosts kube-api.yaml
```
---	
***① 查看***
```
    #systemctl status kube-apiserver |grep 'Active:'
    Active: active (running) since Tue 2019-04-16 14:25:01 CST; 55s ago
```
---	 
***② 查看日志***
```
    # journalctl -u kube-apiserver
	# tailf /var/log/messages
	# systemctl status kube-apiserver -l
```
---	 
***③ Kubernetes API 健康检查***

      * Kubernetes API 服务器提供 3 个 API 端点（healthz、livez 和 readyz）来表明 API 服务器的当前状态。*
```
    # kubectl get --raw='/readyz?verbose'
    [+]ping ok
    [+]log ok
    [+]etcd ok
    [+]poststarthook/start-kube-apiserver-admission-initializer ok
    [+]poststarthook/generic-apiserver-start-informers ok
    [+]poststarthook/start-apiextensions-informers ok
    [+]poststarthook/start-apiextensions-controllers ok
    [+]poststarthook/crd-informer-synced ok
    [+]poststarthook/bootstrap-controller ok
    [+]poststarthook/rbac/bootstrap-roles ok
    [+]poststarthook/scheduling/bootstrap-system-priority-classes ok
    [+]poststarthook/start-cluster-authentication-info-controller ok
    [+]poststarthook/start-kube-aggregator-informers ok
    [+]poststarthook/apiservice-registration-controller ok
    [+]poststarthook/apiservice-status-available-controller ok
    [+]poststarthook/kube-apiserver-autoregistration ok
    [+]autoregister-completion ok
    [+]poststarthook/apiservice-openapi-controller ok
    [+]shutdown ok
    healthz check passed



    # kubectl get --raw='/healthz?verbose'
    [+]ping ok
    [+]log ok
    [+]etcd ok
    [+]poststarthook/start-kube-apiserver-admission-initializer ok
    [+]poststarthook/generic-apiserver-start-informers ok
    [+]poststarthook/start-apiextensions-informers ok
    [+]poststarthook/start-apiextensions-controllers ok
    [+]poststarthook/crd-informer-synced ok
    [+]poststarthook/bootstrap-controller ok
    [+]poststarthook/rbac/bootstrap-roles ok
    [+]poststarthook/scheduling/bootstrap-system-priority-classes ok
    [+]poststarthook/start-cluster-authentication-info-controller ok
    [+]poststarthook/start-kube-aggregator-informers ok
    [+]poststarthook/apiservice-registration-controller ok
    [+]poststarthook/apiservice-status-available-controller ok
    [+]poststarthook/kube-apiserver-autoregistration ok
    [+]autoregister-completion ok
    [+]poststarthook/apiservice-openapi-controller ok
    [+]shutdown ok
    healthz check passed



    #kubectl get --raw='/livez?verbose'
    [+]ping ok
    [+]log ok
    [+]etcd ok
    [+]poststarthook/start-kube-apiserver-admission-initializer ok
    [+]poststarthook/generic-apiserver-start-informers ok
    [+]poststarthook/start-apiextensions-informers ok
    [+]poststarthook/start-apiextensions-controllers ok
    [+]poststarthook/crd-informer-synced ok
    [+]poststarthook/bootstrap-controller ok
    [+]poststarthook/rbac/bootstrap-roles ok
    [+]poststarthook/scheduling/bootstrap-system-priority-classes ok
    [+]poststarthook/start-cluster-authentication-info-controller ok
    [+]poststarthook/start-kube-aggregator-informers ok
    [+]poststarthook/apiservice-registration-controller ok
    [+]poststarthook/apiservice-status-available-controller ok
    [+]poststarthook/kube-apiserver-autoregistration ok
    [+]autoregister-completion ok
    [+]poststarthook/apiservice-openapi-controller ok
    [+]shutdown ok
    healthz check passed
```

**9、安装kube-controller-manager**
```
     #ansible-playbook -i hosts kube-controller-manager.yaml	 
```
---
***①查看***
```
     # systemctl status kube-controller-manager|grep Active
     Active: active (running) since Tue 2019-04-16 14:27:19 CST; 4min 20s ag
```
***②查看日志***
```
     # journalctl -u kube-controller-manager
```
---
**10、安装kube-scheduler**
```
     # ansible-playbook -i hosts kube-scheduler.yaml
```
***①查看***
```
     # systemctl status kube-scheduler|grep Active
      Active: active (running) since Tue 2019-04-16 14:55:07 CST; 15s ago
```
***②查看日志***
```
     # journalctl -u kube-scheduler
	 # tailf /var/log/messages
	 # systemctl status kube-apiserver -l
```

**11、安装docker-18.09**
```
     #ansible-playbook -i hosts docker.yaml
     #docker info
     Client:
     Context:    default
     Debug Mode: false
     Plugins:
     app: Docker App (Docker Inc., v0.9.1-beta3)
     buildx: Build with BuildKit (Docker Inc., v0.5.1-docker)

     Server:
     Containers: 0
     Running: 0
     Paused: 0
     Stopped: 0
     Images: 0
     Server Version: 18.09.9
     Storage Driver: overlay2
     Backing Filesystem: extfs
     Supports d_type: true
     Native Overlay Diff: true
     Logging Driver: json-file
     Cgroup Driver: systemd
       ......
```

**12、安装 kubelet**
```
    # ansible-playbook -i hosts kubelet.yaml
```

**13、安装 kube-proxy**
```
    # ansible-playbook -i hosts kube-proxy.yaml
```

**14、清除相关文件数据**
```
    # ansible-playbook -i hosts clean.yaml
```

### 四. FAQ:

   *kube-apiserver 常见错误日志：*

```	
   [-] poststarthook/rbac/bootstrap-roles failed: reason withheld
   ...
   healthz is failed
```
  
       **解决**
       *版本BUG,知心如下却很正常！*
```
	 # kubectl get --raw='/healthz?verbose'
    [+]ping ok
    [+]log ok
    [+]etcd ok
    [+]poststarthook/start-kube-apiserver-admission-initializer ok
    [+]poststarthook/generic-apiserver-start-informers ok
    [+]poststarthook/start-apiextensions-informers ok
    [+]poststarthook/start-apiextensions-controllers ok
    [+]poststarthook/crd-informer-synced ok
    [+]poststarthook/bootstrap-controller ok
    [+]poststarthook/rbac/bootstrap-roles ok
    [+]poststarthook/scheduling/bootstrap-system-priority-classes ok
    [+]poststarthook/start-cluster-authentication-info-controller ok
    [+]poststarthook/start-kube-aggregator-informers ok
    [+]poststarthook/apiservice-registration-controller ok
    [+]poststarthook/apiservice-status-available-controller ok
    [+]poststarthook/kube-apiserver-autoregistration ok
    [+]autoregister-completion ok
    [+]poststarthook/apiservice-openapi-controller ok
    [+]shutdown ok
    healthz check passed
```	
	