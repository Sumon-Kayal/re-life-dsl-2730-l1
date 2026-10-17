# Repurpose a D-Link DSL-2730U as a Wi-Fi Access Point

[![License: CC BY 4.0][cc-by-shield]][cc-by]

Give an old ADSL router a second life. This guide turns a **D-Link DSL-2730U** (hardware revision **I1**) into a Wi-Fi access point and Ethernet switch behind your main router, for example to bring Wi-Fi to another room over an Ethernet cable you already own. It uses the **stock firmware** only: you disable the D-Link’s DHCP server over Telnet and connect it LAN-to-LAN to your main router.

> [!IMPORTANT]
> **Firmware-specific.** The CLI commands in this guide were tested on a D-Link DSL-2730U, **hardware revision I1**, running firmware **`IN_R_01.00.01`**. Other hardware revisions or firmware builds may behave differently. This is **not** an OpenWrt installation guide, and it does not involve flashing firmware.

## At a glance

| Item | Details |
|---|---|
| Tested on | D-Link DSL-2730U, hardware revision I1, firmware `IN_R_01.00.01` |
| Method | Stock firmware · DHCP disabled with `dhcp mode none` · LAN-to-LAN cable |
| Main router | Your existing router, with its DHCP server left enabled |
| Result | A wired Wi-Fi access point and Ethernet switch |
| Limitations | Not a wireless repeater; the DSL port is not converted into an Ethernet WAN port |
| Cost | No extra cost if you already own the cable and both routers |

## Table of contents

- [At a glance](#at-a-glance)
- [Overview](#overview)
- [What you need](#what-you-need)
- [Before you start](#before-you-start)
- [Quick start](#quick-start)
- [Step 1: Install a Telnet client](#step-1-install-a-telnet-client)
  - [Android (Termux)](#android-termux)
  - [Windows](#windows)
  - [Linux](#linux)
- [Step 2: Open the router CLI](#step-2-open-the-router-cli)
- [Step 3: Disable DHCP](#step-3-disable-dhcp)
- [Step 4: Verify before saving](#step-4-verify-before-saving)
- [Step 5: Save the change](#step-5-save-the-change)
- [Step 6: Reconnect and test](#step-6-reconnect-and-test)
- [Recovering management access](#recovering-management-access)
- [Troubleshooting](#troubleshooting)
- [Safety notes](#safety-notes)
- [Tested device](#tested-device)
- [References](#references)
- [Contributing](#contributing)
- [Disclaimer](#disclaimer)
- [License](#license)

## Overview

In the tested setup, the D-Link was reused behind a main router as a wired Wi-Fi access point and Ethernet switch. The main router stays in charge of DHCP, so its DHCP server remains enabled. The D-Link has its DHCP mode set to `none` and saved.

```text
Main router · DHCP enabled · e.g. 192.168.0.1
        │
        │  Ethernet cable
        │  LAN port ──► LAN port
        ▼
D-Link DSL-2730U · access point · DHCP disabled and saved · 192.168.1.1
        │
        │  Wi-Fi (or Ethernet)
        ▼
Phones, computers, and other devices ──► internet
```

| Item | Value |
|---|---|
| Main router address | Your router’s own address (this guide uses `192.168.0.1` as an example) |
| D-Link management IP | `192.168.1.1` |
| Main router DHCP server | Remains **enabled** |
| D-Link DHCP mode | Set to `none` |
| Cable | LAN port on the main router → LAN port on the D-Link |
| D-Link DSL port | Unused |

**What you get**

- A wired Wi-Fi access point and Ethernet switch, running the stock firmware.
- Wi-Fi in another room, using an Ethernet cable you already own.

**What you do not get**

- **A wireless repeater.** The D-Link needs an Ethernet cable back to the main router.
- **An Ethernet WAN on the DSL port.** The DSL port stays unused; this method does not convert it into an Ethernet WAN port.
- **Custom firmware.** This guide is the verified stock-firmware method only. Any custom-firmware work is experimental and out of scope here.

> [!NOTE]
> **Example addresses and subnets.** This guide uses `192.168.1.1` for the D-Link and `192.168.0.x` as an example subnet for the main router; substitute your own values. It assumes the main router’s LAN uses a **different subnet** from the D-Link (`192.168.1.x`), as in the tested setup. A main router on `192.168.1.x` was not tested and is not covered here.

> [!NOTE]
> After DHCP is disabled, devices may receive addresses from the main router (`192.168.0.x` in the example) while the D-Link’s management IP stays `192.168.1.1`. Reaching the management page can then require temporarily setting a static IP on your phone. See [Recovering management access](#recovering-management-access).

## What you need

- A **D-Link DSL-2730U**, ideally hardware revision **I1** on firmware **`IN_R_01.00.01`**, the combination this guide was tested on.
- A **main router** with its DHCP server enabled.
- An **Ethernet cable** to run between a LAN port on each router. An existing cable works, which keeps the cost at zero.
- A phone or computer with a **Telnet client**: Termux on Android, Windows, or Linux (see [Step 1](#step-1-install-a-telnet-client)).

## Before you start

1. Know the D-Link’s Wi-Fi password, its login credentials, and its management IP (`192.168.1.1` in this example).
2. Keep the main router’s DHCP server **enabled**.
3. Check that the main router’s LAN subnet differs from the D-Link’s `192.168.1.x` (see the note in the [Overview](#overview)).
4. Keep the LAN-to-LAN cable disconnected while you configure and save the D-Link. Connect it only after DHCP is disabled and saved. Leave the DSL port unused.
5. Do **not** use `reset`, undocumented commands, or flash commands.
6. If the router is already working, record its current settings before changing anything.

## Quick start

1. Keep the LAN-to-LAN cable disconnected while configuring and saving the D-Link, and keep the main router’s DHCP server enabled ([Before you start](#before-you-start)).
2. Connect to the D-Link at `192.168.1.1` over Telnet ([Step 1](#step-1-install-a-telnet-client), [Step 2](#step-2-open-the-router-cli)).
3. Set DHCP mode to `none` ([Step 3](#step-3-disable-dhcp)).
4. Verify with `show dhcp` and `show status` ([Step 4](#step-4-verify-before-saving)).
5. Run `save` ([Step 5](#step-5-save-the-change)).
6. Connect the routers LAN-to-LAN after saving.
7. Reconnect your phone and test internet access ([Step 6](#step-6-reconnect-and-test)).

Already connected over Telnet and signed in? The router-side commands are:

```text
config
dhcp mode none
show dhcp
show status
```

Check that the output matches [Step 4](#step-4-verify-before-saving) (DHCP mode `None`, DHCP server `Disable`, LAN IP unchanged). **Only then** save:

```text
save
```

If anything differs, stop and investigate instead of saving.

## Step 1: Install a Telnet client

> [!WARNING]
> Telnet sends session data without encryption. Use it only on your trusted local network, and never expose Telnet management to the internet.

### Android (Termux)

Install Termux from a trusted source, then open it and run:

```sh
pkg update
pkg install inetutils
```

Check that the client exists:

```sh
telnet --help
```

Connect to the D-Link:

```sh
telnet 192.168.1.1
```

If it connects, sign in using the credentials configured on the router. Do not assume factory-default credentials if they have been changed.

#### If Termux says `Unable to connect ... Connection timed out`

A timeout means the TCP connection did not complete. It does **not** by itself prove that disabling DHCP also disabled Telnet. Check:

- Is the phone connected to the D-Link Wi-Fi?
- Is the phone on the same IPv4 subnet as `192.168.1.1`?
- Is Telnet enabled in the router’s access-control/service settings?
- Can the web interface open at `http://192.168.1.1`?

The D-Link can keep forwarding internet traffic even when its management address is not reachable from the client’s current subnet. See [Recovering management access](#recovering-management-access).

### Windows

**Windows 10/11:** enable the built-in Telnet Client. Open **Settings → System → Optional features → View features** (wording can vary), search for **Telnet Client**, and install it.

Alternatively, open **Command Prompt as Administrator** and run:

```bat
dism /online /Enable-Feature /FeatureName:TelnetClient
```

Then open Command Prompt or PowerShell and connect:

```bat
telnet 192.168.1.1
```

If Windows reports that `telnet` is not recognized, close and reopen the terminal after installing the feature.

### Linux

**Debian, Ubuntu, or Linux Mint:**

```sh
sudo apt update
sudo apt install telnet
```

On some Debian/Ubuntu-based distributions, the package `inetutils-telnet` may be available instead:

```sh
sudo apt install inetutils-telnet
```

Connect:

```sh
telnet 192.168.1.1
```

## Step 2: Open the router CLI

After connecting and signing in, the router may show a prompt similar to:

```text
$
```

Enter the configuration menu:

```text
config
```

On the tested firmware, the prompt changes to:

```text
[cfig]$
```

To list the available commands, enter:

```text
?
```

On the tested firmware, the menu included `dhcp`, `lan`, `show`, `save`, and `reset`.

> [!CAUTION]
> **Do not use `reset`.** It restores factory defaults and restarts the router.

## Step 3: Disable DHCP

At the `[cfig]$` prompt, inspect the DHCP submenu:

```text
dhcp
```

The tested firmware listed these subcommands:

```text
ipreserve
mode
relay
server
```

Set DHCP mode to `none`:

```text
dhcp mode none
```

This is the key command for the tested firmware. It disables the D-Link’s DHCP service mode; it does **not** change the D-Link’s LAN management IP.

## Step 4: Verify before saving

Run:

```text
show dhcp
```

Confirm the output contains:

```text
DHCP Mode: None
```

Then run:

```text
show status
```

Confirm the status reports:

```text
DHCP Server             : Disable
```

Also verify the LAN IP remains what you expect, for example:

```text
IP Address              : 192.168.1.1
Subnet Mask             : 255.255.255.0
```

> [!NOTE]
> `show dhcp` may still display the old pool range and interface names even when `DHCP Mode: None` is active. Use the reported mode and `show status` as the checks.

If the output is different, **do not save blindly.** Stop and investigate the prompt and output first.

## Step 5: Save the change

Once the checks confirm DHCP is disabled, run:

```text
save
```

If the router asks for confirmation or displays an unexpected prompt, read it carefully and do not guess. After saving, run `show status` again to confirm the DHCP server remains disabled.

## Step 6: Reconnect and test

1. Exit the CLI when finished (see [Troubleshooting](#troubleshooting) if you need help exiting Telnet).
2. On your phone or computer, disconnect from the D-Link Wi-Fi and reconnect. If necessary, use **Forget network** and join again.
3. Open a few websites.
4. Check the device’s network details.

In the example network, a client receiving settings from the main router should normally have:

- IPv4 address: in the main router’s subnet, for example `192.168.0.x`
- Gateway: the main router’s address, for example `192.168.0.1`

The exact client IP may differ. The important point is that the **main router** should provide the client’s gateway and DHCP lease. If websites do not load, see [Troubleshooting](#troubleshooting).

## Recovering management access

After DHCP is disabled, your phone may receive an address from the main router (for example `192.168.0.x`) while the D-Link’s management address remains `192.168.1.1`. These are different IPv4 subnets, so Telnet or the web page can time out. That does not necessarily mean the router is broken: Wi-Fi and internet forwarding can keep working while management access fails.

### Temporary recovery from Android Wi-Fi settings

While connected to the D-Link Wi-Fi, edit that Wi-Fi network and temporarily set **IP settings** to **Static**:

| Field | Example value |
|---|---|
| IP address | `192.168.1.2` |
| Gateway | `192.168.1.1` |
| Network prefix length | `24` |
| DNS 1 | `192.168.1.1` |

Only use `192.168.1.2` if it is not already in use. A static address in the D-Link’s subnet may allow the phone to open:

```text
http://192.168.1.1
```

If the address conflicts or the page still does not open, stop and reassess. **Do not factory-reset the router.** Restore the phone’s IP settings to DHCP when finished.

### Optional: easier management later

For easier management from the main router’s network, you can consider assigning the D-Link an unused management address in the main router’s subnet (for example, `192.168.0.2`) and making sure it does not conflict with the main router’s DHCP pool or another device. Do this only after checking your main router’s actual LAN configuration and learning the correct LAN-IP command or menu for your firmware. This is optional; do not change it just to make the access point work.

## Troubleshooting

- **Websites fail after connecting:** check the device’s IPv4 address and gateway, and make sure it got a fresh lease from the main router.
- **Stale DHCP lease (device still shows gateway `192.168.1.1`):** disconnect and reconnect, or forget and rejoin the Wi-Fi, then inspect the network details again.
- **Wi-Fi is visible and internet works, but Telnet times out:** this is likely a management reachability or service issue, not proof that DHCP is enabled. Check the device’s subnet, try the web page, and verify whether Telnet is enabled. See [Recovering management access](#recovering-management-access).
- **No internet for any client:** confirm the LAN-to-LAN wiring (a LAN port on both routers, DSL port unused), that the main router’s DHCP server is enabled, and that the D-Link DHCP mode is `None`.
- **Using the DSL port as the uplink:** this method does not convert the DSL port into an Ethernet WAN port. Use a LAN port on both routers instead.
- **Cannot reach management after using a static IP:** restore the phone’s IP settings to DHCP after the test, and avoid reset or flash commands.
- **Need to exit Telnet:** type `exit` if the router CLI supports it, or use Telnet’s escape sequence (`Ctrl+]`) and then type `quit`.

## Safety notes

- Do **not** use `reset`, undocumented commands, or flash commands.
- Never flash firmware intended for a different hardware revision. The commands here were tested on hardware revision **I1**.
- Record the router’s current settings before changing anything.
- Do not save blindly: if the output differs from what this guide describes, stop and investigate first.
- Do not assume factory-default credentials if they have been changed.
- If you cannot reach the management page, do not factory-reset the router; use the static-IP recovery above.
- Telnet is unencrypted: use it only on a trusted local network, and never expose Telnet management to the internet.
- Treat any custom or third-party firmware work (including OpenWrt) as experimental, and keep it separate from this verified stock-firmware method.

## Tested device

| Property | Value |
|---|---|
| Model | D-Link DSL-2730U |
| Hardware revision | I1 (reported hardware version `I1-2/8`) |
| Firmware | `IN_R_01.00.01` |
| Reported CPU | 448.92 MHz |
| Reported flash | 2 MB |
| Reported RAM | 8 MB |
| DHCP disable command | `dhcp mode none` |
| Verification commands | `show dhcp`, `show status` |
| Save command | `save` |

## References

- [D-Link DSL-2730U user manual (ManualsLib)](https://www.manualslib.com/manual/1145448/D-Link-Dsl-2730u.html)
- [D-Link DSL-2730U manual copy (Manualzz)](https://manualzz.com/doc/7147524/dlink-dsl-2730u-user-manual)

The CLI syntax in this guide is based on the tested `IN_R_01.00.01` I1 device described above. Consult the manual and your own device’s output before applying these steps to another revision.

## Contributing

This is a community guide. Tried it on a different DSL-2730U hardware revision or firmware build? Corrections and notes are welcome. Please open an issue or pull request and include your hardware revision and firmware version.

## Disclaimer

This guide is provided as is, without warranty of any kind. Changing router settings can interrupt your network, so record your current settings first and follow the safety notes above. This project is not affiliated with or endorsed by D-Link or any other vendor or internet provider; all product names are trademarks of their respective owners.

## License

This work is licensed under a [Creative Commons Attribution 4.0 International License][cc-by].

[![CC BY 4.0][cc-by-image]][cc-by]

In short, you may share and adapt this guide for any purpose, including commercially, as long as you give appropriate credit, link to the license, and indicate if you made changes. You may not suggest that the author endorses you or your use. When crediting, include the title, the author, a link to this repository, and a link to the license. The full terms are in the [legal code](https://creativecommons.org/licenses/by/4.0/legalcode).

[cc-by]: https://creativecommons.org/licenses/by/4.0/
[cc-by-image]: https://licensebuttons.net/l/by/4.0/88x31.png
[cc-by-shield]: https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg
