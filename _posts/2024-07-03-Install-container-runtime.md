## Install Container Runtime

This post will guide you through the steps of installing container runtime for Kubernetes nodes.

# Intalling ContainerD
==**Ref**==: https://github.com/containerd/containerd/blob/main/docs/getting-started.md
## Installing containerd
Download the `containerd-<VERSION>-<OS>-<ARCH>.tar.gz` archive from [https://github.com/containerd/containerd/releases](https://github.com/containerd/containerd/releases) and run following command:
```Shell
tar Cxzvf /usr/local containerd-1.6.2-linux-amd64.tar.gz
```
Output:
```Shell
bin/
bin/containerd-shim-runc-v2
bin/containerd-shim
bin/ctr
bin/containerd-shim-runc-v1
bin/containerd
bin/containerd-stress
```

## systemd
If you intend to start containerd via systemd, you should also download the `containerd.service` unit file from [https://raw.githubusercontent.com/containerd/containerd/main/containerd.service](https://raw.githubusercontent.com/containerd/containerd/main/containerd.service) into `/etc/systemd/system/containerd.service` and run following command:
```Shell
systemctl daemon-reload
systemctl enable --now containerd
```

## Installing runc
Download the `runc.<ARCH>` binary from [https://github.com/opencontainers/runc/releases](https://github.com/opencontainers/runc/releases) and install it as `/usr/local/sbin/runc`
```Shell
install -m 755 runc.amd64 /usr/local/sbin/runc
```

## Installing CNI Plugins
Download the `cni-plugins-<OS>-<ARCH>-<VERSION>.tgz` archive from [https://github.com/containernetworking/plugins/releases](https://github.com/containernetworking/plugins/releases) and extract it under `/opt/cni/bin`
```Shell
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```
Output:
```Shell
./
./macvlan
./static
./vlan
./portmap
./host-local
./vrf
./bridge
./tuning
./firewall
./host-device
./sbr
./loopback
./dhcp
./ptp
./ipvlan
./bandwidth
```

Set owner to root for the bin directory:
```Shell
cd /opt/cni/bin
chown root:root -R bin
```

## Customizing containerd
Load the default config
```Shell
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
```

### Configuring the `systemd` cgroup driver
```Shell
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
```

## Install Container Runtime Interface (CRI) CLI
==**Ref**==: 
- https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md

- using `wget`:
```shell
VERSION="v1.30.0" # check latest version in /releases page
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
```

- using `curl`:
```shell
VERSION="v1.30.0" # check latest version in /releases page
curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-${VERSION}-linux-amd64.tar.gz --output crictl-${VERSION}-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
```
