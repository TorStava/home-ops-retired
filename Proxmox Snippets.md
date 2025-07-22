# Proxmox Snippets

I haven't automated the creation and configuration of the VM's, but these snippets makes it very fast to spin up the wanted number of VM's with a defined configuration.



## Creating Talos template

## Set environment variables
Note that memory and CPU will be updated later so we can set them to low values in the template.
```shell
export TEMPLATE_ID="999"
export IMAGE="metal-amd64.iso"
export TEMPLATE_NAME="talos-template"
export MEMORY="2048"
export CPU_CORES="1"
```

## Download Talos
Download the Talos ISO image.

Use the [Talos Linux Image Factory](https://factory.talos.dev/) tool to generate the download link.

Select the qemu-guest-agent system extension if using the image with Proxmox.

```shell
wget https://factory.talos.dev/image/ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515/v1.10.4/$IMAGE
```

Upload to proxmox using the GUI.

## Create template
Not really necessary, but I like to create a template that can be cloned. Alternatively, the VM's could just be created directly in a loop.

```shell
qm create $TEMPLATE_ID \
 --memory $MEMORY \
 --cpu cputype=host \
 --core $CPU_CORES \
 --name $TEMPLATE_NAME \
 --net0 virtio,bridge=vmbr0 \
 --vga type=virtio \
 --scsihw virtio-scsi-single \
 --scsi0 local-zfs:20,iothread=1,ssd=1,discard=on \
 --ide2 local:iso/$IMAGE,media=cdrom \
 --boot c \
 --agent 1
```

Convert to template
```shell
qm template $TEMPLATE_ID
```

## Clone the template to create VM's

First, create three master nodes. Edit the settings as needed.

```shell
for i in $(seq 1 3);
do
qm clone 999 8$((i))0 --name k3s-master-$i --full
qm disk resize 8$((i))0 scsi0 100G
qm set 8$((i))0 --memory 16384 --cores 4
qm set 8$((i))0 --onboot 1 --startup order=8$((i))0
done
```

The, create three worker nodes. Edit settings as needed.

```shell
for i in $(seq 1 3);
do
qm clone 999 8$((i+3))0 --name k3s-worker-$i --full
qm disk resize 8$((i+3))0 scsi0 100G
qm set 8$((i+3))0 --memory 65536 --cores 8
qm set 8$((i+3))0 --onboot 1 --startup order=8$((i+3))0
done
```

Finally, start the VM's.

```shell
for i in $(seq 1 6);
do
qm start 8$((i))0
done
```

## Other useful snippets

Stop all VM's
```shell
for i in $(seq 1 6); do
qm stop 8$((i))0
done
```

!!!DANGER!!! Destroy all VM's
```shell
for i in $(seq 1 6);
do
qm destroy 8$((i))0
done
```

Remove the disk from all VM's (useful during testing). The disk needs to be manually deleted in the Proxmox GUI since it's only the link to the disk that gets removed, not the disk itself.
```shell
for i in $(seq 1 6);
do
qm set 8$((i))0 --delete scsi0
done
```

Create new disk for each VM. Edit size as needed, leave the rest of the parameters as is.
```shell
for i in $(seq 1 6);
do
qm set 8$((i))0 --scsi0 local-zfs:256,iothread=1,ssd=1,discard=on
done
```

Add bridge networks.
```shell
for i in $(seq 1 6);
do
qm set 8$((i))0 --net1 virtio,bridge=vmbr1
qm set 8$((i))0 --net2 virtio,bridge=vmbr2
qm set 8$((i))0 --net3 virtio,bridge=vmbr3
done
```

## Passthrough physical disks
Find disk id's on the Proxmox host:

```shell
ls -l /dev/disk/by-id/
```

Pass through disk to VM
```shell
qm set 860 --scsi1 /dev/disk/by-id/ata-VK000480GWSRR_S4NANA0R806668
```

Note that the disk in the VM will not have the original name, but something like:
```
scsi-0QEMU_QEMU_HARDDISK_drive-scsi1
scsi-0QEMU_QEMU_HARDDISK_drive-scsi1-part1
```

You can check the recognized disks on the Talos nodes by running `talosctl list /dev/disk/by-id `.


## Notes
Tried to set static IP's using this method, but this only works when using bootimg.

Ended up setting static IP reservations in the DHCP server instead for the initial bootup.

```shell
for i in $(seq 1 6);
do
qm set 8$((i))0 --ipconfig0 ip=192.168.1.8$i/24,gw=192.168.1.1
qm set 8$((i))0 --nameserver "1.1.1.1 9.9.9.9" --searchdomain ""
done
```
