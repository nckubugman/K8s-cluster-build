# K8s-cluster-build
To Build a K8s-cluster (at least 2 servers).

# 第一章: 基礎環境設置(全部節點)

[系統環境]

CentOS: 7.8.2003
Docker: 18.09.6
Calico: 3.19.0
Kubernetes: 1.18.19
Kubernetes Network: IPVS
Kubernetes DashBoard: 2.0.3


Kubernetes需要一定的環境來保證正常運行，像是各節點時間同步，DNS解析，關閉防火牆等等。

1. 修改Host
    
    分佈式系統環境中的多主機通訊通常機於主機名稱進行，這裡要修改hosts的文件進行DNS解析。
    
    $ vim  /etc/hosts
    
    加入下面內容:
    
    10.30.14.56 k8s-master
    
    10.30.14.54 k8s-node1
    
2. 修改Hostname (修改Hostname後最好重開機)
    
    Kubernetes中會以各個服務的Hostname為其Node命名，所以需要進入不同的Server修改Hostname名稱。
    
    (1) 修改10.30.14.56，設置Hostname，然後將hostname寫入Host
    
    $ hostnamectl set-hostname master-node
    
    $ echo "127.0.0.1 $(hostname)" >> /etc/hosts
    
    (2) 修改10.30.14.54，設置Hostname，然後將hostname寫入Host
    
    $ hostnamectl set-hostname work-node-1
    
    $ echo "127.0.0.1 $(hostname)" >> /etc/hosts
    
    *可用ping試試看兩台server能否互連
    
    *不能的話就重開機試試
    
3. 主機時間同步
    
    將各個Server的時間同步，並設置開機啟動時間同步服務。
    
    $ systemctl start chronyd.service && systemctl enable chronyd.service
    
4. 關閉並禁用SELinux
    
    關閉SELinux，編輯/etc/sysonfig selinux文件，以徹底禁用SELinux。
    
    $ setenforce 0
    
    $ sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
    
    執行完可以用以下指令檢查是否關閉
    
    $ getenforce
    
5. 禁用Swap設備
    
    關閉當前已啟用的所有Swap設備:
    
    $ swapoff -a && sysctl -w vm.swappiness=0
    
    編輯fstab配置文件，註解掉標示為swap設備的所有行:
    
    $ vim /etc/fstab
    
    #/dev/mapper/centos-swap swap  swap defaults 0 0
    
6. 設置Kernel參數
    
    $ cat <<EOF > /etc/sysctl.d/k8s.conf
    
    net.ipv4.ip_forward = 1
    
    net.bridge.bridge-nf-call-ip6tables = 1
    
    net.bridge.bridge-nf-call-iptables = 1
    
    net.ipv4.tcp_keepalive_time = 600
    
    net.ipv4.tcp_keepalive_intvl = 30
    
    net.ipv4.tcp_keepalive_probes = 10
    
    EOF
    
    使配置生效:
    
    $ modprobe br_netfilter
    
    $ sysctl -p /etc/sysctl.d/k8s.conf
    
    查看是否生成相關文件
    
    $ ls /proc/sys/net/bridge
    
7. 配置IPVS module
    
    為了kubeproxy開啟IPVS模式的前提需要加載以下的Kernel module:
    
    - ip_vs
    - ip_vs_rr
    - ip_vs_wrr
    - ip_vs_sh
    - nf_conntrack_ipv4
    
    $ cat > /etc/sysconfig/modules/ipvs.modules << EOF
    
    #!/bin/bash
    
    modprobe — ip_vs
    
    modprobe — ip_vs_rr
    
    modprobe — ip_vs_wrr
    
    modprobe — ip_vs_sh
    
    modprobe — nf_conntrack_ipv4
    
    EOF
    
    執行bash file
    
    #先修改權限
    
    $ chmod 755 /etc/sysconfig/modules/ipvs.modules
    
    #執行bash
    
    $ bash /etc/sysconfig/modules/ipvs.modules
    
    #檢查是否加載成功
    
    $ lsmod | grep -e ip_vs -e nf_conntrack_ipv4
    
    最後要安裝ipset和ipvsadm
    
    $ yum install -y ipset ipvsadm
    
8. 配置資源限制
    
    echo "* soft nofile 65536"  >> /etc/security/limits.conf
    
    echo "* hard nofile 65536" >> /etc/security/limits.conf
    
    echo "* soft nproc 65536"  >> /etc/security/limits.conf
    
    echo "* hard nproc 65536"  >> /etc/security/limits.conf
    
    echo "* soft memlock unlimited"  >> /etc/security/limits.conf
    
    echo "* hard memlock unlimited"  >> /etc/security/limits.conf
    
9. 安裝相關工具
    
    $ yum install -y epel-release
    
    $ yum install -y yum-utils device-mapper-persistent-data lvm2 net-tools conntrack-tools wget ntpdate libseccomp libtool-ltdl
 
