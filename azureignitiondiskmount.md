# Mounting disks to /var/lib/containers on RHCOS on Azure using ignition

This will show the steps needed to add additional disks to an Azure VM, format the appropriate disk, and mount the disk to /var/lib/containers on startup

## Add disks to VMs

edit the master and worker ARM templates to add additional data disks-
This will create one disk to mount to /var/lib/containers and will leave an additional disk dedicated to CephFS on workers. Masters will only need the container disk.

```
        "storageProfile": {
          "imageReference": {
            "id": "[resourceId('Microsoft.Compute/images', variables('imageName'))]"
          },
          "osDisk": {
            "name": "[concat(variables('vmName'),'_OSDisk')]",
            "osType": "Linux",
            "createOption": "FromImage",
            "managedDisk": {
              "storageAccountType": "Premium_LRS"
            },
            "diskSizeGB": 32
          },
          "dataDisks" : [
              {
                  "lun": 0,
                  "name": "[concat(variables('vmName'),'_ContainerDisk')]",
                  "createOption": "empty",
                  "diskSizeGB": 128
              },
              {
                  "lun": 1,
                  "name": "[concat(variables('vmName'),'_CephDisk')]",
                  "createOption": "empty",
                  "diskSizeGB": 1024
              }
          ]
        },

```

## Mounting disk to /var/lib/containers using ignition

Create the following .fcc file to add some systemd services to ignition to automatically mount our additional disk

```yaml
variant: fcos
version: 1.0.0
systemd:
  units:

  - contents: |
    [Unit]
    Description=Make and label container file sytem

    [Service]
    Type=oneshot
    RemainAfterExit=yes
    ExecStart=/usr/local/bin/mountscript.sh
    ExecStart=/bin/udevadm trigger --type=devices --action=change
    TimeoutSec=0

    [Install]
    WantedBy=var-lib-containers.mount
    enabled: true
    name: makecontainerfs.service

  - contents: |
    [Unit]
    Description=Mount /dev/disk/by-label/containerfs to /var/lib/containers
    Before=local-fs.target
    Requires=makecontainerfs.service
    After=makecontainerfs.service

    [Mount]
    What=/dev/disk/by-label/containerfs
    Where=/var/lib/containers
    Type=ext4
    Options=defaults

    [Install]
    WantedBy=local-fs.target
    enabled: true
    name: var-lib-containers.mount

  - contents: |
    [Unit]
    Description=Restore recursive SELinux security contexts
    DefaultDependencies=no
    After=var-lib-containers.mount
    Before=shutdown.target

    [Service]
    Type=oneshot
    RemainAfterExit=yes
    ExecStart=/sbin/restorecon -R /var/lib/containers/
    TimeoutSec=0

    [Install]
    WantedBy=multi-user.target
    enabled: true
    name: restorecon-var-lib-containers.service
```

```bash
docker run -i --rm quay.io/coreos/fcct:release --pretty --strict < ex.fcc > transpiled_config.ign
```

Splice the systemd units into master and worker .ign files

Next we will create a script to identify, format, and label the appropriate disk for the container filesystem. This script works by identifying the smallest unformatted disk attached to the VM. If you are using CephFS- it will leave your additional larger disk unformatted for Ceph to consume.

Create a file mountscript.sh-

```bash
#!/bin/sh
if test -f "/dev/disk/by-label/containerfs"; then
    exit 0
fi
VOLUME=$(lsblk -J -b | jq -r '[ .blockdevices[] | select(.type=="disk") | select(.children==null) | .size |= tonumber ] | sort_by(.size) | .[0].name')
chcon -t fixed_disk_device_t /dev/${VOLUME}
rm -rf /var/lib/containers
mkfs.ext4 /dev/${VOLUME}
e2label /dev/${VOLUME} containerfs
udevadm trigger
```

Base64 encode it

```bash
cat mountscript.sh | base64 -w0
```

and add the following entry to your worker and master .ign files

```json
  "storage": {
    "files": [
      {
        "filesystem": "root",
        "path": "/usr/local/bin/mountscript.sh",
        "user": {
          "name": "root"
        },
        "contents": {
          "source": <insert-base64-encoded-mountscript>,
          "verification": {}
        },
        "mode": 365
      },
```



## Verify it's working

SSH into worker and master nodes and run

```bash
lsblk -f
```

and you should see something like this

```
NAME          FSTYPE      LABEL             UUID                                 MOUNTPOINT

sdc           ext4        containerfs       abe69268-0540-47c7-af1c-89a5f0ca58d8 /var/lib/containers

```

## troubleshoot
```bash
journalctl -b -f u makecontainerfs.service -u var-lib-containers.mount -u restorecon-var-lib-containers.service
```
