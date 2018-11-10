To configure Supermicro X10SDV-8C-TLN4f+

Two BIOS Versions:

| ESXi Host | esxi-0.nono.io | esxi-1.nono.io |
|-----------|----------------|----------------|
| Version   | 1.2            |            1.3 |
| Date      |     04/21/2017 |     02/13/2018 |

No need to change the BIOS settings—the defaults are fine.

Follow [these
instructions](https://twitter.com/nono_io/status/1061300260205604864) to create
a bootable USB key.

Press F11 during boot to get to the boot menu. Make sure to boot from the [UEFI
version](https://twitter.com/nono_io/status/1061309103845175296) of the USB key.

On the console: press F2 to "customize system".
- Configure Management Network
  - IPv6 Configuration
    - Set static IPv6 address and network Configuration
      - Static address #1: `2601:646:100:69f0::41/64`
  - DNS Configuration
    - Use the following DNS server addresses and hostname
      - Hostname: esxi-1.nono.io
- Apply changes and restart management newtork? **Y**
- Test Managment Network
- Troubleshooting Options
  - Enable ESXi Shell
  - Enable SSH
- ESC
