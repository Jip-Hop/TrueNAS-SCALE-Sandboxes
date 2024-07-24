# TrueNAS SCALE Sandboxes

This repository provides research and a working POC on how to integrate [Sandboxes](https://www.truenas.com/docs/scale/scaletutorials/apps/sandboxes/) based on [`systemd-nspawn`](https://manpages.debian.org/bookworm/systemd-container/systemd-nspawn.1.en.html) into TrueNAS SCALE. Originally `systemd-nspawn` was included in SCALE as implicit dependency and no effort was put to accommodate it. Fortunately `systemd-nspawn` proved flexible enough and a usable solution was offered in the form of [`jailmaker`](https://github.com/Jip-Hop/jailmaker/), which follows a self-contained design without altering the root filesystem.

However, since iXsystems decided to build support for Sandboxes based on `systemd-nspawn` into SCALE there's no need to leave the root filesystem unaltered when developing proper integration. Hence it is now possible to create a solution where `systemd-nspawn` and `machinectl` (mostly) work like on any other system. **NOTE: this is a proposal**. It currently works on SCALE 24.04 but may stop working in future versions. The goal of this repo is to serve as a reference when implementing Sandboxes and iXsystems decides if this will make it into the [TrueNAS codebase](https://github.com/truenas/). The actual implementation is up to their developers.

**This repo exists for educational purposes. Try at your own risk!**

## Configuring a Sandbox

It's important to realize that configuring a Sandbox has to be done on two levels: 

1. The level of the `systemd` service manager
2. The level of `systemd-nspawn` itself

This can be done through CLI args (using `systemd-run` and `systemd-nspawn` directly, this is how `jailmaker` does it) or through `.service` and `.nspawn` config files.

From the [`systemd-nspawn` manual](https://manpages.debian.org/bookworm/systemd-container/systemd-nspawn.1.en.html)

> systemd-nspawn may be invoked directly from the interactive command line or run as system service in the background. In this mode each container instance runs as its own service instance; a default template unit file systemd-nspawn@.service is provided to make this easy, taking the container name as instance identifier. Note that different default options apply when systemd-nspawn is invoked by the template unit file than interactively on the command line. [...] Along with each container a settings file with the .nspawn suffix may exist, containing additional settings to apply when running the container. See systemd.nspawn(5) for details. **Settings files override the default options used by the systemd-nspawn@.service template unit file, making it usually unnecessary to alter this template file directly**.

### Level 1: `systemd`

However, in some cases it may be required to override the `/lib/systemd/system/systemd-nspawn@.service` file, which by default restricts access to certain devices (e.g. GPU or `/dev/fuse`). Or to disable seccomp. When using `systemd-run` this `.service` template file doesn't apply, so instead of selectively overriding you need to explicitly add all required options.

### Level 2: `systemd-nspawn`

Most config can be done using the `systemd-nspawn` CLI or `.nspawn` config files.

## Applying Patch

This code will patch the system to integrate Sandboxes. The patch is non-destructive and can easily be [undone](#undo-patch), but it temporarily alters the root filesystem through bind mounts. Run the code below. Replace the placeholder path for `sandboxes_dir`.

```sh
git clone https://github.com/Jip-Hop/TrueNAS-SCALE-Sandboxes
chmod -R 755 scripts
sudo chown -R 0:0 scripts
# Store the Sandboxes scripts in the /root home directory,
# it will be preserved on upgrades of SCALE
sudo cp -a scripts /root/usr-local-bin

sudo zfs set jip-hop:sandboxes_dir=/mnt/tank/path/to/desired/sandboxes/dir boot-pool
# TODO: is the boot-pool the best place to store this setting?
# Obviously when actually implementing Sandboxes properly this would go into the TrueNAS config DB
# sudo zfs get -H -o value jip-hop:sandboxes_dir boot-pool # Get the setting from the boot-pool
# sudo zfs inherit jip-hop:sandboxes_dir boot-pool # Clear the setting from the boot-pool

# Make the scripts available globally as if they were stored in /usr/local/bin
# And make the Sandboxes directories persistent
# This needs to be done again after every reboot
/root/usr-local-bin/sandboxes-patch
```

This process simulates configuring a Sandboxes dataset or directory by storing this user choice in the `sandboxes_dir` zfs property and makes the `sandboxes-create` command available regardless of the current working directory.

The `sandboxes-patch` script mounts the locations where `systemd` expects the `nspawn` related config files into the `sandboxes_dir` so that they will remain persistent (even after upgrading SCALE).

Then open the TrueNAS SCALE web interface and add `/root/usr-local-bin/sandboxes-patch` as Post Init Script with Type `Command`. Ensure `sandboxes_dir` actually exists and is stored on one of your data pools. Then reboot the NAS.

## Undo Patch

The easiest way to undo the patch it to disable the Post Init Script and reboot the system. Or you can follow the steps below to undo the patch without reboot.

```sh
machinectl list # shows running sandboxes
sudo machinectl stop # do this for all running sandboxes
sudo umount /usr/local/bin
sudo umount /run/systemd/system
sudo umount /var/lib/machines
sudo umount /etc/systemd/nspawn
sudo umount /etc/systemd/system/machines.target.wants
sudo rm /run/sandboxes_patch_applied
```

To completely remove all Sandboxes as well you may run the following.

```sh
sudo rm -r /mnt/tank/path/to/desired/sandboxes/dir # remove all sandboxes and config
```

## Manually Creating a Sandbox

After the reboot, login as root user and create your first Sandbox.

```sh
SANDBOX_NAME=debian
# Download some rootfs
machinectl pull-tar --verify=no https://github.com/debuerreotype/docker-debian-artifacts/raw/dist-amd64/bookworm/slim/rootfs.tar.xz "$SANDBOX_NAME"
# Show this image is now available to start as a Sandbox
machinectl list-images
# Install required packages using systemd-nspawn without booting the Sandbox
systemd-nspawn -M "$SANDBOX_NAME" apt-get update
systemd-nspawn -M "$SANDBOX_NAME" apt-get -y --no-install-recommends install dbus systemd systemd-sysv
# Optionally packages useful for debugging
# systemd-nspawn -M "$SANDBOX_NAME" apt-get -y --no-install-recommends install iproute2 inetutils-ping curl dnsutils net-tools nano
systemd-nspawn -M "$SANDBOX_NAME" systemctl enable systemd-networkd

# Allow login as root from console: https://github.com/systemd/systemd/issues/852
systemd-nspawn -M "$SANDBOX_NAME" rm -f /etc/securetty

# Setup a root password
systemd-nspawn -M "$SANDBOX_NAME" passwd
```

[NOTE on `systemd-resolved`](https://wiki.archlinux.org/title/systemd-nspawn#Domain_name_resolution):

> For the second case where systemd-resolved runs on the host, systemd-nspawn expects it to also run in the container, so that the container can use the stub symlink file /etc/resolv.conf from the host.

Since `systemd-resolved` is not running on the host we should not enable and install it in the Sandbox either.

Then copy the following text:

```ini
# This will make everything under /mnt (pools, datasets) accessible to the Sandbox
# By default usernamespacing is used so you need to set appropriate owner/group and file permissions
# so that processes in the Sandbox can actually read/write these files
[Files]
BindReadOnly=/mnt
```

And paste it in the editor opened by:

```sh
nano "/etc/systemd/nspawn/$SANDBOX_NAME.nspawn"
```

With this config file you can configure e.g. networking, bind mounts and disable usernamespacing. The example `.nspawn` config will make everything under `/mnt` accessible to the Sandbox.

TODO: show how to allow access to devices or disable seccomp with `systemctl edit --runtime systemd-nspawn@$SANDBOX_NAME.service` and point out this known issue regarding the `--runtime` flag.

```sh
# Enable automatically starting the Sandbox after a reboot
machinectl enable "$SANDBOX_NAME"
# Start the Sandbox now
machinectl start "$SANDBOX_NAME"
# Open a shell as the root user inside the Sandbox
machinectl shell "$SANDBOX_NAME"
```

You should now be able to execute commands in the Sandbox and it should be started automatically after the next reboot and will not be erased when you upgrade SCALE.

## Benefits

This approach benefits from the documentation and standard behavior of `systemd-nspawn`, because it's not reinventing the wheel. Only where Sandboxes deviates from how `systemd-nspawn` works by default needs to be documented. There's very little required, in terms of code, to integrate Sandboxes into TrueNAS SCALE this way. It does not require a custom config file format and code to parse and 'intercept' commands (e.g. to start a Sandbox) which are otherwise natively available. This means for example that `machinectl` can list images, `systemd-nspawn` can start an image (regardless of workdir) and you may temporarily override settings without editing config files for testing (then make permanent by editing config).

## Downsides

Handling special cases (GPU passthrough, setup of multiple bridge interfaces etc.) may be more challenging compared to a centralized approach like `jailmaker`. A centralized approach sits in between the config files and the actual starting of a Sandbox. This way all config related to a single Sandbox can be managed in a central place. Custom logic, which can't be achieved with CLI args of config options, can be implemented this way too (e.g. GPU passthrough with drivers).

## Missing Features v.s. Jailmaker

- Centrally managed config per Sandbox
- Convenient GPU passthrough (including nvidia libs)
- Automated rootfs download and setup
- Initial setup (customizing) the Sandbox
- Embedding hook scripts inside config files
- Config templates
- Creating a ZFS dataset for each Sandbox

## Known Issues

- Edits done with `systemctl edit systemd-nspawn@$SOME_SANDBOX.service` are lost after upgrading SCALE. A workaround is available by (ab)using `/run/systemd/system`. This is however not in line with the semantics of these dirs: `/run/systemd/system` should be lost each reboot but with this workaround is persistent and `/etc/systemd/system` should be persistent but edits to the `.service` files here are lost on upgrade. Fixing this properly requires integrating into the TrueNAS SCALE upgrade process. See comments inside `sandboxes-patch` for more info.
- The `-U` flag in the `/lib/systemd/system/systemd-nspawn@.service` file is causing issues. This sets `PrivateUsersOwnership` to `auto` and it will settle on `map` which causes the following errors: can't login with `machinectl login "$SANDBOX_NAME"` and running `systemd-run --machine "$SANDBOX_NAME" --quiet --pipe --wait --collect --service-type=exec ls /root` causes `ls: cannot open directory '/root': Permission denied`. As a workaround we explicitly set `--private-users=pick --private-users-ownership=chown` in the `override.conf` file for `systemd-nspawn@.service`. TODO: test again with latest update of SCALE if `map` works.

## Recommendation

If the SCALE web interface is going to support creating Sandboxes and editing their config, please make it so that changing a config option in through the GUI won't completely re-write the config file (and lose all changes not done through the GUI) as is the case with the current `libvirt` implementation (see [NAS-114640](https://ixsystems.atlassian.net/browse/NAS-114640) and [NAS-108713](https://ixsystems.atlassian.net/browse/NAS-108713)). This would mean not storing the config in the config DB, but directly on disk in the `sandboxes_dir` alongside the Sandboxes `rootfs` directories.

## References
- https://wiki.arcoslab.org/tutorials/tutorials/systemd-nspawn
- https://github.com/Jip-Hop/jailmaker