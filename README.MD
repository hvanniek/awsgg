# OSTree Greengrass setup

## Background

This repo contains the ansible playbooks to setup a Red Hat Enterprise Linux 9.2 image builder server.
The image created is an edge installer image that will install AWS Greengrass .jar files and run them as a daemon on first boot.

References:
- [Red Hat official documentation for RHEL OSTree](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html-single/composing_installing_and_managing_rhel_for_edge_images/index)
- [Red Hat Device Edge introduction](https://cloud.redhat.com/blog/introducing-the-new-red-hat-device-edge)
- [Image Builder quickstart bash scripts](https://github.com/luisarizmendi/rhel-edge-quickstart)
- [Ansible Collection for OSTree image management](https://github.com/redhat-cop/infra.osbuild)
- [Upgrade and Rollback edge demo](https://github.com/luisarizmendi/edge-demos/blob/main/demos/upgrade-and-rollback)


## Pre-requisites
Active Red Hat Linux subscription
Some VMs or baremetal servers

## Preparation
BEFORE running the playbooks, ensure the following is in place.

### Preparing the Image Builder

You need a subscribed [Red Hat Enterprise Linux 9](https://access.redhat.com/downloads/content/479/ver=/rhel---9/9.1/x86_64/product-software) system (minimal install is enough) with at least 2 vCPUs, 4 GB memory and 50 GB disk.

If you don't want to use `root`, be sure that the user has [passwordless sudo access](https://developers.redhat.com/blog/2018/08/15/how-to-enable-sudo-on-rhel).


### Preparing your laptop

Your will need to:

* Install Ansible

> laptop
```
dnf install -y ansible
```

* Download the `infra.osbuild` Ansible collection

> laptop
```
ansible-galaxy collection install -f git+https://github.com/redhat-cop/infra.osbuild
```

* Modify the Ansible `inventory` file with your values

* Copy your public SSH key into the Image Builder system, so you can open passwordless SSH sessions with the user that you configured in your Ansible inventory. (double check .ssh and authorized_keys permissions in case you are still asked for password after copying the key).

> laptop
```
ssh-copy-id <user>@<image builder IP>
```

* If you are using your laptop as hypervisor, be sure that you have at least 2 vCPU, 1.5GB memory and 20 GB disk free to create the Edge device VM (in addition the Image Builder VM that you should have already up and running).

## Configuring the image builder

The image builder will need several additional packages to be installed in order to successfully build the edge installer needed for greengrass.

Change into the AWSGG/playbooks directory and run the 00-prepare-systems.yml playbook
```
cd playbooks
ansible-playbook -vv 00-preparation-systems.yml
```
This will:
- install the infra.osbuild collections
- Install RPM packages required on the build server
- Enable cockpit and osbuild-composer services
- Create a custom repository with the PIP install files needed by greengrass
- 


## Create the image
The 01-build-ostree-aws.yml role will create the image needed to install on each device.
Run this role in the playbooks directory as follows:
```
ansible-playbook -vv 01-build-ostree-aws.yml
```

The last line printed will provide the link on where to download the iso from the build server. Something like http://.../demo_aws_greengrass/images/demo_aws_greengrass-custom-kernelarg.iso

The process executes the following tasks:
- Define names 
- Set SSH keys
- Configure the image for kickstart kickoff
- Create http server for images
- Update kickstarts in pre-built image and publish to http server
- 


## Modifications to final image
The kickstart_greengrass.j2 file sets up users and builds the shel script used to install greengrass.
Should additional items be needed in the script, it can be updated here and will automatically deploy upon the next installation.

Setting the root account's password can be done by replacing the value with the output from 
```
python3 -c 'import crypt,getpass;pw=getpass.getpass();print(crypt.crypt(pw) if (pw==getpass.getpass("Confirm: ")) else exit())'
```


## Manual steps to still apply after provisioning
Download AWS certificates and copy them to /var/greengrass/v2/certificates

```
cat > /var/usrlocal/bin/pre-install-aws-certs.sh << EOF
#!/bin/bash

mkdir -p ~/environment/certificates

\#Copy certificates here

cd ~/environment/certificates
mv AmazonRootCA1.pem rootCA.pem
mv *-certificate.pem.crt certificate.pem
mv *-private.pem.key privateKey.pem
mv *-public.pem.key publicKey.pem

\# Create the destination dir and populate it
sudo mkdir -p /var/greengrass/v2/certificates
sudo chmod 755 /var/greengrass/v2/certificates
sudo cp -avr ~/certificates /var/greengrass/v2


EOF
```
