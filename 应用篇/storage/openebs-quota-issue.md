# Issue
when we using openebs version 4.1.0 to providing local PV hostpath feature, we found the actual use of storage is out of control.

# pre-requi
1. xfprog is installed on OS
   ```bash
   rpm -qa | grep xfsprog
   ```
   result should should the package installed
2. openebs base path /var/openebs/local should be mounted with pqota option

```bash
df -Th /var/openebs/local
mount | grep "openebs"
# result should be: /dev/xxx on /var/openebs/local type xfs (rw,relate, seclable,attr2, inode64,logbufs=8,logbsize=32k, noquota)
```
if you found there have noquota on mount option, please umount and mount it again.
"pquota" is not usable with remount mount option.
```bash
umount /dev/xxx/openebs
mount -o rw,pquota /dev/xxxx/mount
# this time will show there have prjquota on mount result
cat /etc/fstab | grep openebs
# this result should should with pquota too, like: /dev/xxx/openebs /var/openebs/local xfs defaults, pquota 0 0 
```

3. openebs helm chart value
under localpv-provisioner, set hostpathClass.xfsQuota.enabled to true. This will carry the xfsQuota to default storage class in cas.openebs.io/config annotation.

softLimitGrace and hardLimitGrace are used in conjunction with the PV storage request capacity to determine the soft and hard limits quota.
```yaml
localpv-provisioner:
  hostpathClass:
    xfsQuota:
      enabled: true
      softLimitGrace: "0%"
      hardLimitGrace: "0%"
```

5. verify creating Storageclass
```bash
kubectl apply -f -<<EOF

EOF

```
