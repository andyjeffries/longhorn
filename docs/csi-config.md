# Longhorn CSI on RancherOS/CoreOS + RKE or K3S

## Requirements
  1. Kubernetes v1.11 or higher.
  2. Longhorn v0.4.1 or higher.
  3. For RancherOS only: Ubuntu console.


## Instruction
### For RancherOS/CoreOS + Kubernetes v1.11 only 
  The following step is not needed for Kubernetes v1.12+.

  Add extra_binds for kubelet in RKE `cluster.yml`:
  ```
   services:
     kubelet:
       extra_binds:
       - "/opt/rke/var/lib/kubelet/plugins:/var/lib/kubelet/plugins" 
   ```
   
### For each node:
#### RancherOS:
  ##### 1. Switch to ubuntu console

  `sudo ros console switch ubuntu`, then type `y`

  ##### 2. Install open-iscsi for each node. 
  ```
  sudo apt update
  sudo apt install -y open-iscsi
  ```
  ##### 3. Modify configuration for iscsi. 
    
  1. Open config file `/etc/iscsi/iscsid.conf`
  2. Comment `iscsid.startup = /bin/systemctl start iscsid.socket`
  3. Uncomment `iscsid.startup = /sbin/iscsid`

#### CoreOS:    

  #####  1. If you want to enable iSCSI daemon automatically at boot, you need to enable the systemd service:

  ```
  sudo su
  systemctl enable iscsid
  reboot
  ```
  
  #####  2. Or just start the iSCSI daemon for the current session:

  ```
  sudo su
  systemctl start iscsid
  ```

#### K3S: 
  ##### 1. For Longhorn v0.7.0+ (v0.7.0 is included)
  Longhorn v0.7.0+ supports k3s v0.10.0+ (v0.10.0 is included) only by default. 
  
  If you want to deploy these new Longhorn versions on k3s v0.10.0- (v0.10.0 is not included), you need to set `--kubelet-root-dir` to `<data-dir>/agent/kubelet` for the Deployment `longhorn-driver-deployer` in `longhorn/deploy/longhorn.yaml`. 
  `data-dir` is a `k3s` arg and it can be set when you launch a k3s server. By default it is `/var/lib/rancher/k3s`.
  
  ##### 2. For Longhorn v0.7.0- (v0.7.0 is not included)
  Longhorn v0.7.0- supports k3s v0.10.0- (v0.10.0 is not included) by default. 
  
  If you want to deploy these Longhorn versions on k3s v0.10.0+, you need to set `--kubelet-root-dir` to `/var/lib/kubelet` for the Deployment `longhorn-driver-deployer` in `longhorn/deploy/longhorn.yaml`

## Troubleshooting
### Common issues
#### Failed to get arg root-dir: Cannot get kubelet root dir, no related proc for root-dir detection ...

This error is due to Longhorn cannot detect where is the root dir setup for Kubelet, so the CSI plugin installation failed.

User can override the root-dir detection by manually setting argument `kubelet-root-dir` here: 
https://github.com/rancher/longhorn/blob/master/deploy/longhorn.yaml#L329

#### How to find `root-dir`?

**For RancherOS/CoreOS**
 
Run `ps aux | grep kubelet` and get argument `--root-dir` on host node. 

e.g.
```
$ ps aux | grep kubelet
root      3755  4.4  2.9 744404 120020 ?       Ssl  00:45   0:02 kubelet --root-dir=/opt/rke/var/lib/kubelet --volume-plugin-dir=/var/lib/kubelet/volumeplugins
```
You will find `root-dir` in the cmdline of proc `kubelet`. If it's not set, the default value `/var/lib/kubelet` would be used. In the case of RancherOS/CoreOS, the root-dir would be `/opt/rke/var/lib/kubelet` as shown above.

If kubelet is using a configuration file, you would need to check the configuration file to locate the `root-dir` parameter.

**For K3S v0.10.0-**

Run `ps aux | grep k3s` and get argument `--data-dir` or `-d` on k3s server node.

e.g.
```
$ ps uax | grep k3s
root      4160  0.0  0.0  51420  3948 pts/0    S+   00:55   0:00 sudo /usr/local/bin/k3s server --data-dir /opt/test/k3s/data/dir
root      4161 49.0  4.0 259204 164292 pts/0   Sl+  00:55   0:04 /usr/local/bin/k3s server --data-dir /opt/test/k3s/data/dir
``` 
You will find `data-dir` in the cmdline of proc `k3s`. By default it is not set and `/var/lib/rancher/k3s` will be used. Then joining `data-dir` with `/agent/kubelet` you will get the `root-dir`. So the default `root-dir` for K3S is `/var/lib/rancher/k3s/agent/kubelet`.

If K3S is using a configuration file, you would need to check the configuration file to locate the `data-dir` parameter.

**For K3S v0.10.0+**

It is always `/var/lib/kubelet`

## Background 
#### CSI doesn't work with RancherOS/CoreOS + RKE before Longhorn v0.4.1 
The reason is:

1. RKE sets argument `root-dir=/opt/rke/var/lib/kubelet` for kubelet in the case of RancherOS or CoreOS, which is different from the default value `/var/lib/kubelet`.
                                                                             
2. **For k8s v1.12+**

     Kubelet will detect the `csi.sock` according to argument `<--kubelet-registration-path>` passed in by Kubernetes CSI driver-registrar, and `<drivername>-reg.sock` (for Longhorn, it's `io.rancher.longhorn-reg.sock`) on kubelet path `<root-dir>/plugins`.
   
   **For k8s v1.11**
   
     Kubelet will find both sockets on kubelet path `/var/lib/kubelet/plugins`.
   
3. By default, Longhorn CSI driver create and expose these 2 sock files on host path `/var/lib/kubelet/plugins`.

4. Then kubelet cannot find `<drivername>-reg.sock`, so CSI driver doesn't work.

5. Furthermore, kubelet will instruct CSI plugin to mount Longhorn volume on `<root-dir>/pods/<pod-name>/volumes/kubernetes.io~csi/<volume-name>/mount`.

   But this path inside CSI plugin container won't be binded mount on host path. And the mount operation for Longhorn volume is meaningless.
   
   Hence Kubernetes cannot connect to Longhorn using CSI driver.
   
#### Longhorn v0.7.0- doesn't work on K3S v0.10.0+
K3S now sets its kubelet directory to `/var/lib/kubelet`. See [the K3S release comment](https://github.com/rancher/k3s/releases/tag/v0.10.0) for details.

## Reference
https://github.com/kubernetes-csi/driver-registrar

https://coreos.com/os/docs/latest/iscsi.html
