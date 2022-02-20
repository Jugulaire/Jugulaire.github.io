---
layout: post
title: Cloud-init avec KVM
tags: [Mermaid]
mermaid: true
---

![cloud-init]({{ site.baseurl }}/images/cloud-init-1200.png){:class="img-responsive"}

## Prérequis 

- Avoir KVM de déployé sur votre machine 
- ```sudo apt-get install -y cloud-image-utils```

## Création de l'image 

Téléchargement de la version cloud-init de Debian : 

```bash
Jugu@X220:~/Téléchargements$ wget https://cloud.debian.org/images/cloud/buster/20210621-680/debian-10-generic-amd64-20210621-680.qcow2 -O debian-10-cloud.qcow2
```

Commit de l'image pour l'agrandir au passage et ne pas "casser" l'image de base : 

```bash
jugu@X220:~/Téléchargements$ qemu-img create -b debian-10-cloud.qcow2 -f qcow2 -F qcow2 snapshot-debian-10-20G.qcow2 20G
```

Verif des infos du snapshot : 

```bash
jugu@X220:~/Téléchargements$ qemu-img info snapshot-debian-10-20G.qcow2 
image: snapshot-debian-10-20G.qcow2
file format: qcow2
virtual size: 20G (21474836480 bytes)
disk size: 196K
cluster_size: 65536
backing file: debian-10-cloud.qcow2
backing file format: qcow2
Format specific information:
    compat: 1.1
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
```

## (si ce n'est pas déjà fait) Générer une clé SSH

```bash
ssh-keygen -t rsa -b 4096 -f id_rsa -C test1 -N "" -q
```

## Création du fichier de config 

Dans un fichier `cloud_init.cfg` : 

```yaml
#cloud-config
hostname: Node1
fqdn: Node1.kube.test
manage_etc_hosts: true
users:
  - name: kube
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: users, admin
    home: /home/kube
    shell: /bin/bash
    lock_passwd: false
    ssh-authorized-keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDxZV0uayWSqDMmFp2s4Eg+mWO2efwQLNCnauNMGwK166DskcvHcarEpKLObxlnpZd/bBLfxsMp9a8DC2cI/Y77IVF1C2WWOeNHZFIVlkv6zX9CYEN9dBv8tyuu9kJ8CauQdgSGVtB40W9zK44tJHqH21sL3JoRI/OBV8ITBA7jDp4NXJWjhq8R/lkSd/CDNZ06EVl1obMyqfd0f8TP6JU4EBl3UEi2RVpVzEtg9qgEVUXWihlLss2kUtHuYLRsAqEQfJq41KXNQjwfaF3TF32c7lJqCBwGFl9l0YnSd4k0vGuRVEbif++algFS4FSolIiXZa60oT6TbGd6WcG5JV0E6uaeTCR8R6drHIiVd34tmhmztY9f+HuikJJIxaMTaOpOu1pOx1Erw3NF1fVNK+iAxcNFcKc9JomMssL6SVaL+nxZlc9IDW32UuymJk3FV7F/dUtDBs7GwY2vdmk4nhu4PCXHDhhKiOMaJGi8ruykVo76ch4neXhbxXgE5Iot9xc1DDUq0NMcDO5GGuE5HyQq8dbov794FRzTKDhmI1UCzQOPtn2YaAzDrrg4aCj7BSyZz6fA0tg0WXC/+KhGYCWSzwMybkpRgPZZi24LS9NEFMt0SE/8rrKh+QQBYBQEbACwwsmMTE8Es9V0whr4YNzUsPJVpr8DI3rHHMniS5igRQ== test1
# only cert auth via ssh (console access can still login)
ssh_pwauth: false
disable_root: false
chpasswd:
  list: |
     kube:kube
  expire: False

package_update: true
packages:
  - qemu-guest-agent
# written to /var/log/cloud-init-output.log
final_message: "The system is finally up, after $UPTIME seconds"
```

### Pour scripter le remplacement de la clé ssh :

```bash
pubkey=$(cat id_rsa.pub)
sed "s%      - <sshPUBKEY>%      - $pubkey%" cloud_init.cfg
```

## Config du réseau 

Dans `network_config_static.cfg` :

```yaml
version: 2
ethernets:
  ens2:
     dhcp4: false
     # default libvirt network
     addresses: [ 192.168.100.10/24 ]
     gateway4: 192.168.100.1
     nameservers:
       addresses: [ 192.168.100.1,8.8.8.8 ]
       search: [ kube.test ]
```

> Mon reseau par défaut sous KVM est modifié, vérifiez le votre.

## Ajout des métadonnées dans une image de seed 

```bash
# Création de l'image
jugu@X220:~/Téléchargements/cloud-init$ cloud-localds -v --network-config=network_config_static.cfg seed_test.img cloud_init.cfg

wrote seed_test.img with filesystem=iso9660 and diskformat=raw

# Affichage des infos de l'image
jugu@X220:~/Téléchargements/cloud-init$ qemu-img info seed_test.img 
image: seed_test.img
file format: raw
virtual size: 368K (376832 bytes)
disk size: 368K
```

## Lancement de la VM avec virt-install 

```bash
virt-install --name Kube_Node1 \
  --virt-type kvm --memory 2048 --vcpus 2 \
  --boot hd,menu=on \
  --disk path=seed_test.img,device=cdrom \
  --disk path=snapshot-debian-10-20G.qcow2_node1,device=disk \
  --graphics none \
  --os-type Linux --os-variant debian9 \
  --network network:new_default \
  --console pty,target_type=serial 
```

### Même chose sans console (pour du script)

```bash
virt-install --name Kube_Node1 \
  --virt-type kvm --memory 2048 --vcpus 2 \
  --boot hd,menu=on \
  --disk path=seed_test.img,device=cdrom \
  --disk path=snapshot-debian-10-20G_node1.qcow2,device=disk \
  --graphics none \
  --os-type Linux --os-variant debian9 \
  --network network:new_default \
  --noautoconsole
```

### trouver le bon os-variant : 

```bash
sudo apt install libosinfo-bin
osinfo-query os | grep [ton-OS]
```

### pousser nos machines dans l'inventory 

```bash
[debian]
192.168.100.11
192.168.100.12
192.168.100.13


[debian:vars]
ansible_ssh_user=kube
ansible_ssh_private_key_file=/home/jugu/cloud-init/id_rsa
```

## Gerer les VM avec virsh 

Pour couper une vm, lister les machines :

```bash

jugu@X220:~/cloud-init$ virsh list --all
 Id    Name                           State
----------------------------------------------------
 4     Kube_Node1                     running
 5     Kube_Node2                     running
 6     Kube_Node3                     running
 -     Kali                           shut off
 -     Nixos                          shut off
 -     ubuntu11.10                    shut off
 -     win10                          shut off

```

Puis la couper avec un `shutdown`: 

```bash
jugu@X220:~/cloud-init$ virsh shutdown Kube_Node1
```

> Note : Il faut couper les vm avant de les supprimer

Supprimer une VM 

```bash
virsh undefine Kube_Node3
```

## Le projet Gitlab 

[ici](https://gitlab.com/P0pR0cK5/cloud-init)

## Sources 

[KVM: Testing cloud-init locally using KVM for an Ubuntu cloud image](https://fabianlee.org/2020/02/23/kvm-testing-cloud-init-locally-using-kvm-for-an-ubuntu-cloud-image/)

[Introduction à Cloud-init](https://www.grottedubarbu.fr/introduction-cloud-init/)

[QEMU:Documentation/CreateSnapshot](https://wiki.qemu.org/Documentation/CreateSnapshot)

[An Introduction to Cloud-Config Scripting](https://www.digitalocean.com/community/tutorials/an-introduction-to-cloud-config-scripting)

[Goffinet et ses scripts](https://github.com/goffinet/virt-scripts)

## Script de création auto 

```bash
#!/bin/bash

nbVM=3
ram=512
CPU=2
HDD="20G"

#create folder to store config 
echo "Create folder to store cloud-init stuff"
if [ -d "cloud_init.d" ]
then
	echo "Folder exist"
else

	mkdir cloud_init.d
	if [ "$?" = "0" ] ; then
		echo "Folder OK !"
	else
		echo "Folder Fucked up ! BRUH" 1>&2
		exit 1
	fi
fi
# creating folder to store img files
echo "Create folder to store IMG files"
if [ -d "disks_images.d" ]
then
	echo "Folder exist"
else

	mkdir "disks_images.d"
	if [ "$?" = "0" ]; then
        	echo "Folder OK !"
	else
        	echo "Folder Fucked up ! BRUH" 1>&2
        	exit 1
	fi
fi

for idVM in $(seq 1 $nbVM)
do
	#Start of the loop
	echo "Building VM $idVM"
	#creating the image 
	echo "Creating image.."
	cd disks_images.d
	qemu-img create -b ../debian-10-cloud.qcow2 -f qcow2 -F qcow2 snapshot-debian-10-20G_n0de$idVM.qcow2 $HDD
	if [ "$?" = "0" ]; then
  		echo "Image done !"
	else
  		echo "Cannot create image !" 1>&2
  		exit 1
	fi
	#modify the cloud init vars
	echo "generating cloud-init files"
	#change hostname 
	cd ../cloud_init.d
	sed "s/<HOSTNAME>/Node$idVM/g" ../cloud_init.cfg > cloud_init_Node$idVM.cfg
	if [ "$?" = "0" ]; then
  		echo "setting Hostname OK !"
	else
  		echo "Cannot set hostname !" 1>&2
  		exit 1
	fi
	#set ip address 
	sed "s/<ID>/1$idVM/g" ../network_config_static.cfg > network_config_static_Node$idVM.cfg
	if [ "$?" = "0" ]; then
  		echo "setting IP OK !"
	else
  		echo "Cannot set IP !" 1>&2
  		exit 1
	fi
	cd ..
	# create cloud init image
	cloud-localds -v --network-config=cloud_init.d/network_config_static_Node$idVM.cfg disks_images.d/seed_test_Node$idVM.img cloud_init.d/cloud_init_Node$idVM.cfg
	if [ "$?" = "0" ]; then
                echo "Cloud init image DONE"
        else
                echo "Cloud init image failed ! Bruh !" 1>&2
                exit 1
        fi
	# Install the VM 
	virt-install --name Kube_Node$idVM --virt-type kvm --memory $ram --vcpus $CPU --boot hd,menu=on --disk path=disks_images.d/seed_test_Node$idVM.img,device=cdrom --disk path=disks_images.d/snapshot-debian-10-20G_n0de$idVM.qcow2,device=disk --graphics none --os-type Linux --os-variant debian9 --network network:new_default --console pty,target_type=serial --noautoconsole 
	if [ "$?" = "0" ]; then
                echo "VM created !"
        else
                echo "VM Fucked Up !" 1>&2
                exit 1
        fi
done
```
