# FlightGear Demo with RHEL Image Mode
RHEL Image Mode enables management of edge device systems in a simplified
way by taking advantage of container application infrastructure. By
packaging operating system updates as an OCI container, RHEL Image Mode
simplifies the distribution and deployment of operating systems and
their updates, easing the amount of resources necessary to maintain a
disparate fleet of edge devices.

This demo shows an edge device completely changing it's workload and
filesystem configuration both by switching to a different bootable
container image from the one that's currently running and by patching
a sound issue. This demo illustrates that in a very visual and audible
way where multiple bootable container images are built and then swapped
to the edge device.

To make this interesting, this demo runs the open source FlightGear flight
simulator on an edge device using RHEL Image Mode. The simulator will
run in kiosk mode with a session for an unprivileged user. A privileged
user will also be configured on the edge device to enable switching the
bootable container image.

## Demo Setup
Start with a minimal install of the latest RHEL 9 release either on
baremetal or on a guest VM. Use UEFI firmware, if able to, when installing
your system. Also make sure there's sufficient disk space on the RHEL
instance to support the demo. I typically configure a 128 GiB disk on
the guest VM. During RHEL installation, configure a regular user with
`sudo` privileges on the host.

These instructions assume that this repository is cloned or copied to your
user's home directory on the host (e.g. `~/flightgear-kiosk-demo`). The
instructions below follow that assumption.

The open source FlightGear flight simulator is not normally available
for RHEL 9 so I had to custom build eight RPMs in addition to using EPEL
for other dependencies. You can find instructions on how to do that
[here](https://github.com/dthurston/build-flightgear-rpms)
but it's far easier, after cloning this repo, to download the custom built
[FlightGear RPMs](https://drive.google.com/drive/folders/112i4mOfHXXEoZNdSln_xWgMdx3ssWHtz?usp=drive_link)
and copy the `fg-rpms.tgz` file to the local copy of this repository
(e.g. `~/flightgear-kiosk-demo`).

The scenery data for FlightGear is quite extensive and downloads from the
official site are throttled. To speed this up, please download the [scenery
cache](https://drive.google.com/drive/folders/112i4mOfHXXEoZNdSln_xWgMdx3ssWHtz?usp=drive_link)
and copy the `SceneryCache.tgz` file to the local copy of this repository
(e.g. `~/flightgear-kiosk-demo`).

Download the RHEL boot ISO that matches `rhel_version` in
`group_vars/all/main.yml`, e.g. [rhel-9.8-x86_64-boot.iso](https://access.redhat.com/downloads/content/rhel),
to the local copy of this repository.

### Install Ansible
The setup is automated with Ansible playbooks that run against the
local host. Ansible is in the RHEL AppStream repository, so register the
host first and then install it along with the collections the playbooks
use.

    sudo subscription-manager register
    sudo dnf -y install ansible-core
    cd ~/flightgear-kiosk-demo
    ansible-galaxy collection install -r requirements.yml

### Configure the demo
All settings are in `group_vars/all/main.yml`. The secrets are in
`group_vars/all/vault.yml`. Edit the vault file to set your Red Hat
Simple Content Access credentials and the password for the edge device
user, then encrypt it.

    cd ~/flightgear-kiosk-demo
    vi group_vars/all/vault.yml
    ansible-vault encrypt group_vars/all/vault.yml

Use `ansible-vault edit group_vars/all/vault.yml` to change it later.
The secrets are:

| Variable        | Description |
| --------------- | ----------- |
| vault_rhsm_user | Your username for Red Hat Simple Content Access |
| vault_rhsm_pass | Your password for Red Hat Simple Content Access |
| vault_edge_pass | The plaintext password for the user on the target edge device |

The settings in `group_vars/all/main.yml` are:

| Variable          | Description |
| ----------------- | ----------- |
| rhel_version      | RHEL release for the `rhel-bootc` base image and the boot ISO |
| bootc_base_image  | The RHEL bootable container base image |
| boot_iso          | Minimal boot ISO used to create a custom ISO with a custom kickstart file |
| epel_url          | The Extra Packages for Enterprise Linux URL |
| edge_user         | The name of a user on the target edge device |
| edge_hash         | A SHA-512 hash of the edge user's password, computed from the vault |
| demo_user         | Unprivileged user on the edge device in kiosk mode |
| ssh_key_file      | SSH private key for logging in to the edge device, created if missing |
| host_ip           | The IP address of the container registry, defaults to this host's address |
| registry_port     | The port for the container registry |
| registry_insecure | Boolean for whether the registry is used without TLS |
| local_registry    | Boolean for whether `site.yml` runs a local registry on this host |
| container_repo    | The fully qualified name for your bootable container repository |
| scenarios         | The FlightGear scenarios to cache and build images for |

### Run everything at once
The `site.yml` playbook runs every step below in order: it prepares the
host, sets up the local registry, refreshes the FlightGear caches,
builds and pushes all of the images, and generates the ISO. Run Ansible
as your regular user, not with `sudo`. The `-K` option asks for your
`sudo` password for the steps that need root, and `-J` asks for the
vault password.

    cd ~/flightgear-kiosk-demo
    ansible-playbook site.yml -K -J

If the first playbook says a reboot is needed, reboot and run
`site.yml` again. Each step can also be run on its own as described
below.

### Prepare the host
The `setup.yml` playbook registers with Red Hat, updates the system,
enables the CodeReady Builder and EPEL repositories, installs the
container and ISO image tools, configures podman for the insecure
registry, and creates the `~/.ssh/id_core` keypair you'll use later
to access the edge device.

    cd ~/flightgear-kiosk-demo
    ansible-playbook setup.yml -K -J

Reboot if the playbook says updates require it.

### Set up a local registry (optional)
You can use a publicly accessible registry like [Quay](https://quay.io)
but if you want to run this demo disconnected, you can also set up a
local insecure container registry on this host. The `registry.yml`
playbook opens the registry port and runs the registry as a Quadlet
service.

    cd ~/flightgear-kiosk-demo
    ansible-playbook registry.yml -K -J

If you use a different registry, set `host_ip`, `registry_port`,
`registry_insecure` and `container_repo` to match and set
`local_registry: false`. When the insecure registry runs on another RHEL
instance, `setup.yml` still configures this host to use it.

## Review the FlightGear scenario files
Each FlightGear scenario is defined in a parameter file following
the naming convention `fgdemo<n>.conf`, where n is simply an integer
(e.g. `fgdemo1.conf`, `fgdemo2.conf`, etc). These files define the area
of operation for scenery, starting location for the aircraft, and various
other parameters such as altitude, speed, flight dynamics model, etc.

Several example scenarios are included in this repo but it's easy to add
more. The included scenarios are:

* F-35B above Langley AFB in Virginia (`fgdemo1.conf`)
* F-22A above Edwards AFB in California (`fgdemo2.conf`)
* B-52F above Barksdale AFB in Louisiana (`fgdemo3.conf`)
* UH-60 on the tarmac at Reagan National Airport in DC (`fgdemo4.conf`)

There's an extensive set of FlightGear [aircraft models](https://mirrors.ibiblio.org/flightgear/ftp/Aircraft-2020)
that you can use.

Each scenario is listed in `scenarios` in `group_vars/all/main.yml`
with the image tag prefix and its parameter file. The playbooks read the
aircraft model and starting position from the parameter file. To add a
scenario, create a new `fgdemo<n>.conf` file and add it to the list.

    scenarios:
      - tag: f35
        conf: fgdemo1.conf
      ...

## Refresh the scenario content for FlightGear
The FlightGear scenery is quite detailed and bulk downloads are throttled
to avoid overloading the servers. Normally, the simulator pulls scenery
on-demand using a feature called `terrasync`, but for disconnected use
cases, the scenery snapshot (`SceneryCache.tgz`) that you downloaded
earlier seeds scenery for flight over defined geographic areas.

The `cache.yml` playbook extracts `SceneryCache.tgz` into `SceneryCache`
the first time it runs, then refreshes the data with any changes since
the scenery data was last synced. How long this takes depends on the
extent of changes that are necessary. It also downloads the aircraft
model for each scenario into `AircraftCache`.

    cd ~/flightgear-kiosk-demo
    ansible-playbook cache.yml -J

After this playbook runs, both the `AircraftCache` and the `SceneryCache`
directories should be up to date.

## Build the bootable container images
The `build.yml` playbook logs in to Red Hat's container registry with
your customer portal credentials, pulls the RHEL bootable container
base image, then builds and pushes these images in order:

* `base`, which contains the Firefox browser running in kiosk mode
  (`BaseContainerfile`)
* `fgfs`, an intermediate image that installs the FlightGear flight
  simulator and its extensive scenery files. This image will be quite
  large. (`FGBaseContainerfile`)
* `<tag>-fixed` for each scenario, a bootable container with working
  sound (`FGDemoContainerfile`)
* `<tag>-broken` for each scenario, built from the matching `fixed`
  image with the sound turned off. We'll use this to illustrate
  patching. (`FGNoSoundContainerfile`)

Run the playbook as your regular user so podman runs rootless.

    cd ~/flightgear-kiosk-demo
    ansible-playbook build.yml -J

To rebuild only part of the set, use the `base`, `fgfs`, `fixed` or
`broken` tags, e.g. `ansible-playbook build.yml -J --tags fixed,broken`.

## Deploy the image using an ISO file
The `iso.yml` playbook generates an installable ISO file for your
bootable container. It writes a kickstart file that pulls the bootable
container image from the registry and installs it to the filesystem on
the target system. This kickstart file is then injected into the
standard RHEL boot ISO you downloaded earlier. It's important to note
that the content for the target system is actually in the bootable
container image in the registry. This ISO merely contains enough to start
the system and then use the kickstart file to pull the operating system
content from the container registry.

    cd ~/flightgear-kiosk-demo
    ansible-playbook iso.yml -K -J

The generated file is named `bootc-flightgear.iso`. Use that file to boot
a physical edge device or virtual guest. Ensure that you use the UEFI
firmware option for a virtual guest or install to a physical edge device
that supports UEFI. Make sure this system is able to access your public
registry to pull down the bootable container image.

You may see a core dump with FlightGear when running in a guest VM if
memory is low. I've successfully tested running the bootable containers
on a laptop with 16GB of memory and a 512GB SDD.

Test the deployment by verifying that the kiosk user automatically logs
into a desktop where only the web browser is available with no other
desktop controls.

## Switching between operating system images
Three bootable container images have been built where all of them will
run in kiosk mode once installed.

* Firefox browsing to the RHEL Image Mode landing page.
* FlightGear with an F-35B flying over Langley AFB in VA.
* FlightGear with an F-22A flying over Edwards AFB in CA.
* FlightGear with a B-52F flying over Barksdale AFB in LA.
* FlightGear with a UH-60 on the tarmac at Reagan National Airport in DC.

Once the `base` image is installed, it's very simple to switch between
them. The target device will auto-login to the unprivileged user `kiosk`
at startup and render the browser in full screen and kiosk mode. No
other desktop controls are available to the `kiosk` user.

To switch the bootable container operating system, login as the
`edge_user` defined in `group_vars/all/main.yml` earlier using ssh and the
`id_core` private key created earlier.

    ssh -i ~/.ssh/id_core core@IP_ADDRESS

where `IP_ADDRESS` is the address of the edge device.

Then type the following commands to switch to the F-22 flight simulation.

    sudo bootc switch HOSTIP:REGISTRYPORT/bootc-flightgear:f35-broken
    sudo reboot

where `HOSTIP` and `REGISTRYPORT` match the `host_ip` and `registry_port`
values in `group_vars/all/main.yml`. The other possibilities are:

    HOSTIP:REGISTRYPORT/bootc-flightgear:base
    HOSTIP:REGISTRYPORT/bootc-flightgear:f35-fixed
    HOSTIP:REGISTRYPORT/bootc-flightgear:f22-broken
    HOSTIP:REGISTRYPORT/bootc-flightgear:f22-fixed
    HOSTIP:REGISTRYPORT/bootc-flightgear:b52-broken
    HOSTIP:REGISTRYPORT/bootc-flightgear:b52-fixed
    HOSTIP:REGISTRYPORT/bootc-flightgear:uh60-broken
    HOSTIP:REGISTRYPORT/bootc-flightgear:uh60-fixed

You can demonstrate patching by switching from a `broken` tagged image
to a `fixed` tagged image.
