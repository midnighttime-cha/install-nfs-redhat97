# การติดตั้ง HA-NFS Cluster ด้วย DRBD บน RHEL 9
จะเป็นอธิบายขั้นตอนการตั้งค่า High Availability NFS ระหว่างโหนด nfs01 และ nfs02 โดยใช้ DRBD ในการ Sync ข้อมูล และ Pacemaker ในการควบคุม Failover

## 1. การเตรียมระบบ (ทำทั้ง 2 เครื่อง)

1.1. ตั้งค่า Hostname และ /etc/hosts
```bash
# บน nfs01
hostnamectl set-hostname nfs01
# บน nfs02
hostnamectl set-hostname nfs02

# แก้ไข /etc/hosts ทั้งสองเครื่อง
cat >> /etc/hosts <<EOF
192.168.1.20 nfs01
192.168.1.21 nfs02
EOF
```

1.2. ปิด SELinux และ Firewall
```bash
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

firewall-cmd --permanent --add-service=high-availability
firewall-cmd --permanent --add-service=nfs
firewall-cmd --permanent --add-port=7788/tcp # DRBD
firewall-cmd --reload
```

## 2. ติดตั้งและตั้งค่า DRBD (ทำทั้ง 2 เครื่อง)

2.1 ติดตั้ง Packages
```bash
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
dnf install -y https://www.elrepo.org/elrepo-release-9.el9.elrepo.noarch.rpm
dnf config-manager --set-enabled elrepo-extras
dnf config-manager --set-enabled elrepo-kernel
dnf install -y drbd9x-utils kmod-drbd9x
```

2.2 ตรวจสอบ Module และไฟล์คอนฟิก
```bash
modprobe drbd
lsmod | grep drbd
cat /etc/drbd.conf
```

2.3 กำหนดค่า Resource
```bash
cat > /etc/drbd.d/nfsdata.res <<EOF
resource nfsdata {
    protocol C;
    device /dev/drbd0;
    meta-disk internal;
    on nfs01 {
        address 192.168.1.20:7788;
        disk /dev/mapper/rhel-nfs_data;
    }
    on nfs02 {
        address 192.168.1.21:7788;
        disk /dev/mapper/rhel-nfs_data;
    }
}
EOF
```

2.4 เริ่มการทำงาน (Initial Sync) ทำทั้ง 2 เครื่อง:
```bash
drbdadm down nfsdata

wipefs -a /dev/mapper/rhel-nfs_data

drbdadm create-md nfsdata (พิมพ์ yes)

drbdadm up nfsdata
```
เฉพาะบน nfs01 (บังคับ Primary ครั้งแรก):
```bash
drbdadm primary --force nfsdata
mkfs.xfs /dev/drbd0
```

## 3. ติดตั้ง Cluster Stack (ทำทั้ง 2 เครื่อง)
```bash
subscription-manager repos --enable=rhel-9-for-x86_64-highavailability-rpms
dnf install -y pacemaker pcs resource-agents
systemctl enable --now pcsd
echo "Cluster@2024!" | passwd --stdin hacluster
```

## 4. กำหนดค่า Cluster (ทำจาก nfs01)
```bash
pcs host auth nfs01 nfs02 -u hacluster -p Cluster@2024!
pcs cluster setup nfscluster nfs01 nfs02 --force
pcs cluster start --all
pcs cluster enable --all
pcs property set stonith-enabled=false
pcs property set no-quorum-policy=ignore
```

## 5. การสร้าง Resource (Final Fix)

5.1 ล้างสถานะ Error และสร้าง DRBD Promotable
```bash
pcs resource cleanup
pcs resource delete nfs_drbd --force
pcs resource create nfs_drbd ocf:linbit:drbd drbd_resource=nfsdata op monitor interval=20s
pcs resource promotable nfs_drbd promoted-max=1 promoted-node-max=1 clone-max=2 clone-node-max=1 meta notify=true
```

5.2 สร้าง NFS Group
`หมายเหตุ: ใช้ IP 192.168.1.100 เพื่อให้อยู่ในวงเดียวกับ Host`
```bash
pcs resource delete g_nfs --force
mkdir -p /mnt/nfs_share

pcs resource create nfs_fs ocf:heartbeat:Filesystem device="/dev/drbd0" directory="/mnt/nfs_share" fstype="xfs"
pcs resource create nfs_ip ocf:heartbeat:IPaddr2 ip=192.168.1.100 cidr_netmask=24
pcs resource create nfs_server ocf:heartbeat:nfsserver nfs_shared_infodir="/mnt/nfs_share/nfsinfo" nfs_no_notify=true

pcs resource group add g_nfs nfs_fs nfs_ip nfs_server
```

5.3 ตั้งค่า Constraints
```bash
pcs constraint colocation add g_nfs with promoted nfs_drbd-clone INFINITY
pcs constraint order promote nfs_drbd-clone then start g_nfs
```

## 6. การตรวจสอบสถานะที่ถูกต้อง ก่อนใช้งานจริง ตรวจสอบให้แน่ใจว่า:
- DRBD Sync เสร็จสิ้น: drbdadm status ต้องแสดง disk:UpToDate ทั้งสองฝั่ง และ replication:Established
- สถานะ Resource: pcs status ต้องไม่มี Failed Actions (หากมีให้รัน pcs resource cleanup)
- Mount Point: เครื่องที่เป็น Primary ต้องมองเห็น /mnt/nfs_share ผ่านคำสั่ง df -h
