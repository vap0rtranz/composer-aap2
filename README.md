# AAP Zero Touch Lab
The goal of this runbook is to showcase the steps for building a Zero Touch lab environment that is entitled to the correct RH repos.  Or *near* zero :)  

Success criteria is to replace Packer/Terraform that has traditionally been used for lab buildouts, re-use the *same* process for Cloud & on-prem, and to assist in standing up AAP2 buildouts.

## Chicken or Egg Problem
We have a problem: chicken or egg?

You could use automation (aka. Ansible/Puppet/Chef) to build a lab but there's one problem:

`dnf install ansible`

or insert puppet | chef | perl (yes perl!)

i.e. How does someone automate the automation before there _is_ any automation available?!  Hmmm ...

In other words, we gotta bootstrap something somewhere 1st. IMHO, it's clearer to create my own custom image on a correclty entitled builder box and then bake my changes in, especially when dealing with the "Red Hat headache" of entitlements.  But to each their own.

## Builder Box

It takes a "builder box" to build custom OS images.  This builder OS is built by someone else (i.e. Red Hat) and then used here build custom OS images.

## Architecture of Lab
My lab consists of :

1. builder box
2. baremetal servers
3. Cloud servers

The builder box will be used to "bootstrap" the custom OS to target #2 & #3, the baremetal & Cloud boxes.  Specifically, the target tech is:

1. RHEL VM 
2. HPE DL380
3. Azure Cloud

## Packer ?

If you want to keep using Packer, skip the entire Composer sections.  

Goto: "Setup Centralized Automation"


# Build Custom OS Base
2-3 base OS images are needed to build this lab:

1. base OS for central automation = AAP2
2. base OS for baremetal hypervisor = RHEL8 + KVM/libvirt
3. base OS for Cloud compute = Azure Machine
4. base OS for VM guest = Windows or whatever

The order of these ^ is important.  

These base OS images should include only what is required for the above services.  

## Setup OS Builder Box

The below happens on RHEL8 VM.

### Get RHEL8.

I used Ricardo Gerardi's playbooks to streamline this (yes using ansible to install the builder box!  chicken or egg?): https://www.redhat.com/sysadmin/build-VM-fast-ansible

Or you can download the ready-made VM guest of RHEL 8.5 from Red Hat's website.  Look for "KVM Guest Image" or qcow2 format: https://access.redhat.com/downloads

(If using the ready-made VM, you can use virtsh tooling to reset the root password, inject your SSH keys, uninstall cloud-init, etc. Here's what I do:

`virt-customize -a rhel-8.5-update-2-x86_64-kvm.qcow2 --hostname builder-box --root-password password:${PASSWORD} --ssh-inject root:file:${SSH_PATH} --uninstall cloud-init --selinux-relabel`

Note: RHEL >v8.3 is required.

### Setup RHEL Composer

The builder box must have entitlements to RHEL subscriptions.  Specifically, the builder box pulls from Red Hat's repos via standard methods (dnf, and the dnf-makecache systemd service).

#### Enabling Red Hat Content Access

Composer defaults to only 2 RHEL repos: BaseOS, and AppStream.  Many packages are already in these default repos, but not all packages -- such as ansible-core.  See: /usr/share/osbuild-composer/repositories, and references to Red Hat's repos (download.devel.redhat.com/rhel-8/rel-eng/RHEL-8/latest-RHEL-8/compose)

The builder box (that runs Composer) controls access to content / packages via repo Sources.  You cannot Compose images with repo content directly, so entitling the builder box & enabling repos are the next steps.

Verify Builder Box Access

```bash
sudo subscription-manager status
+-------------------------------------------+
   System Status Details
+-------------------------------------------+
Overall Status: Disabled
Content Access Mode is set to Simple Content Access. This host has access to content, regardless of subscription status.
System Purpose Status: Disabled
```

That ^ is OK for Simple Content Access mode via Insights client.

_*Subscribtion & Entitlements*_

The builder box should be subscribed to the correct RH account repos (especially to gain access to paywalled repos).

Unsubscribe & Remove Previous RH Entitlements

```bash
subscription-manager unregister
subscription-manager remove --all
yum clean all
```

Re-subscribe to correct RH account

```bash
subscription-manager register --activation <ACTIVATION_ID> --org <ORG_ID>
```

That ^ ACTIVATION_ID gets created on RH's site: https://access.redhat.com/management/activation_keys

I also enable Insights by default now, but that isn't required for Composer.

```bash
yum install insights-client # <-- KVM images already have this
insights-client --register
```

#### Install Composer 

`yum install osbuild-composer composer-cli cockpit-composer bash-completion`

Enable CLI services

```bash
systemctl enable osbuild-composer.socket
systemctl enable osbuild-composer.service
systemctl start osbuild-composer.socket
systemctl start osbuild-composer.service
```

(Yes you could use --now but know that .socket & .service are dependencies so they should be started in order.)

Enable Composer for user so root/sudo isn't needed.

```bash
sudo usermod -aG weldr $USER
newgrp weldr
```

Enable Cockpit WebUI & CLI tab completion

`systemctl enable --now cockpit.socket`

`source /etc/bash_completion.d/composer-cli`

^ not required but can be handy for later when creating blueprints, searching packages, etc.

Verify base RHEL images & dependencies are available

```bash
$ composer-cli sources list
baseos
appstream
$ compose-cli compose types
ami
ec2
...
```

### Building An Image 

Update RHEL Builder Box to latest updates / patches

`yum update`

Create a Blueprint in TOML syntax.  Example is on weldr.io:

https://weldr.io/lorax/composer-cli.html#example-blueprint

Upload Blueprint to Composer

`composer-cli blueprints push example-custom-base.toml`

Verify Blueprint 

```bash
$ composer-cli blueprints list
...
$ composer-cli blueprints depsolve example-custom-base
```

The above ^ depsolve should list all packages that would be installed for this OS image.

Compose (aka. build) the OS image for KVM / qcow2

`composer-cli compose start example-custom-base qcow2`

Check status of Compose builds.  Note the UUID. UUID is needed later.

`composer-cli compose status`

Once built, export the image file to Build Box

`composer-cli compose image <UUID>`

We now have a custom built OS for KVM/QEMU.  These steps confirmed that Compose can build from the default repos.  

#### Re-building for Another Platform Provider

Easily re-build for another platform provider, aka Cloud, by simply re-running compose on the same blueprint.  No other changes needed.

Re-build blueprint to another provider platform (ex: AWS AMI instead of KVM/qcow2 from previous example)

`composer-cli compose start example-custom-base ami`

List target platform types

`composer-cli compose types`

#### Pushing Custom Image to Cloud & elsewhere

Custom images may be:

- automatically pushed to Cloud providers
- manually moved around as files

Manually move file format of custom build by exporting its image:

`composer-cli compose image <UUID>`

Automatically push image to Cloud provider

Now to add non-default repos ...

### Adding Additional / Custom Content

Additional (non-default) repos must be either: 

- available via public repo as a Compose Source, or 
- configured on builder box, i.e. paywalled via RH content access.  

The composed image "piggy-backs" off the builder box's authentication mechanisms for access to paywalled content / packages.

WARN: I would _not_ manipulate the builder box's DNF/Yum repos to gain access to repos for Composed images.  aka. /etc/yum.repos.d.  Use standard RHEL tools on the Builder Box to ensure certs & URLs & GPG key & etc. are accurate.  _<-- this is "the Red Hat headache" where Composer is more straightforward than other options._

Enable the builder / composer box's repos that you need for your custom OS image.  

Example: enable AAP2 paywalled repo on Builder Box

`yum config-manager --set-enable ansible-automation-platform-2.1-for-rhel-8-x86_64-rpms`

Verify the paywalled repo works on the Builder Box (<-- not required)

`sudo yum install ansible-core`

Enable overlayed repos for Composer.

```bash
mkdir -p /etc/osbuild-composer/repositories
cp /usr/share/osbuild-composer/repositories/rhel-85.json /etc/osbuild-composer/repositories/
cp -p /etc/osbuild-composer/repositories/rhel-85.json{,.bak}
```

Append a new repo below the BaseOS and AppStream repos in /etc/osbuild-composer/repositories/rhel-85.json.  

Compose may use RHSM authentication.  For these RH paywalled repos, copy the baseurl from the Builder Box's official RH paywalled repos in: /etc/yum.repos.d/redhat.repo.  

Enable this by configuring overlayed repo by reconfiguring the JSON in :

- set `baseurl` # <-- to URL in /etc/yum.repos.d/redhat.repo
- set `"rhsm": true`

Example: repo name "ansible-automation-platform-2.1-for-rhel-8-x86_64-rpms" = baseurl https://cdn.redhat.com/content/dist/layered/rhel8/x86_64/ansible-automation-platform/2.1/os

Insert the baseurl into /etc/osbuild-composer/repositories/rhel-85.json

Smarter folks may automate this with something like:

```bash
sed -i -e 's|cdn.redhat.com|<SERVER>/<REPO>/<ORG>/<ENV>/<CV>|' /etc/osbuild-composer/repositories/rhel-85.json
```

Following should _not_ be needed unless using Sat.  
Switch Composer to correct repo cert.

```bash
mv /etc/rhsm/ca/redhat-uep.pem{,.rpmsave}
ln -s /etc/rhsm/ca/katello-server-ca.pem /etc/rhsm/ca/redhat-uep.pem
```

Restart Builder to include new repos

`systemctl restart osbuild-worker@.service.d osbuild-worker@1.service osbuild-composer.service`

Verify that custom / paywalled repos are available

`composer-cli compose sources list`

Verify that package dependencies pull from additional repos via a Blueprint that has a package behind that repo:

`composer-cli blueprints depsolve $BLUEPRINT_NAME`

That ^ should pass.


# Setup Centralized Automation
Centralized automation will be done by the new Controller v4, aka. Ansible Automation Platform v2.

## Build RHEL Server Image Baked w/ AAP Setup

Pre-req: _Setup OS Builder Box_ from above

## Overlay the AAP2 Repo to Composer

```bash
mkdir -p /etc/osbuild-composer/repositories
git clone <REPO>
cp <REPO>/rhel-85.json /etc/osbuild-composer/repositories/
yum config-manager --set-enable ansible-automation-platform-2.1-for-rhel-8-x86_64-rpms
systemctl restart osbuild-worker@.service.d osbuild-worker@1.service osbuild-composer.service
```

... WIP ...

## Setup AAP2 Option A - Automated 

Basic process flow:

1. Start the custom built OS.
2. SSH into the custom built OS.
3. Run downloader playbook.
4. Run controller setup playbook.

### Pull the AAP2 Installer

The AAP2 Installer can be automatically downloaded.

I based this process on Lucas Benedito's role: https://www.redhat.com/en/blog/automating-installation-ansible-automation-platform-ansible-and-satellite
Process:

Setup the AAP Downloader Role:
`ansible-galaxy install lucas_benedito.installer_downloader`

Config the Downloader:

```bash
vim vars.yml
---
offline_token: <RH_API_TOKEN>
requires_satellite: False
```

```bash
vim aap2-download.yml
---
- name: Downloader for AAP installers
  hosts: localhost
  vars_files:
    - vars.yml
  roles:
    - { role: lucas_benedito.installer_downloader }
```

Pull the AAP Installer

`ansible-playbook aap2-download.yml`

That ^ downloads the installers to ./data

### Setup AAP2 Controller

WIP - the Controller setup can be automated.  See example:

https://gitlab.com/redhat-cop/ansible-ssa/role-controller-setup

or

https://github.com/redhat-cop/tower_utilities/

### Setup AAP2 Option B - Manual

With Ansible installed in this custom OS image, you can start the OS and run the AAP installer manually.

# Standup Hypervisors in Lab

# Extras

## Blueprint Format / TSOL

Blueprints are TSOL format.

A fuller, blueprint example is at Welder:
https://weldr.io/lorax/composer-cli.html

## Security

One big point of building custom OS images is they come pre-baked with all sorts of security nicities but these nicities merit attention.  For example, I would _not_ share my custom OS image bacause it includes:

- SSH keys
- certs 
- username passwords
- tokens

## Troubleshooting

If compose fails w/ DNF error, verify that the dnf-makecache service is running, and that a `dnf makecache` is able to work against the Red Hat composer repo (download.devel.redhat.com/rhel-8/rel-eng/RHEL-8/latest-RHEL-8/compose)

Removing Composer's Cache

```bash
rm -rf /var/cache/osbuild-composer/
rm -rf /var/cache/osbuild-worker/
```

Removing Repo Overlays

`rm -rf /etc/osbuild-composer/`
