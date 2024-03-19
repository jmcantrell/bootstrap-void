# bootstrap-void

A mildly-opinionated Void Linux installer.

Aside from the opinions listed below, care is taken to ensure the resulting system closely matches what you would get from following the [official installation guide][install].

## Opinions

Boot loading is handled by GRUB with a GPT partition table using BIOS or UEFI mode, depending on the detected hardware capabilities.

Logical volume management is handled by LVM, including a volume for swap (allowing for hibernation).

If enabled, [full disk encryption][fde] is handled by LUKS, using the LVM on LUKS method.

The file system is formatted using btrfs with subvolumes (see `./config/subvolumes`).

Any wireless connections created in the installation environment will be persisted to the installed system.

The following services are installed and enabled:

- [fstrim][ssd] (if installation disk is a solid-state drive)
- [acpid][power]
- [dhcpcd]
- [wpa_supplicant][wireless]
- avahi-daemon
- [chronyd][ntp]
- sshd
- [cronie][cron]
- socklog-unix
- nanoklogd

## Usage

In general, the installation steps are as follows:

1. Boot into the [Void Linux ISO][iso]
1. Change the directory to this repository
1. Set required [environment](#environment) variables
1. Prepare the environment: `. ./scripts/prepare`
1. Run the installation script: `./scripts/install`

After installation, the system is left mounted for inspection or further configuration.

If all is well, `poweroff` and eject the installation media.

## Configuration

The desired system is described by a [configuration directory](#configuration-files).
The default configuration directory at `./config` is what I consider a reasonable starting point based on the opinions outlined earlier and should serve as a template for customization.
The details of that system are controlled entirely by [environment](#environment).
These can be set manually, added to `$INSTALL_CONFIG/env`, or sourced manually from another file before sourcing the prepare script.

Once the necessary variable overrides are set, source the preparation script to fill in the blanks.
If the script succeeds, a list of all the relevant environment variables and their values will be displayed as a sanity check (with sensitive information hidden).

To prepare the environment for the default configuration:

```sh
. ./scripts/prepare
```

To prepare the environment for a different configuration:

```sh
export INSTALL_CONFIG=/path/to/config/dir
. ./scripts/prepare
```

### Environment

The following variables can be defined anywhere, as long as they're exported in the environment used to perform the installation.

**NOTE**: Boolean values must be specified as `0` (false) or `1` (true).

#### Required

- `INSTALL_DEVICE`: The disk that will contain the new system (e.g. `/dev/sda`, **WARNING**: all existing data will be destroyed without confirmation)
- `INSTALL_MEMORY_SIZE`: The amount of physical memory (e.g. `4G`)

#### Installation

- `INSTALL_CONFIG`: The directory containing [configuration files](#configuration-files) (default: `./config`)
- `INSTALL_MOUNT`: The path where the new system will be mounted during installation (default: `/mnt/install`)

#### Host Details

- `INSTALL_HOSTNAME`: The system host name (default: `void`)
- `INSTALL_LANG`: The default language (default: `en_US.UTF-8`)
- `INSTALL_KEYMAP`: The default keyboard mapping (default: `us`)
- `INSTALL_FONT`: The default console font
- `INSTALL_TIMEZONE`: The system time zone (default: the timezone set in the live environment, i.e. from `/etc/localtime`, or "UTC" if it's not set)

#### Users

- `INSTALL_ROOT_PASSWORD`: The root account password (only used if not setting a privileged user, default: `hunter2`)
- `INSTALL_SUDOER_USERNAME`: The primary privileged user's name (if set, the root account will be disabled)
- `INSTALL_SUDOER_PASSWORD`: The primary privileged user's password (default: `hunter2`)
- `INSTALL_SUDOER_SHELL`: The primary privileged user's shell (default: same as the default for `useradd`)
- `INSTALL_SUDOER_GROUP`: The group name used to determine privileged user status (default: `wheel`)
- `INSTALL_SUDOER_GROUP_NOPASSWD`: A boolean indicating that users in the group `$INSTALL_SUDOER_GROUP` should be allowed to use `sudo` without authenticating

#### Hardware

- `INSTALL_BOOT_FIRMWARE`: The firmware used for booting (default: automatically determined based on the presence of `/sys/firmware/efi/efivars`, possible values: `bios` or `uefi`)
- `INSTALL_DEVICE_IS_SSD`: A boolean indicating whether or not the installation disk is a solid-state drive (default: automatically determined based on the value in `/sys/block/$(basename $INSTALL_DEVICE)/queue/rotational`, see `./bin/is-device-ssd`)
- `INSTALL_LOGLEVEL`: Kernel log level (default: `4`)
- `INSTALL_CONSOLEBLANK`: The number of seconds of inactivity to wait before putting the display to sleep (default: `0`, i.e., disabled)

#### Partition Table

**NOTE**: Values for partition start and size must be specified in a way that [sfdisk(8)][sfdisk] can understand

- `INSTALL_BOOT_PART_NAME`: The name of the boot partition (default: `boot`)
- `INSTALL_BOOT_PART_START`: The start of the boot partition
- `INSTALL_BOOT_PART_SIZE`: The end of the boot partition (default: `100M` for UEFI, `1M` for BIOS)
- `INSTALL_SYS_PART_NAME`: The name of the operating system partition (default: `sys`)
- `INSTALL_SYS_PART_START`: The start of the operating system partition
- `INSTALL_SYS_PART_SIZE`: The end of the operating system partition (default: `+`)
- `INSTALL_UEFI_MOUNT`: The path where the EFI partition will be mounted (if applicable, default: `/efi`)

#### Full Disk Encryption

- `INSTALL_DEVICE_IS_LUKS`: A boolean dictating whether or not to use full disk encryption
- `INSTALL_LUKS_PASSPHRASE`: The passphrase to use for full disk encryption (default: `hunter2`, occupies key slot 0)
- `INSTALL_LUKS_ROOT_KEYFILE`: The path of the keyfile used to allow the initrd to unlock the root file system without asking for the passphrase again (default: `/crypto_keyfile.bin`, which is the default value used by `mkinitcpio`, occupies key slot 1)
- `INSTALL_LUKS_MAPPER_NAME`: The mapper name used for the encrypted partition (default: `sys`)

#### Volume Management

**NOTE**: Values for volume size and extents must be specified in a way that [lvcreate(8)][lvcreate] can understand.

- `INSTALL_LVM_VG_NAME`: The volume group name (default: `sys`)
- `INSTALL_LVM_SWAP_LV_NAME`: The name for the swap logical volume (default: `swap`)
- `INSTALL_LVM_SWAP_LV_SIZE`: The size of the swap logical volume (default: `$INSTALL_MEMORY_SIZE`)
- `INSTALL_LVM_ROOT_LV_NAME`: The name for the root logical volume (default: `root`)
- `INSTALL_LVM_ROOT_LV_EXTENTS`: The extents of the root logical volume (default: `+100%FREE`)

#### File System

- `INSTALL_FS_SWAP_LABEL`: The label for the swap file system (default: `swap`)
- `INSTALL_FS_ROOT_LABEL`: The label for the root file system (default: `root`)
- `INSTALL_FS_MOUNT_OPTIONS`: The mount options used for file systems (default: `autodefrag,compress=zstd`)

### Configuration Files

Within a configuration directory, the following files are recognized:

#### `$INSTALL_CONFIG/env`

This file, if it exists, will be sourced at the beginning of the preparation script.
It's treated as a bash script, and any variables relevant to installation (see [environment](#environment)) should be exported.

#### `$INSTALL_CONFIG/subvolumes`

This file, if it exists, defines the extra btrfs subvolumes that will be created.
This should **not** include the root subvolume, as its presence and mount point is not optional.
It will always be created and mounted at `/`.

If it's executable, it should output one subvolume mapping per line to stdout.
If it's a regular file, it should contain one subvolume mapping per line with no blank lines or comments.

Every line must be of the form:

```
name /path/to/subvolume
```

See `./config/subvolumes` for the default list.

#### `$INSTALL_CONFIG/packages`

This file, if it exists, defines the extra packages that will be installed on the new system.

If it's executable, it should output one package per line to stdout.
If it's a regular file, it should contain one package per line with no blank lines or comments.

Aside from these extra packages, only the packages necessary for a functional system will be installed (see `./bin/list-packages`).

By default, `./config/packages` does not exist, i.e., no extra packages are installed.

#### `$INSTALL_CONFIG/install`

This script, if it exists, will be run in a chroot just after packages have been installed.

## Installation

After the preparation script is sourced, the only other necessary step is to run the installation script:

```sh
./scripts/install
```

This script is intentionally kept extremely simple and easy to read.
It serves as a good overview of the installation process.
As `./bin` is now in `PATH`, feel free to execute each step separately to verify they're working as intended.

The commands can also be useful outside of the context of installation.
For example, the following can be used to mount an existing system (provided the configuration directory and environment match):

```sh
luks-open
swap-open
fs-mount
```

[acpid]: https://docs.voidlinux.org/config/power-management.html
[cron]: https://docs.voidlinux.org/config/cron.html
[dhcpcd]: https://docs.voidlinux.org/config/network/index.html#dhcpcd
[fde]: https://docs.voidlinux.org/installation/guides/fde.html
[install]: https://docs.voidlinux.org/installation/guides/chroot.html
[iso]: https://voidlinux.org/download/
[lvcreate]: https://man.voidlinux.org/lvcreate.8
[microcode]: https://docs.voidlinux.org/config/firmware.html
[ntp]: https://docs.voidlinux.org/config/date-time.html
[power]: https://docs.voidlinux.org/config/power-management.html
[sfdisk]: https://man.voidlinux.org/sfdisk.8
[ssd]: https://docs.voidlinux.org/config/ssd.html
[wireless]: https://docs.voidlinux.org/config/network/wpa_supplicant.html
