# kindling-spark-os

*Kindling Spark OS is in beta. Visit the discord for support https://discord.gg/M7XTrRJW3*

For an **already installed NixOS** GB10 host, use the [NixOS module](#nixos-host-services)
instead of the Ubuntu image installer below.

A minimal, read-only OS image for NVIDIA GB10 boxes (DGX Spark, ASUS Ascent GX10), installed beside
the box's DGX OS and booted from GRUB.

`kindling-spark-os` gives models about 4 GB more memory than DGX OS: about 2 GiB from a kernel with
64 KiB pages, and the 2 GiB display carveout, which the GPU driver leaves unused (see
[dispram](#dispram)).

![The kindling spark-os status screen on tty1: image version, addresses, GPU, memory, containers and the mentat cluster](docs/status-screen.png)

The image is an erofs file on the DGX OS disk, and every boot starts from the same image. At boot:

- the initramfs mounts the image read-only under a RAM overlay
- the initramfs copies in the box's identity: hostname, machine-id, ssh host keys, network
  config and `/etc/spark`
- systemd binds `/home`, `/srv`, `/var/log`, `/var/lib/docker`, `/var/lib/containerd`,
  `/var/lib/spark-watchdog` and `/var/lib/nfs` from the DGX OS disk. `--mounts` adds more.
- `sparkos-users` copies in DGX OS's login users (uid 1000 and up) and `/etc/sudoers.d`

Users log in with their DGX OS passwords and keys, and keep their DGX OS group memberships. If a
user's login shell is missing from the image, that user gets `/bin/bash`. The image has a fallback
`admin` user at uid 1000, and `sparkos-users` removes it when DGX OS has a user with uid 1000.
`/etc/subuid` and `/etc/subgid` stay on DGX OS.

DGX OS stays installed as the rescue system and manages GRUB.

The image carries:

- the kernel and NVIDIA driver of one supported pair (`stacks.yaml`), signed by Canonical for
  Secure Boot
- Docker and the NVIDIA container toolkit, RDMA and ConnectX tools, NVIDIA's DGX tuning packages
- `mentatd`, the [mentat](https://github.com/mmastrac/mentat) node daemon
- the [spark agent](agent/): a status page and MCP tools, registered with mentat
- `dispramd`, which lends the GPU's 2 GiB display carveout to CUDA (see [dispram](#dispram))
- `spark-watchdog`, a host log snapshot for the agent, and a cluster status console on tty1

## NixOS host services

The fork's `nixosModules.default` ports the Kindling host services to a native NixOS
configuration. It runs `mentatd`, the spark agent, host diagnostic snapshots, and the
NetworkManager health check. ConnectX MTU management, the tty1 console, the hardware watchdog,
and dispram are optional. The module does **not** run `setup.sh`, install a kernel, edit DGX OS
GRUB, copy DGX OS users, or bind its disk into the root filesystem. NixOS manages its own
persistent state, boot entries, and rollback. This is **not** the Ubuntu read-only erofs image or
its ten-minute trial/revert mechanism; do not replace a remote host's boot entry without a
separate recovery plan.

First install a bootable NixOS system using the [DGX Spark NixOS hardware module](
https://github.com/graham33/nixos-dgx-spark). Its documentation requires a firmware update
before NixOS will boot and Secure Boot disabled for the USB installer. Keep your host-specific
`hardware-configuration.nix`, boot-loader settings, users, and existing `system.stateVersion`.
For an existing NixOS host flake, add this fork as an input alongside the hardware module:

```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    dgx-spark.url = "github:graham33/nixos-dgx-spark";
    kindling.url = "path:/absolute/path/to/kindling-spark-os"; # this checkout
    kindling.inputs.nixpkgs.follows = "nixpkgs";
  };

  outputs = { nixpkgs, dgx-spark, kindling, ... }: {
    nixosConfigurations.spark = nixpkgs.lib.nixosSystem {
      system = "aarch64-linux";
      modules = [
        dgx-spark.nixosModules.dgx-spark
        kindling.nixosModules.default
        ./configuration.nix # your existing host configuration and hardware-configuration.nix
        {
          hardware.dgx-spark.enable = true;
          services.kindling.enable = true;
          # services.kindling.connectxMtu = 4096; # only if your switches support MTU 4200
          # services.kindling.gpuMaxClockMHz = 1500; # optional GPU clock ceiling
          # services.kindling.console.enable = true; # takes over tty1
        }
      ];
    };
  };
}
```

[straylight-software/nixos-dgx-spark](https://github.com/straylight-software/nixos-dgx-spark)
works in place of that hardware module, with the same `nixosModules.dgx-spark`. Its `main` is
graham33's module as it was on 2025-12-14: NVIDIA's 6.17 kernel, the 580.95.05 driver, and no
64 KiB kernel option. It builds the kernel from `pkgs.linux_6_17`, which nixpkgs has since
removed, so the host needs a nixpkgs from before then, such as the revision in its lock:
`nixpkgs.url = "github:NixOS/nixpkgs/45d3c39fc013ef15db3cf308a21fbd4b2d69a73d";`.

Place the cluster key outside the Nix store, in `/etc/spark/mentat.key` (mode 0400), and set
`MENTAT_SECRET_FILE=/etc/spark/mentat.key` in `/etc/spark/node.env`. Use the same key on every
cluster node. Optional `/etc/spark/agent.env` accepts `MENTAT_ROUTER_URL` and
`ALLOWED_SOURCES`. The snapshot and the status screen list containers with the Docker CLI: on
Docker's socket, or on Podman's (`/run/podman/podman.sock`) when the host runs Podman and not
Docker. Run one of the two, not both.

mentatd finds its peers by broadcast, and it announces only on interfaces that have a
broadcast address. NixOS's scripted networking adds static addresses
(`networking.interfaces.<name>.ipv4.addresses`) without one, so a box set up that way
announces nothing and never meshes. systemd-networkd (`networking.useNetworkd = true`) adds
them with one, and `ip -4 addr` shows it as `brd`. Without one, list the other boxes in
`node.env`, as `MENTAT_PEERS=192.168.1.12:6379,192.168.1.13:6379`.

mentatd announces every IPv4 interface that is up, except `docker*`, `veth*`, `br-*` and
`virbr*`. An address that every box has, such as Podman's `podman0` (10.88.0.1 by default) or
a VM's QEMU user network (10.0.2.15), then looks to the router like a way to reach any box, and
can lead it back to itself: in the VM test, the router ran spark2's agent tools on spark1. Name
the cluster's interfaces in `node.env` instead, as the VM test does with
`MENTAT_ANNOUNCE_IFACES=eno1,enp1s0f0np0`.

`services.kindling.connectxMtu` and `ethernetMtu` give each `en*` port its MTU when the port
appears, at boot or later, as a ConnectX port does when its cable goes in. On NetworkManager
hosts, the dispatcher sets it again whenever a profile comes up, and also kicks mentatd's health
check after a network change. Without NetworkManager, mentatd's own restart backoff, 2 to 60
seconds, stands in for that check.

The Ubuntu image runs no firewall. NixOS's firewall is on, and closed to the cluster by
default. `services.kindling.openFirewall = true` opens what mentatd and the agent listen on,
TCP 6379, 6380 and 8090 and UDP 6382, on **all** interfaces. A cluster needs more than that:

- The router must reach each model's announced endpoints, `MENTAT_OPENAI_API` and
  `MENTAT_MCP_API` (8000 and 9000 in mentat's examples). The recipe picks those ports.
- A model group that spans boxes runs NCCL's bootstrap, torch.distributed's rendezvous and
  vLLM's MessageQueue between them, on TCP ports picked at start. They use the boxes' fabric
  addresses when mentat placed the group on a fabric, and their LAN addresses otherwise.

So trust the links that carry that traffic and reach nothing but the cluster, such as the
ConnectX ports, and open the endpoint ports. `ip -br link` lists the ports. One cabled QSFP
port shows as two interfaces, and NCCL uses both:

```nix
networking.firewall.trustedInterfaces = [ "enp1s0f0np0" "enP2p1s0f0np0" ];
networking.firewall.allowedTCPPorts = [ 8000 9000 ]; # your recipes' endpoint ports
services.kindling.openFirewall = true;
```

A trusted interface accepts every port, so never trust one that reaches past the cluster. A
group that spans boxes over the LAN needs the LAN trusted too, which is only safe on a LAN that
holds nothing but the cluster. The agent serves host information to allowed source subnets, so
do not expose :8090 to the public Internet.

Before a box, try the module in VMs. `nix build -L .#checks.x86_64-linux.vm` (or
`aarch64-linux` on an Arm host) boots two NixOS VMs joined by a LAN and a fabric, with the
firewall above. It checks that the daemons name themselves and mesh, that a router on one VM
runs the other's agent tools, that the fabric passes what the LAN blocks, and that the MTU
unit, the snapshot, the status screen, the dispram mounts under Podman and the watchdog work.
The VMs have no GPU, no GB10 kernel and no RDMA, so `nvidia-smi`, dispramd and the ConnectX
MTU path still need a box. For a host on the straylight-software module, add
`--override-input nixpkgs github:NixOS/nixpkgs/45d3c39fc013ef15db3cf308a21fbd4b2d69a73d` to run
the same test on its nixpkgs.
`nix run .#checks.x86_64-linux.vm.driverInteractive` starts the same VMs from a Python prompt:
`start_all()` boots them, and `spark1.shell_interact()` gives a root shell.

Check the generation before scheduling a reboot with
`sudo nixos-rebuild build --flake .#spark`. On a NixOS host, use
`sudo nixos-rebuild boot --flake .#spark` after review. NixOS rollback uses its normal boot
menu/previous generation, **not** `sparkos-promote` or `sparkos-rollback`. After boot, verify
`systemctl status mentatd spark-agent spark-dmesg-snapshot.timer`,
`curl http://127.0.0.1:6380/status`, `curl http://127.0.0.1:8090/`, `nvidia-smi`, and the
running model containers. A missing `/etc/spark/node.env` deliberately prevents `mentatd`
from starting. The hardware watchdog is off unless
`services.kindling.watchdog.enable = true`; it can reset a host if process creation fails.
Enabling it disables systemd's runtime watchdog so only one process owns `/dev/watchdog`.

`nixos-rebuild switch` leaves `mentatd` and the tty1 status screen running as they were.
Restarting `mentatd` on the box that heads a model group kills that group, and restarting the
status screen hangs up whoever logged in on tty1. A new `mentatd` therefore takes over at the
next boot, or at a `systemctl restart mentatd` you run when no model group would mind.

The 64 KiB kernel is opt-in via `hardware.dgx-spark.use64kKernel = true` in the hardware
module; this module then disables transparent huge pages, as the Ubuntu image does. Check
page-size compatibility of swap and CUDA workloads before using it. See the [upstream kernel
warning](https://github.com/NVIDIA/open-gpu-kernel-modules/issues/1269) for GPU faults with
some newer drivers and large allocations on 64 KiB pages.

Dispram is **disabled**: the NixOS hardware module may choose a different driver from the
Ubuntu stack validated in `stacks.yaml`. Only after validating the exact NixOS driver and
carveout on your box, pin the matching NVIDIA `open-gpu-kernel-modules` source tree in your
flake lock, then add a flake input and pass it to your configuration. For the
repository's validated Ubuntu driver **only**, this means adding
`ogkm = { url = "github:NVIDIA/open-gpu-kernel-modules/c8e699821c23e4335bf23330a54a23d15cfb95e9"; flake = false; };`
to `inputs`, adding `ogkm` to `outputs`' argument set, and setting:

```nix
services.kindling.dispram = {
  enable = true;
  validatedDriver = "580.178.04";
  ogkmVersion = "580.178.04";
  ogkmSource = ogkm.outPath;
};
```

Replace **both** declared versions and the pinned source when validating a different driver.
The build fails unless the source's `version.mk` names `ogkmVersion`, and the daemon checks
`/proc/driver/nvidia/version` at runtime and exits without lending on a mismatch. With
`services.kindling.enable`, the module links the client plugin at
`/opt/kindling/dispram/python` and creates `/run/dispram`, the paths recipes mount (see
[dispram/README.md](dispram/README.md#vllm)), so a recipe starts whether or not dispramd runs.
Without dispramd the plugin stands aside. Dispram has **not** been
validated with the NixOS driver or on a NixOS GB10 host. Do not enable it just to chase
the Ubuntu image's 2 GiB claim.

### Firmware updates on a NixOS host

NVIDIA publishes the Spark's firmware to [LVFS](https://fwupd.org/) as UEFI capsules: the SoC
firmware (UEFI and GPU initialisation), the embedded controller, the USB-C power delivery
controller and the TPM. fwupd stages each capsule in `EFI/UpdateCapsule` on the EFI system
partition and sets the capsule bit in the `OsIndications` EFI variable, and the firmware applies
it on the next boot. ConnectX-7 and NVMe firmware are not offered on LVFS.

A Spark on its factory firmware boots only DGX OS, so take the first update there
(`sudo fwupdmgr refresh && sudo fwupdmgr update`). After that, NixOS can update itself.
graham33's hardware module enables fwupd. With straylight-software's, add
`services.fwupd.enable = true;`.

Check three things first. The last two make an update fail without any warning:

- `df -h /boot` shows about 60 MB free. fwupd wants twice the capsule plus 1 MB, and the SoC
  capsule is 30 MB. systemd-boot keeps every generation's kernel on that partition. If it is
  short, lower `boot.loader.systemd-boot.configurationLimit` and collect garbage. Without the
  room, fwupd refuses before it writes anything.
- `findmnt -no OPTIONS /sys/firmware/efi/efivars` starts with `rw`, as it does on NixOS by
  default. With it read-only, fwupd stages the capsules, reports success, and leaves
  `OsIndications` unset, so the firmware never looks at them.
- NixOS's entry comes first in `BootOrder` (`efibootmgr` lists it), or DGX OS and NixOS share
  one EFI system partition. UEFI firmware built on EDK2 looks for capsules only on the partition
  of the first active boot entry (`BootNext`, then `BootOrder`). Whether the Spark's firmware
  does the same has not been checked.

Then:

    sudo fwupdmgr refresh --force
    fwupdmgr get-updates
    sudo fwupdmgr update --no-reboot-check
    ls /boot/EFI/UpdateCapsule
    echo $(( 0x$(od -An -tx8 -j4 /sys/firmware/efi/efivars/OsIndications-8be4df61-93ca-11d2-aa0d-00e098032b8c | tr -d ' ') & 4 ))
    sudo systemctl reboot

`ls` lists one `fwupd-<guid>.cap` per update, and the `echo` prints `4`. Reboot through the
firmware: `systemctl kexec` skips it. A box applying SoC and EC capsules stays dark for about
ten minutes, so do not cut its power. Afterwards, `fwupdmgr get-devices` shows the new versions.
An embedded controller that still shows its old version can need a power cycle: shut down,
unplug, wait, and reconnect. If the screen asks for a UEFI administrator password during the
capsule update, stop: NVIDIA says that password cannot be reset and handles it through support.

In a VM that presented a Spark's ESRT entries, SMBIOS strings and capsule-on-disk support,
NixOS's fwupd (2.1.8, and 2.0.16 on straylight-software's nixpkgs) accepted NVIDIA's current
LVFS cabinets, staged each capsule byte for byte and set `OsIndications`. No firmware has been
flashed from NixOS on a box.

## Quick start

On the box, from DGX OS:

    git clone --branch stable https://github.com/kindlingai/kindling-spark-os.git
    kindling-spark-os/setup.sh --secret CLUSTER_KEY

Then `sudo systemctl reboot`. You can run `kindling-spark-os/setup.sh --check` to do a pre-check
on your system without installing.

`CLUSTER_KEY` is the secret for your cluster. We recommend using `openssl rand -hex 32`, but you'll
need to pass the same key to the rest of the boxes.

`setup.sh` checks the box first: an arm64 GB10, UEFI boot, DGX OS (or a running spark-os), GRUB,
NVIDIA's apt keyrings, sudo, Docker, disk space, and the package and image hosts.

### Trial mode

The OS is installed in "trial" mode and it auto-reboots back to DGX OS in case of failure. You'll
need to connect via ssh and run `sparkos-promote` within 10 minutes to keep it installed. 

`sparkos-rollback` goes back at any time.

## Supported stacks

Canonical builds each signed NVIDIA module package for one kernel ABI against exactly one driver
version, so a stack is that pair. `stacks.yaml` lists the pairs that have passed a trial boot, with
every `.deb`'s URL and sha256, and names a default. `tools/pin-stack.sh ABI DRIVER` writes a new
entry.

Ubuntu does not snapshot its ports archive, so an old pair's `.debs` can disappear from the mirror.
The setup image for a pair keeps a copy of them.

## Build the setup image

On an arm64 host with Docker (a box will do):

    tools/build-image.sh            # the default stack, tagged :<stack> (+ as -) and :latest
    tools/build-image.sh 7.0.0-1019+580.178.04

## Install on a box

From DGX OS:

    docker run --rm --privileged --network host -v /:/host kindling-spark-os install --trial

From a running spark-os, mount the DGX OS root disk instead:

    docker run --rm --privileged --network host -v /run/sparkos/host:/host kindling-spark-os install --trial

`install` builds the image (about 4 minutes), copies it to `/images/spark-os-VERSION.erofs` with its
kernel and initramfs in `/boot/sparkos/VERSION/`, and adds a GRUB entry. `--trial` makes that entry
the next boot, once. Reboot when ready.

Options:

- `--flavour nvidia-64k` (default) or `nvidia`. 64 KiB pages return about 2 GiB on a 128 GiB box.
  The 64k flavour boots with THP off, which keeps `vm.min_free_kbytes` near 45 MB. With THP on, it
  grows to 5% of RAM. Models on the 64k flavour must load their weights eagerly: see [vLLM](#vllm).
- `--version NAME`. The default is the release, kernel ABI and flavour, such as `0.9-1019-64k`, the
  same on every box, with `-2`, `-3`, ... when that name is already installed on the box.
- `--packages "P ..."`: extra packages, from Ubuntu, NVIDIA's repositories or a `--sources` file.
- `--sources FILE` (repeatable): an apt source on DGX OS, a `.list` or `.sources` file. The
  file's keyrings come from DGX OS too.
- `--mounts "DIR ..."`: more directories to bind from the DGX OS disk. Recipes that keep weights or
  compose directories in `/var/tmp` want `--mounts /var/tmp`. System directories are refused.
- `--hostname-kindling`: under spark-os, name the box `kindling-XXXX`, where XXXX is the last four
  hex digits of its LAN port's MAC (NVIDIA names boxes `gx10-XXXX` the same way). DGX OS keeps its
  own hostname. spark-os reads the name from `/etc/kindling-spark-os/hostname` on the DGX OS disk.
  Delete that file to go back.
- `--connectx-mtu N`: the RoCE MTU for the ConnectX ports, default 4096 (`0` turns it off). RoCE
  runs at the largest IB MTU that fits in the Ethernet MTU less 88 bytes, so a port at the usual
  1500 runs RoCE at 1024. At boot, and whenever a link comes up, a ConnectX port too small for N is
  raised to N + 104 (4200 for 4096). A larger MTU, such as 9000, stays. The switch or peer on that
  link must accept the larger frames. `setup.sh --check` lists the ports the image will raise.
- `--ethernet-mtu N`: an exact MTU for the other Ethernet ports, such as the RJ45 port. By default
  they keep DGX OS's setting.
- `--gpu-max-clock MHZ`: GPU 0 graphics/SM clock ceiling, applied at boot. Default: unchanged.
  `0` leaves clocks alone. CPU and memory clocks are not changed. See [GPU clock limits](#gpu-clock-limits).
- `--site DIR` (below).

`setup.sh` takes the same options. `list` shows the installed images and GRUB's state. `stack`
shows this setup image's kernel and driver pair.

### GPU clock limits

Clock limiting is opt-in. The Ubuntu image accepts `./setup.sh --gpu-max-clock 1500` (or the
same option on `kindling-setup build` and `install`). It writes `/etc/kindling/gpu-max-clock`
into the image, and `sparkos-gpu-clocks.service` applies it on each boot. This does not change
the currently running DGX OS. Omit the option, or use `0`, to build an image without a cap.

On NixOS, set `services.kindling.gpuMaxClockMHz = 1500;` in the host configuration. The
NVIDIA driver must be enabled. The default is `null`, which installs no clock-limit service.
Removing an active cap with `null` and switching stops the service and restores driver defaults.
This does not restart mentatd or the model containers.

Both use the shared `overlay/usr/local/sbin/sparkos-gpu-clocks` helper to run
`nvidia-smi --id=0 --lock-gpu-clocks=0,1500`. The zero minimum lets the GPU idle below the
ceiling, rather than pinning it at that speed. The driver's supported clock steps can make the
observed speed lower than requested. CPU and memory clocks are unchanged. The host administrator
owns the setting. The agent's MCP tools remain read-only.

For a temporary limit on a running Ubuntu image, without rebooting or restarting a model:

```sh
sudo sparkos-gpu-clocks --dry-run 1500
sudo sparkos-gpu-clocks 1500
nvidia-smi --query-gpu=clocks.sm,utilization.gpu,temperature.gpu,power.draw --format=csv
sudo sparkos-gpu-clocks reset
```

Stopping `sparkos-gpu-clocks.service` also restores the driver's clock defaults. This resets
clock settings only, not the GPU or its CUDA processes. It does not restore a lock set by another
tool, so do not run competing clock managers. A manual limit does not change the boot setting.
Clock changes require host root privileges (and `CAP_SYS_ADMIN` inside a GPU container).
Driver errors fail the service rather than silently claiming a cap was applied. Check
`systemctl status sparkos-gpu-clocks` and `journalctl -u sparkos-gpu-clocks` after applying a
persistent limit, then observe clocks under your own workload. A cap can change model latency.


### Swap

DGX OS swaps to a 16 GiB `/swap.img`. Some model loads peak above RAM and need it, and without swap
they are OOM-killed. A swap header records its page size, so a 64k image cannot use `/swap.img`.
For a 64k image, `install` makes a second file of the same size, `/swap-64k.img`, once. `install`
skips that file, with a warning, when less than 32 GiB would stay free (`MIN_FREE_GIB`). At boot,
`sparkos-swap` turns on the file that matches the running page size.

### Trial and promote

A new image boots in trial mode: `sparkos.trial panic=10` on the cmdline. A panic reboots. An
unconfirmed boot reboots 10 minutes after it starts. GRUB's one-shot entry is spent by then, so both
land back in DGX OS. To keep a trial running past 10 minutes without promoting it, run
`sudo touch /run/sparkos-confirmed`. That lasts for this boot only.

Once the image is good, run `sparkos-promote` on it. That confirms the boot, drops the trial flag,
and makes the image GRUB's default. `sparkos-promote` refuses if `update-grub` fails or the new entry
still carries the trial flag. DGX OS stays in the menu.

A promoted image still falls back by itself. Each spark-os entry calls GRUB's `recordfail` and
boots with `panic=30`, and `sparkos-boot-ok.service` clears the flag once the image reaches
multi-user. If a boot fails before then (a panic, a missing file, or a hang after spark-watchdog
starts), GRUB's next boot goes to DGX OS (entry 0) without anyone at the console. A hard hang
earlier in boot needs a power cycle, and the same fallback then applies.

### Rollback

`sparkos-rollback` makes DGX OS the default boot again and reboots. `sparkos-rollback VERSION` goes
back to an earlier installed image instead: as the default if that image was promoted, or as a
one-shot trial if it was not. `--no-reboot` sets GRUB without rebooting. Both forms cancel any
queued one-shot boot and clear GRUB's recordfail flag. `install` puts the same command in
the DGX OS root, so it works from either OS.

`enter-dgx-os [COMMAND]` runs a shell or command in the DGX OS install from spark-os, in a private
mount namespace. Use `enter-dgx-os` for tools the image does not carry: apt, update-grub, grub-reboot.

## Per-node settings

These files are in `/etc/spark` on each box's own disk. The initramfs copies them in at boot.

- `node.env` for `mentatd`: `MENTAT_SECRET_FILE`, and optionally `MENTAT_PEERS`,
  `MENTAT_ANNOUNCE_IFACES` and `MENTAT_NODE_IP`. Without it `mentatd` does not start.
  `setup.sh --secret` writes one with the key alone, which is enough: `mentatd` waits for the LAN,
  uses its address, and finds the other boxes by their broadcasts on the same network.
- `agent.env` for the spark agent, optional. The agent reads the cluster from the local `mentatd`
  and answers loopback and the box's own subnets. `MENTAT_ROUTER_URL` names the router: the page
  links to it, and the agent always lets it in. `ALLOWED_SOURCES` replaces the subnets with
  comma-separated CIDR blocks or address prefixes.

Boxes moving over from the container deployments should stop the `mentatd` and `spark-agent`
containers (`docker update --restart=no` and `docker stop`). Otherwise the containers take the same ports.

## Third-party software

### Tailscale

On a box that runs Tailscale under DGX OS, install it in the image from the same apt source, and
bind its state from the DGX OS disk:

    ./setup.sh --sources /etc/apt/sources.list.d/tailscale.list --packages tailscale \
               --mounts /var/lib/tailscale

With `/var/lib/tailscale` bound, the box keeps its tailnet identity and login. `setup.sh --check`
warns when DGX OS runs Tailscale and the options leave it out. A box reachable only over the tailnet
cannot be reached on a trial boot without Tailscale, and reverts after 10 minutes.

### vLLM

On the 64k kernel, CUDA hangs when it copies from a file-backed `mmap` to the GPU. vLLM's default
safetensors loader makes that copy for every tensor in the memory-mapped checkpoint. On a test box
(driver 580.178.04), stock vLLM sat at shard 0 for over 13 minutes, with libcuda spin-waiting and
the GPU idle. Reading the shards into ordinary memory first loaded the same model in 1.4 s. The 4k
kernel takes the same path without hanging, at its usual slower speed.

On the 64k flavour, start vLLM with `--safetensors-load-strategy eager`. Other loaders must copy
tensors into anonymous memory, for example with `tensor.clone()`, before moving them to the GPU.

## Site layer

Anything specific to one fleet goes in a site layer, so fleet addresses stay out of this repository.
`install --site DIR`, or `/etc/kindling-spark-os/site` on the DGX OS root when it exists:

- `DIR/root/` is copied over the image (units, scripts, config)
- `DIR/packages.txt` lists extra packages
- `DIR/customize.d/*.sh` run inside the image after the base customize step, in order. A site's
  scripts typically enable its units and add `/etc/fstab` lines, such as more binds from
  `/run/sparkos/host`.

## Services

| Unit | What |
|---|---|
| `mentatd` | mentat node daemon (needs `/etc/spark/node.env`) |
| `spark-agent` | status page and MCP tools on :8090, runs as `spark-agent` with `CAP_SYS_PTRACE` only (settings in `/etc/spark/agent.env`, optional) |
| `dispramd` | display carveout lender, on the validated driver only |
| `spark-watchdog` | pets the SBSA watchdog while the box can still fork |
| `spark-dmesg-snapshot.timer` | writes dmesg, docker and systemd state to `/var/log/spark` for the agent each minute |
| `spark-console` | cluster status on tty1; Esc to log in |
| `sparkos-trial-revert.timer` | reverts an unconfirmed trial boot |

## dispram

`dispramd` lends the GPU's 2 GiB display carveout to CUDA processes as ordinary device memory. The
driver never uses that memory on GB10. A patch for vLLM, and a plugin for stock images, put the tail of the KV
cache there. See [dispram/README.md](dispram/README.md). The server side is AGPL-3.0. The client,
vLLM plugin and patch are GPL-3.0 with a bundling exception, so they can ship inside vLLM or any image.

## Licensing

This repository is AGPL-3.0. The images it builds are collections of separately licensed packages,
and each keeps its own license:

- the Linux kernel: GPL-2.0. Source: Ubuntu's `linux-nvidia-7.0` and `linux-signed-nvidia-7.0`
  source packages for the stack's version.
- NVIDIA's open kernel modules: dual MIT/GPL-2.0. Source: `linux-restricted-modules-nvidia-7.0`,
  and [open-gpu-kernel-modules](https://github.com/NVIDIA/open-gpu-kernel-modules) at the driver's tag.
- NVIDIA's userspace driver and GSP firmware: NVIDIA's driver license. Section 1.1(d) of that
  license allows distributing them unmodified, with the license, for use with an open-source kernel.
  The license text is in each package's `copyright` file.
- NVIDIA's DGX tuning packages (`nvidia-spark-limits`, `nv-cpu-governor`, `nvidia-kernel-defaults`
  and others): proprietary, with no right to redistribute. The setup image does not carry these packages.
  `install` fetches them from NVIDIA's repository with the box's own apt keyring. A built image
  contains them, so do not publish one.
- `dispram/rmlist.c` compiles against NVIDIA's open-gpu-kernel-modules headers (MIT).
- Everything else comes from Ubuntu under its usual licenses.

## Credits

The members of [kindlingai](https://github.com/kindlingai) make kindling-spark-os:

- [@mmastrac](https://github.com/mmastrac) (Matt Mastracci)
- [@coffee-the-dev](https://github.com/coffee-the-dev) (Steve)
- [@adapt-ai-systems](https://github.com/adapt-ai-systems) (Chuck)

Thanks to [@joesinvestments](https://github.com/joesinvestments) (Joey) for testing and lots of early
feedback.

The idea behind [dispram](dispram/) was independently discovered a few weeks before us by emihuang on
the Nvidia forums.
