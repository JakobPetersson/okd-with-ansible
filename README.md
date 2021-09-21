# OKD with ansible

This repository holds ansible files for installing
OKD using Ansible on hardware.

[https://docs.okd.io/latest/installing/installing_bare_metal/installing-bare-metal.html]
{Installing a user-provisioned cluster on bare metal}

It assumes you have the following hardware:
1. Raspberry PI for infrastructure (nginx load balancing
   and serving the ignition files)
2. An intel machine that will first serve as bootstrap and then later worker1.
3. 3 master intel machines.

If you have more hardware, adjust the hosts file accordingly.

# Preparation
1. Pull your RedHat pull secret and place in file "pull-secret".
2. [If you want github integration](https://docs.openshift.com/container-platform/4.8/authentication/identity_providers/configuring-github-identity-provider.html),
   create a file called "github-config.json" with similar content as this:
   ```json
   {
     "clientID": "< github client ID >",
     "clientSecret": "< github client secret >",
     "organizations": [
       "< your github organization >"
     ]
   }
   ```
3. Set up DNS with the following entries (of course, adjust
   addresses to your infrastructure):

   | hostname                       | Address        |
   |--------------------------------|----------------| 
   | infra1.ocp4.example.com        | 192.168.60.180 |
   | api-int.ocp4.example.com       | CNAME infra1.ocp4.example.com |
   | api.ocp4.example.com           | CNAME infra1.ocp4.example.com |
   | apps.ocp4.example.com          | CNAME infra1.ocp4.example.com |
   | *.apps.ocp4.example.com        | CNAME infra1.ocp4.example.com |
   | master1.ocp4.example.com       | 192.168.60.181 |
   | master2.ocp4.example.com	    | 192.168.60.182 |
   | master3.ocp4.example.com	    | 192.168.60.183 |
   | worker1.ocp4.example.com       | 192.168.60.184 |

   If your DNS (like mine) cannot handle wildcards, add these entries as CNAME, pointing to app.ocp4.example.com:

   - alertmanager-main-openshift-monitoring.apps.ocp4.example.com 
   - canary-openshift-ingress-canary.apps.ocp4.example.com
   - console-openshift-console.apps.ocp4.example.com
   - downloads-openshift-console.apps.ocp4.example.com
   - grafana-openshift-monitoring.apps.ocp4.example.com
   - oauth-openshift.apps.ocp4.example.com
   - prometheus-k8s-openshift-monitoring.apps.ocp4.example.com
   - thanos-querier-openshift-monitoring.apps.ocp4.example.com

4. Create a new image for the rasperry pi with enabled ssh and boot it up.
5. Create a bootable USB from the latest version of Fedora coreos. 
   (At the time of writing, the latest working release is [34.20210529.3.0](https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20210529.3.0/x86_64/fedora-coreos-34.20210529.3.0-live.x86_64.iso)
6. Set up static DHCP entries for the machines, matching IP addresses above.
7. Make sure there are no partitions on the bootstrap and masters

```
$ sudo fdisk /dev/sda

Command (m for help): p
Disk /dev/sda: 1.82 TiB, 2000398934016 bytes, 3907029168 sectors
Disk model: ST2000LM015-2E81
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: 1B39BA89-A38F-4F17-94C4-995DC652412E

Device       Start        End    Sectors  Size Type
/dev/sda1     2048       4095       2048    1M BIOS boot
/dev/sda2     4096     264191     260096  127M EFI System
/dev/sda3   264192    1050623     786432  384M Linux filesystem
/dev/sda4  1050624 3907029134 3905978511  1.8T Linux filesystem

Command (m for help): d
Partition number (1-4, default 4): 

Partition 4 has been deleted.

Command (m for help): d
Partition number (1-3, default 3): 3

Partition 3 has been deleted.

Command (m for help): d
Partition number (1,2, default 2): 2

Partition 2 has been deleted.

Command (m for help): d
Selected partition 1
Partition 1 has been deleted.

Command (m for help): w
The partition table has been altered.
Failed to remove partition 3 from system: Device or resource busy
Failed to remove partition 4 from system: Device or resource busy

The kernel still uses the old partitions. The new table will be used at the next reboot. 
Syncing disks.
```

# Run playbook
It's time to run the playbook. There are a number of steps
that will be completed:
1. Cluster configuration will be created and added into
   ignition files.
2. The infrastructure node (RPI) will be setup.

This is how to run the playbook:
```shell
ansible-playbook -i hosts -v deploy-okd.yml
```

Once the playbook tell you to, boot the masters on
Fedora coreos USB and start the installation process. 
The NUCs are a bit slow on the network side so a number of kernel arguments are needed.
```shell
$ sudo coreos-installer \
    install \
    /dev/sda \
    --firstboot-args='console=tty0 rd.neednet=1 rd.net.timeout.carrier=30' \
    --insecure-ignition \
    --ignition-url=http://infra1.ocp4.example.com:8080/master[1-3].ign
```

Once the installation has finished, remove the USB and reboot.
```shell
$ sudo reboot now
```

Verify that the master is trying to pull the secondary ignition from api-int.ocp4.example.com:22623.

Once all masters are waiting for the secondary ignition, continue the playbook
which tell you to boot the first worker machine on
Fedora coreos USB and start the installation for the bootstrap process:
```shell
$ sudo coreos-installer \
    install \
    /dev/sda \
    --firstboot-args='console=tty0 rd.neednet=1 rd.net.timeout.carrier=30' \
    --insecure-ignition \
    --ignition-url=http://infra1.ocp4.example.com:8080/bootstrap.ign
```

Once the installation has finished, remove the USB and reboot.
```shell
$ sudo reboot now
```

After some time, you will be able to login via ssh to the bootstrap machine
and follow the installation:

```shell
$ ssh core@nucbootstrap.ocp4.example.com
The authenticity of host 'nucbootstrap.ocp4.example.com (192.168.60.184)' can't be established.
ECDSA key fingerprint is SHA256:Z3edOf5ImnxO/x9tchkto5LoEQIaFm8DT/7zyGj5r6g.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'nucbootstrap.ocp4.example.com,192.168.60.183' (ECDSA) to the list of known hosts.
This is the bootstrap node; it will be destroyed when the master is fully up.

The primary services are release-image.service followed by bootkube.service. To watch their status, run e.g.

  journalctl -b -f -u release-image.service -u bootkube.service
Fedora CoreOS 34
Tracker: https://github.com/coreos/fedora-coreos-tracker
Discuss: https://discussion.fedoraproject.org/c/server/coreos/

Last login: Sat Sep  4 05:55:40 2021 from 192.168.40.50
[core@nucbootstrap ~]$ journalctl -b -f -u release-image.service -u bootkube.service
```

When the installation has started, continue the playbook.
When the playbook detects that the installation is finished, the playbook will continue with post-installation configuration.

You can now open the cluster console by
opening https://console-openshift-console.apps.ocp4.example.com.

Have fun with your cluster!

## Adding the bootstrap node as a worker

Once the cluster has been correctly installed, shutdown the
bootstrap node, remove the partitions and reinstall using
this command:

```shell
$ sudo coreos-installer \
    install \
    /dev/sda \
    --firstboot-args='console=tty0 rd.neednet=1 rd.net.timeout.carrier=30' \
    --insecure-ignition \
    --ignition-url=http://infra1.ocp4.example.com:8080/worker[1].ign
```

Since this has not been prepared in the cluster earlier,
you need to approve the certificate requests.
This is how you list and approve them:

```shell
$ KUBECONFIG=./openshift-files/auth/kubeconfig \
    ./openshift-client/oc get csr | grep -i pending
NAME        AGE     SIGNERNAME                                    REQUESTOR                                                                   CONDITION
csr-4n948   36m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-7n8zl   51m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-c8nhz   20m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-f4vvb   5m41s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-fz8sk   66m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending

$ KUBECONFIG=./openshift-files/auth/kubeconfig \
    ./openshift-client/oc adm certificate approve csr-fz8sk csr-wq7kq
certificatesigningrequest.certificates.k8s.io/csr-fz8sk approved
certificatesigningrequest.certificates.k8s.io/csr-wq7kq approved

$ KUBECONFIG=./openshift-files/auth/kubeconfig \
    ./openshift-client/oc get nodes
NAME                       STATUS     ROLES           AGE    VERSION
master0.ocp4.example.com   Ready      master,worker   116m   v1.20.0+01994f4-1091
master1.ocp4.example.com   Ready      master,worker   116m   v1.20.0+01994f4-1091
master2.ocp4.example.com   Ready      master,worker   113m   v1.20.0+01994f4-1091
worker1.ocp4.example.com   NotReady   worker          73s    v1.20.0+01994f4-1091
```
