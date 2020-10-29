# Nerves System Jetson Nano

**TODO: Add CICD**
**TODO: Add Hexpm**

The base Nerves System configuration for the NVIDIA Jetson Nano developer kit, 4GB SD model (p3450-0000).

| Processing               |                               |
| ------------------------ | ----------------------------- |
| CPU                      | 64-bit Quad-core ARM A57 @ 1.43GHz |
| GPU                      | 128-core NVIDIA Maxwell @ 921MHz |
| Memory                   | 4GB 64-bit LPDDR4 @ 1600MHz \| 25.6 GB/s |
| Video Encoder\*          | 4Kp30 \| (4x) 1080p30 \| (2x) 1080p60 |
| Video Decoder\*          | 4Kp60 | (2x) 4Kp30 | (8x) 1080p30 | (4x) 1080p60 |
| ------------------------ | ----------------------------- |
| Interfaces               |                               |
| ------------------------ | ----------------------------- |
| USB                      | 4x USB 3.0 A (Host) \| USB 2.0 Micro B (Device) |
| Camera                   | MIPI CSI-2 x2 (15-position Flex Connector) |
| Display                  | HDMI \| DisplayPort           |
| Networking               | Gigabit Ethernet (RJ45)       |
| Wireless                 | M.2 Key-E with PCIe x1        |
| Storage                  | MicroSD Card                  |
| Other I/O                | (3x) I2C \| (2x) SPI \| UART \| I2S \| GPIOs |
| ------------------------ | ----------------------------- |
| Other                    |                               |
| ------------------------ | ----------------------------- |
| Linux Kernel             | 4.9                           |
| IEx terminal             | TODO                          |


\*Indicates maximum number of concurrent streams up to the aggregate throughput.
  Supported video codecs:  H.265, H.264, VP8, VP9 (VP9 decode only)

For more technical specifications, see the [Jetson Developer Blog](https://developer.nvidia.com/blog/jetson-nano-ai-computing/).

Please contact us about this if you're interested in it. This is currently experimental.

## Using

The most common way of using this Nerves System is create a project with `mix
nerves.new` and to export `MIX_TARGET=jetson_nano`. See the [Getting started
guide](https://hexdocs.pm/nerves/getting-started.html#creating-a-new-nerves-app)
for more information.

If you need custom modifications to this system for your device, clone this
repository and update as described in [Making custom
systems](https://hexdocs.pm/nerves/systems.html#customizing-your-own-nerves-system)

If you're new to Nerves, check out the
[nerves_init_gadget](https://github.com/nerves-project/nerves_init_gadget)
project for creating a starter project. It will get you started with the basics
like bringing up networking, initializing the writable application data
partition, and enabling ssh-based firmware updates.  It's easiest to begin by
using the wired Ethernet interface 'eth0' and DHCP.

## Root disk naming - TODO - translate for Nano

If you have multiple SSDs, or other devices connected, it's
possible that Linux will enumerate those devices in a nondeterministic order.
This can be mitigated by using `udev` to populate the `/dev/disks/by-*`
directories, but even this can be inconvenient when you just want to refer to
the drive that provides the root filesystem. To address this, `erlinit` creates
`/dev/rootdisk0`, `/dev/rootdisk0p1`, etc. and symlinks them to the expected
devices. For example, if your root file system is on `/dev/mmcblk0p1`, you'll
get a symlink from `/dev/rootdisk0p1` to `/dev/mmcblk0p1` and the whole disk
will be `/dev/rootdisk0`. Similarly, if the root filesystem is on `/dev/sdb1`,
you'd still get `/dev/rootdisk0p1` and `/dev/rootdisk0` and they'd by symlinked
to `/dev/sdb1` and `/dev/sdb` respectively.

## Provisioning devices - TODO - verify still works on Nano

This system supports storing provisioning information in a small key-value store
outside of any filesystem. Provisioning is an optional step and reasonable
defaults are provided if this is missing.

Provisioning information can be queried using the Nerves.Runtime KV store's
[`Nerves.Runtime.KV.get/1`](https://hexdocs.pm/nerves_runtime/Nerves.Runtime.KV.html#get/1)
function.

Keys used by this system are:

Key                    | Example Value     | Description
:--------------------- | :---------------- | :----------
`nerves_serial_number` | `"12345678"`      | By default, this string is used to create unique hostnames and Erlang node names. If unset, it defaults to part of the Ethernet adapter's MAC address.

The normal procedure would be to set these keys once in manufacturing or before
deployment and then leave them alone.

For example, to provision a serial number on a running device, run the following
and reboot:

```elixir
iex> cmd("fw_setenv nerves_serial_number 12345678")
```

This system supports setting the serial number offline. To do this, set the
`NERVES_SERIAL_NUMBER` environment variable when burning the firmware. If you're
programming MicroSD cards using `fwup`, the commandline is:

```sh
sudo NERVES_SERIAL_NUMBER=12345678 fwup path_to_firmware.fw
```

Serial numbers are stored on the MicroSD card so if the MicroSD card is
replaced, the serial number will need to be reprogrammed. The numbers are stored
in a U-boot environment block. This is a special region that is separate from
the application partition so reformatting the application partition will not
lose the serial number or any other data stored in this block.

Additional key value pairs can be provisioned by overriding the default provisioning.conf
file location by setting the environment variable
`NERVES_PROVISIONING=/path/to/provisioning.conf`. The default provisioning.conf
will set the `nerves_serial_number`, if you override the location to this file,
you will be responsible for setting this yourself.

## Troubleshooting - TODO - accurate for Nano?

### Device boots to GRUB CLI

If this happens, something could be off with the `grub.cfg`. Couple things you
should check here:

1. Make sure all your UUID's match what is in your `fwup.conf`
2. Your boot drive may be off (default uses `hd0`). Run `ls` in the grub CLI and
   you may see something like this:
   ```sh
   grub> ls

   (hd0) (hd1,gpt1) (hd1,gpt2) (hd1,gpt3) (hd1,gpt4) (hd2) (hd3) (hd4)
   ```
   If you do, you need to change which `hd` is looked at in `grub.cfg`, which
   means copying a few configs from here into your Nerves app:
   ```sh
   $ cp fwup.conf /path/to/my_app/config/
   $ cp grub.cfg /path/to/my_app/config/
   ```
   Change the line in your new `config/fwup.conf` to look at your custom `grub.cfg`
   ```sh
   file-resource grub.cfg {
       host-path = "${NERVES_APP}/config/grub.cfg"
   }
   ```
   Then change your new `config/grub.cfg` to match the `hd` needed.
   Lastly, specify to use the new `fwup.conf` in your Nerves app `config/config.exs`
   ```elixir
   config :nerves, :firmware, fwup_conf: "config/fwup.conf"
   ```
   Rebuild your firmware and try to boot again.

## Installation - TODO

If [available in Hex](https://hex.pm/docs/publish), the package can be installed
by adding `nerves_target_jetson_nano` to your list of dependencies in `mix.exs`:

```elixir
def deps do
  [
    {:nerves_target_jetson_nano, "~> 0.1.0"}
  ]
end
```

Documentation can be generated with [ExDoc](https://github.com/elixir-lang/ex_doc)
and published on [HexDocs](https://hexdocs.pm). Once published, the docs can
be found at [https://hexdocs.pm/nerves_target_jetson_nano](https://hexdocs.pm/nerves_target_jetson_nano).

