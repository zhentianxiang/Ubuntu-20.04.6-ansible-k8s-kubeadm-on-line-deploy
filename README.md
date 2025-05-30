### 1. 基础环境配置

```sh
root@ubuntu:~# yum -y install ansible sshpass
root@ubuntu:~# vim /etc/ansible/ansible.cfg
 10 [defaults]
 11 ansible_shell_executable = /usr/bin/bash # 新增这个
root@ubuntu:~# ssh-keygen -t rsa
root@ubuntu:~# vim iplist.txt
11.0.1.11
11.0.1.12
11.0.1.13
11.0.1.14
11.0.1.15
11.0.1.16
11.0.1.17
11.0.1.18
11.0.1.19
root@ubuntu:~# for host in $(cat iplist.txt); do sshpass -p 'your_password' ssh-copy-id -o StrictHostKeyChecking=no 'your_username'@$host; done
root@ubuntu:~# ansible -i hosts.ini all -m shell -a "whoami"
```

### 2. 升级内核

不是必须的，根据实际情况来判断自己是否要升级内核

```sh
# 由于网络问题可以提前搞一下，如果你的机器可以科学上网则无需
root@ubuntu:~# ansible -i hosts.ini k8s -m copy -a "src=./kernel-lt-5.4.160-1.el7.elrepo.x86_64.rpm dest=/tmp/kernel-lt-5.4.160-1.el7.elrepo.x86_64.rpm mode=0644" --become
# 先把 hosts.init 文件配置好在执行
root@ubuntu:~# ansible-playbook -i hosts.ini install_kernel.yml
```

### 3. 准备修改关键配置文件

- 单 master 部署如下修改

```sh
root@ubuntu:~# cp roles/init/templates/no-etcd-hosts.j2 roles/init/templates/hosts.j2
root@ubuntu:~# vim hosts.ini 
[all]
k8s-master1 ansible_connection=local  ip=11.0.1.11
#k8s-master2 ansible_host=11.0.1.12 ip=11.0.1.12 ansible_port=22 ansible_user=root
#k8s-master3 ansible_host=11.0.1.13 ip=11.0.1.13 ansible_port=22 ansible_user=root
#etcd1 ansible_host=11.0.1.14 ip=11.0.1.14 ansible_port=22 ansible_user=root
#etcd2 ansible_host=11.0.1.15 ip=11.0.1.15 ansible_port=22 ansible_user=root
#etcd3 ansible_host=11.0.1.16 ip=11.0.1.16 ansible_port=22 ansible_user=root
k8s-node1 ansible_host=11.0.1.17 ip=11.0.1.17 ansible_port=22 ansible_user=root
k8s-node2 ansible_host=11.0.1.18 ip=11.0.1.18 ansible_port=22 ansible_user=root
k8s-node3 ansible_host=11.0.1.19 ip=11.0.1.19 ansible_port=22 ansible_user=root
# 对应更改all.yml 定义的master ip变量
[k8s]
k8s-master1
#k8s-master2
#k8s-master3
k8s-node1
#k8s-node2
#k8s-node3

[master]
k8s-master1
#k8s-master2
#k8s-master3

[node]
k8s-node1
k8s-node2
k8s-node3

[etcd]
#etcd1
#etcd2
#etcd3

# keepalived 高可用集群 + Nginx 负载均衡

# 如果不部署单 master ha 里面的可以注释掉了,避免产生警告信息
[ha]
#k8s-master1 ha_name=ha-master
#k8s-master2 ha_name=ha-backup
#k8s-master3 ha_name=ha-backup

#24小时token过期后添加node节点
[newnode]
[k8s:children]
master
node
newnode
```
重点修改的如下：

> k8s_extra_ips:  kubrenetes master 节点信息(预留)，目的为了后期方便扩容 master 节点
>
> nic:  keepalived 调用的本地网卡的设备
>
> vip: keepalived 的虚拟 IP，如果部署单节点 master 把这个值写为 master 的 IP 地址即可
>
> lb_port: nginx 的负载均衡监听的端口，如果部署单节点 master 把这个值写为 6443 即可
>
> extra_ips:  etcd 集群的节点信息(预留)，目的为了后期方便扩容 etcd 节点
>
> calico_network: calico 调用本地网卡的设备

```sh
root@ubuntu:~# vim group_vars/all.yml
# 允许普通用户执行 sudo
ansible_become: true
ansible_become_method: sudo

# 安装源配置
docker_ce_repo: '"deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"'
docker_gpg_key: ' http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg'
docker_ce: 'docker-ce=5:20.10.10~3-0~ubuntu-focal'
docker_ce_cli: 'docker-ce-cli=5:20.10.10~3-0~ubuntu-focal'

# kubernetes 安装信息
kube_gpg_key: "https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg"
kube_repo: "deb [arch=amd64] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main"
ghproxy: 'https://ghfast.top'
kube_version: '1.23.0-00'
code_version: 'v1.23.0'
tmp_dir: '/opt/k8s-install/join'
docker_data_dir: '/var/lib/docker'
kubelet_data_dir: '/var/lib/kubelet'
k8s_image_url: 'registry.cn-hangzhou.aliyuncs.com/google_containers'

# k8s master 节点预留 ip
k8s_extra_ips:
  - "11.0.1.36"
  - "11.0.1.37"
  - "11.0.1.38"
  - "11.0.1.39"
service_cidr: '10.96.0.0/16'                                                          # service ip 段
cluster_dns: '10.96.0.1'                                                              # kube-dns 服务器地址
pod_cidr: '192.18.0.0/16'                                                             # pod ip 段
calico_network: '"interface=ens192"'                                                  # 服务器宿主机网卡信息可以用 "," 分割

# keepalived 配置
nic: 'ens192'
Virtual_Router_ID: '51'
vip: '11.0.1.11'
api_vip_hosts: apiserver.cluster.local
notification_emails:
  - acassen@firewall.loc
  - failover@firewall.loc
  - sysadmin@firewall.loc
smtp_server: '127.0.0.1'
smtp_connect_timeout: '30'
auth_pass: 'kubernetes'

# 负载均衡端口
lb_port: '6443'

# ETCD 集群基本配置
etcd_data: '/var/lib/etcd-external'
etcd_version: 'v3.5.9'
etcd_conf: "/etc/etcd/"
etcd_ssl: '/etc/etcd/ssl'
etcd_cluster_token: "etcd-k8s-cluster"
# ETCD 集群初始状态 (new 或 existing)
etcd_initial_cluster_state: "new"
# 启动等待超时(秒)
etcd_startup_timeout: 60

# ETCD 集群备份
etcd_backup_dir: "/var/lib/etcd-backup"
etcd_backup_keep_days: 7
# etcd-server-csr host 信息预留
extra_ips:
  - "11.0.1.40"
  - "11.0.1.41"
  - "11.0.1.42"
  - "11.0.1.43"

# 性能调优参数
etcd_heartbeat_interval: 200
etcd_election_timeout: 2500
etcd_quota_backend_bytes: 5500000000
etcd_auto_compaction_retention: "1"
etcd_snapshot_count: 50000

# 安全配置
etcd_client_cert_auth: true
etcd_peer_client_cert_auth: true
etcd_auto_tls: false
etcd_peer_auto_tls: false
force_cert_gen: true
force_cert_sync: true

# 日志配置
etcd_log_level: "warn"
etcd_log_output: "stderr"

# 功能开关
etcd_enable_pprof: false
etcd_enable_v2: false
etcd_enable_localhost: true

# 第三方 yaml 路径
k8s_app: '/opt/k8s-install/app'                                                       # 创建了一个存放 yaml 文件的主目录
ingress_app: '/opt/k8s-install/app/ingress'                                           # ingress yaml 存放位置
ingres_label: 'ingress/type: nginx'                                                   # ingress 部署节点 label
openebs_app: '/opt/k8s-install/app/openebs_app'                                       # openebs_app yaml 存放位置
openebs_data: '"/data/openebs"'                                                       # openebs local pvc 数据存储目录
calico_app: '/opt/k8s-install/app/calico'                                             # calico yaml 存放位置

# 定义 Kubernetes 版本与 Calico 版本的映射相关文档: https://docs.tigera.io/calico/3.28/getting-started/kubernetes/requirements
k8s_calico_version_map:
  "1.28": "v3.28.0"
  "1.27": "v3.27.0"
  "1.26": "v3.26.0"
  "1.25": "v3.25.0"
  "1.24": "v3.24.0"
  "1.23": "v3.24.0"
  "1.22": "v3.24.0"

# 定义 Kubernetes 版本与 ingress-nginx 版本的映射
k8s_ingress_version_map:
  "1.29": "v1.10.0"
  "1.28": "v1.9.5"
  "1.27": "v1.9.5"
  "1.26": "v1.9.5"
  "1.25": "v1.5.1"
  "1.24": "v1.5.1"
  "1.23": "v1.5.1"
  "1.22": "v1.5.1"
default_calico_version: "v3.25.0"
default_ingress_version: "v1.5.1"

# 自定义 hosts 解析,ansible 会帮我们自动添加
custom_hosts:
  registry.example.com: 127.0.0.1
```
```sh
root@ubuntu:~# ansible-playbook -i hosts.ini single-master-deploy.yml  # 单节点部署
```
- 多 master 集群方式部署

```sh
root@ubuntu:~# cp roles/init/templates/yes-etcd-hosts.j2 roles/init/templates/hosts.j2
root@ubuntu:~# sed -i 's/^#etcd/etcd/' hosts.ini
```


```sh
root@ubuntu:~# vim hosts.ini
[all]
k8s-master1 ansible_connection=local  ip=11.0.1.11
k8s-master2 ansible_host=11.0.1.12 ip=11.0.1.12 ansible_port=22 ansible_user=root
k8s-master3 ansible_host=11.0.1.13 ip=11.0.1.13 ansible_port=22 ansible_user=root
etcd1 ansible_host=11.0.1.14 ip=11.0.1.14 ansible_port=22 ansible_user=root
etcd2 ansible_host=11.0.1.15 ip=11.0.1.15 ansible_port=22 ansible_user=root
etcd3 ansible_host=11.0.1.16 ip=11.0.1.16 ansible_port=22 ansible_user=root
k8s-node1 ansible_host=11.0.1.17 ip=11.0.1.17 ansible_port=22 ansible_user=root
k8s-node2 ansible_host=11.0.1.18 ip=11.0.1.18 ansible_port=22 ansible_user=root
k8s-node3 ansible_host=11.0.1.19 ip=11.0.1.19 ansible_port=22 ansible_user=root
# 对应更改all.yml 定义的master ip变量
[k8s]
k8s-master1
k8s-master2
k8s-master3
k8s-node1
k8s-node2
k8s-node3

[master]
k8s-master1
k8s-master2
k8s-master3

[node]
k8s-node1
k8s-node2
k8s-node3

[etcd]
etcd1
etcd2
etcd3

# keepalived 高可用集群 + Nginx 负载均衡

# 如果不部署单 master ha 里面的可以注释掉了,避免产生警告信息
[ha]
k8s-master1 ha_name=ha-master
k8s-master2 ha_name=ha-backup
k8s-master3 ha_name=ha-backup

#24小时token过期后添加node节点
[newnode]
[k8s:children]
master
node
newnode
```
```sh
# 由于网络问题可以提前搞一下，如果你的机器可以科学上网则无需
root@ubuntu:~# ansible -i hosts.ini etcd -m copy -a "src=./roles/etcd/files/cfssl_linux-amd64 dest=/usr/local/bin/cfssl mode=0755" --become
root@ubuntu:~# ansible -i hosts.ini etcd -m copy -a "src=./roles/etcd/files/cfssljson_linux-amd64 dest=/usr/local/bin/cfssl-json mode=0755" --become
root@ubuntu:~# ansible -i hosts.ini etcd -m copy -a "src=./roles/etcd/files/cfssl-certinfo_linux-amd64 dest=/usr/local/bin/cfssl-certinfo mode=0755" --become
root@ubuntu:~# ansible -i hosts.ini etcd -m copy -a "src=./roles/etcd/files/etcd-v3.5.1-linux-amd64.tar.gz dest=/usr/local/src/etcd-v3.5.1-linux-amd64.tar.gz mode=0644" --become
```



重点修改的如下：

> k8s_extra_ips:  kubrenetes master 节点信息(预留)，目的为了后期方便扩容 master 节点
>
> nic:  keepalived 调用的本地网卡的设备
>
> vip: keepalived 的虚拟 IP，如果部署单节点 master 把这个值写为 master 的 IP 地址即可
>
> lb_port: nginx 的负载均衡监听的端口，如果部署单节点 master 把这个值写为 6443 即可
>
> extra_ips:  etcd 集群的节点信息(预留)，目的为了后期方便扩容 etcd 节点
>
> calico_network: calico 调用本地网卡的设备

```sh
root@ubuntu:~# vim group_vars/all.yml
# 允许普通用户执行 sudo
ansible_become: true
ansible_become_method: sudo

# 安装源配置
docker_ce_repo: '"deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"'
docker_gpg_key: ' http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg'
docker_ce: 'docker-ce=5:20.10.10~3-0~ubuntu-focal'
docker_ce_cli: 'docker-ce-cli=5:20.10.10~3-0~ubuntu-focal'

# kubernetes 安装信息
kube_gpg_key: "https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg"
kube_repo: "deb [arch=amd64] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main"
ghproxy: 'https://ghfast.top'
kube_version: '1.23.0-00'
code_version: 'v1.23.0'
tmp_dir: '/opt/k8s-install/join'
docker_data_dir: '/var/lib/docker'
kubelet_data_dir: '/var/lib/kubelet'
k8s_image_url: 'registry.cn-hangzhou.aliyuncs.com/google_containers'

# k8s master 节点预留 ip
k8s_extra_ips:
  - "11.0.1.36"
  - "11.0.1.37"
  - "11.0.1.38"
  - "11.0.1.39"
service_cidr: '10.96.0.0/16'                                                          # service ip 段
cluster_dns: '10.96.0.1'                                                              # kube-dns 服务器地址
pod_cidr: '192.18.0.0/16'                                                             # pod ip 段
calico_network: '"interface=ens192"'                                                  # 服务器宿主机网卡信息可以用 "," 分割

# keepalived 配置
nic: 'ens192'
Virtual_Router_ID: '51'
vip: '11.0.1.20'
api_vip_hosts: apiserver.cluster.local
notification_emails:
  - acassen@firewall.loc
  - failover@firewall.loc
  - sysadmin@firewall.loc
smtp_server: '127.0.0.1'
smtp_connect_timeout: '30'
auth_pass: 'kubernetes'

# 负载均衡端口
lb_port: '16443'

# ETCD 集群基本配置
etcd_data: '/var/lib/etcd-external'
etcd_version: 'v3.5.9'
etcd_conf: "/etc/etcd/"
etcd_ssl: '/etc/etcd/ssl'
etcd_cluster_token: "etcd-k8s-cluster"
# ETCD 集群初始状态 (new 或 existing)
etcd_initial_cluster_state: "new"
# 启动等待超时(秒)
etcd_startup_timeout: 60

# ETCD 集群备份
etcd_backup_dir: "/var/lib/etcd-backup"
etcd_backup_keep_days: 7
# etcd-server-csr host 信息预留
extra_ips:
  - "11.0.1.40"
  - "11.0.1.41"
  - "11.0.1.42"
  - "11.0.1.43"

# 性能调优参数
etcd_heartbeat_interval: 200
etcd_election_timeout: 2500
etcd_quota_backend_bytes: 5500000000
etcd_auto_compaction_retention: "1"
etcd_snapshot_count: 50000

# 安全配置
etcd_client_cert_auth: true
etcd_peer_client_cert_auth: true
etcd_auto_tls: false
etcd_peer_auto_tls: false
force_cert_gen: true
force_cert_sync: true

# 日志配置
etcd_log_level: "warn"
etcd_log_output: "stderr"

# 功能开关
etcd_enable_pprof: false
etcd_enable_v2: false
etcd_enable_localhost: true

# 第三方 yaml 路径
k8s_app: '/opt/k8s-install/app'                                                       # 创建了一个存放 yaml 文件的主目录
ingress_app: '/opt/k8s-install/app/ingress'                                           # ingress yaml 存放位置
ingres_label: 'ingress/type: nginx'                                                   # ingress 部署节点 label
openebs_app: '/opt/k8s-install/app/openebs_app'                                       # openebs_app yaml 存放位置
openebs_data: '"/data/openebs"'                                                       # openebs local pvc 数据存储目录
calico_app: '/opt/k8s-install/app/calico'                                             # calico yaml 存放位置

# 定义 Kubernetes 版本与 Calico 版本的映射相关文档: https://docs.tigera.io/calico/3.28/getting-started/kubernetes/requirements
k8s_calico_version_map:
  "1.28": "v3.28.0"
  "1.27": "v3.27.0"
  "1.26": "v3.26.0"
  "1.25": "v3.25.0"
  "1.24": "v3.24.0"
  "1.23": "v3.24.0"
  "1.22": "v3.24.0"

# 定义 Kubernetes 版本与 ingress-nginx 版本的映射
k8s_ingress_version_map:
  "1.29": "v1.10.0"
  "1.28": "v1.9.5"
  "1.27": "v1.9.5"
  "1.26": "v1.9.5"
  "1.25": "v1.5.1"
  "1.24": "v1.5.1"
  "1.23": "v1.5.1"
  "1.22": "v1.5.1"
default_calico_version: "v3.25.0"
default_ingress_version: "v1.5.1"

# 自定义 hosts 解析,ansible 会帮我们自动添加
custom_hosts:
  registry.example.com: 127.0.0.1
```

```sh
root@ubuntu:~# ansible-playbook -i hosts.ini multi-master-ha-deploy.yml   # 集群部署
```

### 4. 验证集群

```
[root@localhost ~]# sudo etcdctl --cacert=/etc/etcd/ssl/etcd-ca.pem --cert=/etc/etcd/ssl/etcd-server.pem --key=/etc/etcd/ssl/etcd-server-key.pem --endpoints="https://etcd1:2379,https://etcd2:2379,https://etcd3:2379" member list -w table
+------------------+---------+-------+------------------------+------------------------+------------+
|        ID        | STATUS  | NAME  |       PEER ADDRS       |      CLIENT ADDRS      | IS LEARNER |
+------------------+---------+-------+------------------------+------------------------+------------+
|   d1f1c4b9eec273 | started | etcd1 | https://11.0.1.14:2380 | https://11.0.1.14:2379 |      false |
| 517f93955c7fe7d3 | started | etcd3 | https://11.0.1.16:2380 | https://11.0.1.16:2379 |      false |
| 777a9bac189012d3 | started | etcd2 | https://11.0.1.15:2380 | https://11.0.1.15:2379 |      false |
+------------------+---------+-------+------------------------+------------------------+------------+

[root@localhost ~]# sudo etcdctl --cacert=/etc/etcd/ssl/etcd-ca.pem --cert=/etc/etcd/ssl/etcd-server.pem --key=/etc/etcd/ssl/etcd-server-key.pem --endpoints="https://etcd1:2379,https://etcd2:2379,https://etcd3:2379" endpoint status -w table
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://etcd1:2379 |   d1f1c4b9eec273 |   3.5.9 |  6.2 MB |     false |      false |         5 |       7688 |               7688 |        |
| https://etcd2:2379 | 777a9bac189012d3 |   3.5.9 |  6.2 MB |     false |      false |         5 |       7688 |               7688 |        |
| https://etcd3:2379 | 517f93955c7fe7d3 |   3.5.9 |  6.2 MB |      true |      false |         5 |       7688 |               7688 |        |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

[root@localhost ~]# sudo etcdctl --cacert=/etc/etcd/ssl/etcd-ca.pem --cert=/etc/etcd/ssl/etcd-server.pem --key=/etc/etcd/ssl/etcd-server-key.pem --endpoints="https://etcd1:2379,https://etcd2:2379,https://etcd3:2379" endpoint health --write-out=table
+--------------------+--------+-------------+-------+
|      ENDPOINT      | HEALTH |    TOOK     | ERROR |
+--------------------+--------+-------------+-------+
| https://etcd2:2379 |   true | 14.959001ms |       |
| https://etcd1:2379 |   true | 15.752313ms |       |
| https://etcd3:2379 |   true | 17.795671ms |       |
+--------------------+--------+-------------+-------+

# 查看 ETCD 备份情况
[root@localhost ~]# systemctl status etcd-backup.timer
[root@localhost ~]# systemctl status etcd-backup.service
[root@localhost ~]# ls -lh /var/lib/etcd-backup/
[root@localhost ~]# ETCDCTL_API=3 etcdctl --cacert=/etc/etcd/ssl/etcd-ca.pem \
  --cert=/etc/etcd/ssl/etcd-server.pem --key=/etc/etcd/ssl/etcd-server-key.pem \
  --endpoints=https://etcd1:2379 snapshot status /var/lib/etcd-backup/etcd-snapshot-最新文件名.db

root@ubuntu:~# kubectl get cs

root@ubuntu:~# kubectl get node

root@ubuntu:~# kubectl get pods -A

root@ubuntu:~# kubectl create deployment nginx --image=harbor.meta42.indc.vnet.com/library/nginx:latest --replicas=4

root@ubuntu:~# kubectl expose deployment nginx --port=80 --target-port=80 --type=NodePort
```

### 5. 调整 kube 启动参数

```sh
# 请手动执行以下命令来修改 kube 自定义配置：
root@ubuntu:~# sed -i "/image:/i\    - --feature-gates=RemoveSelfLink=false" /etc/kubernetes/manifests/kube-apiserver.yaml # 每台 master 都执行
root@ubuntu:~# sed -i "s/bind-address=127.0.0.1/bind-address=0.0.0.0/g" /etc/kubernetes/manifests/kube-controller-manager.yaml # 每台 master 都执行
root@ubuntu:~# kubectl get cm -n kube-system kube-proxy -o yaml | sed "s/metricsBindAddress: \"\"/metricsBindAddress: \"0.0.0.0\"/g" | kubectl replace -f -
root@ubuntu:~# kubectl rollout restart daemonset -n kube-system kube-proxy
```

### 6. 解决 node 节点报错

```sh
Sep 14 00:59:22 k8s-node1 kubelet[1611]: E0914 00:59:22.040084    1611 file_linux.go:61] "Unable to read config path" err="path does not exist, ignoring" path="/etc/kubernetes/manifests"
[root@k8s-node1 ~]# ansible -i hosts.ini node -m shell -a "mkdir -pv /etc/kubernetes/manifests"
[root@k8s-node1 ~]# ansible -i hosts.ini node -m shell -a "systemctl restart kubelet"
```

### 7. 新机器加入集群后的操作

```sh
# 新增机器的 IP 地址
[root@k8s-master1 Centos7-ansible-k8s-kubeadm-on-line-deploy-main]# cat iplist.txt 
11.0.1.11
11.0.1.12
11.0.1.13
11.0.1.14
11.0.1.15
11.0.1.16
11.0.1.17
11.0.1.18
11.0.1.19
11.0.1.21 # 新增
11.0.1.22 # 新增
11.0.1.23 # 新增
```
```sh
root@ubuntu:~# for host in $(cat iplist.txt); do sshpass -p 'your_password' ssh-copy-id -o StrictHostKeyChecking=no 'your_username'@$host; done
```

```
root@ubuntu:~# vim hosts.ini
[all]  # 这个分组下面新增3个
k8s-openebs-storage-1 ansible_host=11.0.1.21 ip=11.0.1.21 ansible_port=22 ansible_user=root
k8s-openebs-storage-2 ansible_host=11.0.1.22 ip=11.0.1.22 ansible_port=22 ansible_user=root
k8s-openebs-storage-3 ansible_host=11.0.1.23 ip=11.0.1.23 ansible_port=22 ansible_user=root
#24小时token过期后添加node节点
[newnode]
k8s-openebs-storage-1
k8s-openebs-storage-2
k8s-openebs-storage-3
```

```sh
root@ubuntu:~# ansible -i hosts.ini newnode -m shell -a "whoami"
```

```sh
# 升级内核 --limit 指定 newnode 分组,因为只有新加入的机器需要拷贝一次
root@ubuntu:~# ansible -i hosts.ini newnode -m copy -a "src=./kernel-lt-5.4.160-1.el7.elrepo.x86_64.rpm dest=/tmp/kernel-lt-5.4.160-1.el7.elrepo.x86_64.rpm mode=0644" --become
```

```sh
# --limit 指定 newnode 分组,默认文件里面应该是 k8s 分组
root@ubuntu:~# ansible-playbook -i hosts.ini --limit newnode install_kernel.yml
```

```sh
# 添加节点
root@ubuntu:~# ansible-playbook -i hosts.ini add-node.yml
```

```sh
# 修改 hosts 添加
root@ubuntu:~# vim /etc/hosts
127.0.0.1       localhost

# The following lines are desirable for IPv6 capable hosts
::1     localhost       ip6-localhost   ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

# k8s 高可用 VIP
11.0.1.20   apiserver.cluster.local

# 输出 k8s 组主机
11.0.1.11 k8s-master1
11.0.1.12 k8s-master2
11.0.1.13 k8s-master3
11.0.1.17 k8s-node1
11.0.1.18 k8s-node2
11.0.1.19 k8s-node3
11.0.1.21 k8s-storage-1
11.0.1.22 k8s-storage-2
11.0.1.23 k8s-storage-3

# 输出 etcd 组主机
11.0.1.14 etcd1
11.0.1.15 etcd2
11.0.1.16 etcd3

# 输出自定义的 hosts 解析
127.0.0.1   registry.example.com

# 统一 hosts
root@ubuntu:~# ansible -i hosts.ini all -m copy -a "src=/etc/hosts dest=/etc/hosts mode=0644" --become
```

```sh
# 拷贝 harbor 证书文件，当然 ansbile 中是没有的，需要后期自己部署 harbor，只是为了使用 ansible 统一集群配置
root@ubuntu:~# ansible -i hosts.ini newnode -m copy -a "src=/etc/docker/certs.d/ dest=/etc/docker/certs.d/ mode=0755" --become
```

### 8. 卸载删除集群

```sh
root@ubuntu:~# ansible-playbook -i hosts.ini remove-k8s.yml
```
