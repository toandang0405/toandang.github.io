## This post will guide you through the steps of preparing the OS for installing Kubernetes, install Load Balancer for multiple master nodes

# Install and config Load Balancer for multiple Master Nodes
==**Ref**==: https://github.com/justmeandopensource/kubernetes/tree/master/kubeadm-ha-keepalived-haproxy/external-keepalived-haproxy

Set up 2 load balancer nodes:
```Shell
sudo apt-get update
sudo apt-get install -y keepalived haproxy
```

Configure keepalived healthcheck on 2 nodes:
```Shell
sudo cat >> /etc/keepalived/check_apiserver.sh <<EOF
#!/bin/sh

errorExit() {
  echo "*** $@" 1>&2
  exit 1
}

curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
if ip addr | grep -q YOUR_VIRTUAL_IP; then
  curl --silent --max-time 2 --insecure https://YOUR_VIRTUAL_IP:6443/ -o /dev/null || errorExit "Error GET https://YOUR_VIRTUAL_IP:6443/"
fi
EOF

sudo chmod +x /etc/keepalived/check_apiserver.sh
```

Create keepalived config file on Node 1 (Keepalived Master node):
```Shell
sudo cat >> /etc/keepalived/keepalived.conf <<EOF
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  timeout 10
  fall 5
  rise 2
  weight -2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth1
    virtual_router_id 1
    priority 101
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass mysecret
    }
    virtual_ipaddress {
        YOUR_VIRTUAL_IP
    }
    track_script {
        check_apiserver
    }
}
EOF
```

Create keepalived config file on Node 2 (Keepalived Slave node):
```Shell
sudo cat >> /etc/keepalived/keepalived.conf <<EOF
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  timeout 10
  fall 5
  rise 2
  weight -2
}

vrrp_instance VI_1 {
    state SLAVE
    interface eth1
    virtual_router_id 1
    priority 100
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass mysecret
    }
    virtual_ipaddress {
        YOUR_VIRTUAL_IP
    }
    track_script {
        check_apiserver
    }
}
EOF
```

Enable keepalived service:
```Shell
sudo systemctl enable --now keepalived
sudo systemctl start keepalived
```

Configure haproxy on 2 nodes:
```Shell
sudo cat >> /etc/haproxy/haproxy.cfg <<EOF

frontend kubernetes-frontend
  bind *:6443
  mode tcp
  option tcplog
  default_backend kubernetes-backend

backend kubernetes-backend
  option httpchk GET /healthz
  http-check expect status 200
  mode tcp
  option ssl-hello-chk
  balance roundrobin
    server kmaster1 MASTER_1_IP:6443 check fall 3 rise 2
    server kmaster2 MASTER_2_IP:6443 check fall 3 rise 2
    server kmaster3 MASTER_3_IP:6443 check fall 3 rise 2

EOF
```

Enable and restart haproxy on 2 nodes:
```Shell
systemctl enable haproxy && systemctl restart haproxy
```

Debug error HAProxy
```Shell
haproxy -f /etc/haproxy/haproxy.cfg -db
```

# Prepare the OS for Kubernetes Installation
## Disable Swap
```Shell
sudo swapoff -a
```

Comment the swap line in fstab file
```Shell
sudo vi /etc/fstab (comment the swap line)
#/swap.img      none    swap    sw      0       0
```

## Enable kernel modules and configure sysctl
```Shell
sudo modprobe overlay  
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/kubernetes.conf<<EOF  
net.bridge.bridge-nf-call-ip6tables = 1  
net.bridge.bridge-nf-call-iptables = 1  
net.ipv4.ip_forward = 1  
EOF

sudo tee /etc/modules-load.d/k8s.conf <<EOF  
overlay  
br_netfilter  
EOF
```

## If you has disable IPv6, enabled it in  GRUB config
```Shell
sudo nano /etc/default/grub
```
Change these values:
```Shell
GRUB_CMDLINE_LINUX_DEFAULT="ipv6.disable=0"
GRUB_CMDLINE_LINUX="ipv6.disable=0"
```

```Shell
sudo update-grub
reboot
```

## Update GRUB config for eBPF (if needed)
==**Ref**==: https://docs.cilium.io/en/stable/operations/system_requirements/#advanced-features
```Shell
sudo update-grub CONFIG_BPF=y
sudo update-grub CONFIG_BPF_SYSCALL=y
sudo update-grub CONFIG_NET_CLS_BPF=y
sudo update-grub CONFIG_BPF_JIT=y
sudo update-grub CONFIG_NET_CLS_ACT=y
sudo update-grub CONFIG_NET_SCH_INGRESS=y
sudo update-grub CONFIG_CRYPTO_SHA1=y
sudo update-grub CONFIG_CRYPTO_USER_API_HASH=y
sudo update-grub CONFIG_CGROUPS=y
sudo update-grub CONFIG_CGROUP_BPF=y
sudo update-grub CONFIG_PERF_EVENTS=y
sudo update-grub CONFIG_SCHEDSTATS=y
sudo update-grub CONFIG_NETFILTER_XT_SET=m
sudo update-grub CONFIG_IP_SET=m
sudo update-grub CONFIG_IP_SET_HASH_IP=m
sudo update-grub CONFIG_NETFILTER_XT_TARGET_TPROXY=m
sudo update-grub CONFIG_NETFILTER_XT_TARGET_CT=m
sudo update-grub CONFIG_NETFILTER_XT_MATCH_MARK=m
sudo update-grub CONFIG_NETFILTER_XT_MATCH_SOCKET=m
sudo update-grub CONFIG_XFRM=y
sudo update-grub CONFIG_XFRM_OFFLOAD=y
sudo update-grub CONFIG_XFRM_STATISTICS=y
sudo update-grub CONFIG_XFRM_ALGO=m
sudo update-grub CONFIG_XFRM_USER=m
sudo update-grub CONFIG_INET{,6}_ESP=m
sudo update-grub CONFIG_INET{,6}_IPCOMP=m
sudo update-grub CONFIG_INET{,6}_XFRM_TUNNEL=m
sudo update-grub CONFIG_INET{,6}_TUNNEL=m
sudo update-grub CONFIG_INET_XFRM_MODE_TUNNEL=m
sudo update-grub CONFIG_CRYPTO_AEAD=m
sudo update-grub CONFIG_CRYPTO_AEAD2=m
sudo update-grub CONFIG_CRYPTO_GCM=m
sudo update-grub CONFIG_CRYPTO_SEQIV=m
sudo update-grub CONFIG_CRYPTO_CBC=m
sudo update-grub CONFIG_CRYPTO_HMAC=m
sudo update-grub CONFIG_CRYPTO_SHA256=m
sudo update-grub CONFIG_CRYPTO_AES=m
```

Or:
```Shell
sudo nano /etc/default/grub
```
Add these values:
```Shell
GRUB_CMDLINE_LINUX="bpf bpf_jit net_cls_bpf net_cls_act net_sch_ingress crypto_sha1 crypto_user_api_hash cgroups cgroup_bpf perf_events schedstats"
```

Then:
```Shell
sudo update-grub
reboot
```
