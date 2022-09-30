# Creating a Local Ceph Cluster

The cinder and glance operators are able to configure services to use
Ceph for their backend storage, but this means developers need to
supply a Ceph cluster. Some people have been successful using running
the Ceph "demo" container, but others have had problems where the
monitor service gets stuck in the 'probing' state. As an alternative,
these instructions [^1] describe using *cephadm* to create a small cluster
that can be accessed by OpenShift.

The Ceph cluster requires a spare block device to use for the OSD. One
option is to run the Ceph cluster in a dedicated VM, and configure the
VM with a second virtual disk (e.g. /dev/vdb). The alternative described
here installs the cluster on the same hypervisor that hosts the CRC VM.
This isn't too invasive because *cephadm* runs the Ceph services all in
containers, and it's easy to uninstall everything.

# Prerequisites

Install *cephadm* from [CentOS CBS](https://cbs.centos.org/kojifiles/packages/cephadm/).
This command installs the current latest version (16.2.9) for CS9.

```sh
sudo dnf install -y https://cbs.centos.org/kojifiles/packages/cephadm/16.2.9/1.el9s/noarch/cephadm-16.2.9-1.el9s.noarch.rpm

```
The previous release (16.2.7) supports CS8. 

```sh
sudo dnf install -y https://cbs.centos.org/kojifiles/packages/cephadm/16.2.7/1.el8s/noarch/cephadm-16.2.7-1.el8s.noarch.rpm

```

Install a few more required packages:

```sh
for PKG in container-selinux podman podman-catatonit util-linux lvm2 jq; do
    rpm -q $PKG >/dev/null || sudo dnf install -y $PKG
done

```

# Prepare a Block Device

This step is optional if you are installing Ceph on a dedicated VM,
and can supply the VM with a second (/dev/vdb) disk. To install Ceph
on a system without a spare disk, we use LVM on a loopback device.

```sh
sudo dd if=/dev/zero of=/var/lib/ceph-osd.img bs=1 count=0 seek=10G
sudo losetup /dev/loop3 /var/lib/ceph-osd.img
sudo pvcreate /dev/loop3
sudo vgcreate ceph_vg /dev/loop3
sudo lvcreate -l 100%FREE -n ceph_lv ceph_vg

```

*lsblk* should show you have a ceph_vg-ceph_lv block device, which
can also be visible at /dev/ceph_vg/ceph_lv. Now, create a systemd
service to restore things on system restart.

```sh
cat << 'EOF' | sudo tee /etc/systemd/system/ceph-lvm-losetup.service
[Unit]
Description=Ceph LVM losetup
DefaultDependencies=no
Conflicts=umount.target
Requires=lvm2-monitor.service systemd-udev-settle.service
Before=local-fs.target umount.target
After=var.mount lvm2-monitor.service systemd-udev-settle.service

[Service]
Type=oneshot
ExecStart=/usr/bin/bash -c ' \
if [ -z "$(/sbin/losetup -j /var/lib/ceph-osd.img)" ]; then \
  /sbin/losetup -f /var/lib/ceph-osd.img ; \
  sleep 2 ; \
  vgchange -a y ceph_vg ; \
fi'
RemainAfterExit=yes

[Install]
WantedBy=local-fs-pre.target
EOF

sudo systemctl enable ceph-lvm-losetup.service
sudo systemctl start ceph-lvm-losetup.service

```

# Create the Ceph Cluster

Begin by bootstrapping the cluster, which gets the monitor running. We
seed it with an initial ceph.conf file in order to establish some
critical settings. Notice MON_IP is the hypervisor's magic CRC address.

```sh
MON_IP="192.168.130.1"

cat <<EOF > /tmp/initial_ceph.conf
[global]
osd pool default size = 1

[mon]
mon warn on pool no redundancy = false
EOF

sudo cephadm bootstrap \
    --mon-ip $MON_IP \
    --config /tmp/initial_ceph.conf \
    --log-to-file \
    --skip-prepare-host \
    --skip-dashboard \
    --skip-monitoring-stack \
    --allow-fqdn-hostname \
    --single-host-defaults

FSID=$(sudo cephadm ls | jq '.[]' | jq 'select(.name | test("^mon*")).fsid' | tr -d '"');
echo "FSID is $FSID"

```
If you want to (re)use a known FSID, you can add "--fsid $FSID" to the
bootstrap command.

Add an OSD to the cluster. This example uses the LVM loopback device.

```sh
OSDDEV=/dev/ceph_vg/ceph_lv

sudo cephadm shell -- ceph orch daemon add osd ${HOSTNAME}:${OSDDEV}
echo "Wating 30 seconds for the OSD to come up..."
sleep 30
sudo cephadm shell -- ceph -s 2>/dev/null

```

Create the pools used by OpenStack.

```sh
for POOL in backups volumes vms; do
    sudo cephadm shell -- ceph osd pool create $POOL
    sudo cephadm shell -- ceph osd pool application enable $POOL rbd
done
sudo cephadm shell -- ceph osd pool create images
sudo cephadm shell -- ceph osd pool application enable images rgw
sudo cephadm shell -- ceph df 2>/dev/null

```

Create the "openstack" client and its cephx key.

```sh
sudo cephadm shell -- ceph auth add client.openstack \
  mon 'allow r' \
  osd 'allow class-read object_prefix rbd_children, allow rwx pool=vms, allow rwx pool=volumes, allow rwx pool=images'
CEPHX=$(sudo cephadm shell -- ceph auth get client.openstack 2>/dev/null | awk '$1 == "key" {print $3}')
echo "client.openstack key is $CEPHX"

```

Alternatively, if you already have a favorite keyring file for the
openstack.client then you can import it.

```sh
sudo cephadm shell -m ceph.client.openstack.keyring -- ceph auth import -i /mnt/ceph.client.openstack.keyring

```

# Delete the Ceph Cluster

```sh
sudo cephadm rm-cluster --zap-osds --force --fsid $FSID

```


[^1]: These instructions are based on John Fulton's [ceph.sh script](https://github.com/fultonj/zed/blob/main/standalone/ceph.sh).
