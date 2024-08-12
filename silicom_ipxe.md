# Silicom iPXE support

The default iPXE menu (script) included with Silicom iPXE is a slightly modified netboot.xyz menu. The netboot.xyz menu has several nice features to demonstrate iPXE booting. This menu can be customized for the behaviors we specifically want to highlight. This is what it currently does:

 1. Supports chain loading the iPXE scripts via TFTP, HTTP, HTTPS
 1. Supports typical PXE settings for the DHCP sever (next-server and proxydhcp/next-server) (See instructions from Ricky for setup and <https://ipxe.org/howto/dhcpd#ipxe-specific_options>)
 1. Supports iPXE command line and command line utilities <https://ipxe.org/cmd>

## Using netboot.xzy for network OS install

  Requirements: Internet connection

If the platform is connected to the internet, it will boot to the netboot.xzy menu. There are a few configuration steps you must take to load an OS install or load another utility.

1. Disable Signature Check. netboot.xzy uses HTTPS and self-signs it's menus and other files it serves. The Silicom iPXE doesn't contain the netboot.xyz certificate, so we have to disable the signature check. (See more about HTTPS in a later section)

   ```shell
   Signature Checks:
        netboot.xyz [ enabled: false ]
   ```

    - If you get this error, double check the Signature Checks is set to false.

   ```shell
   <http://boot.netboot.xyz/sigs/utils-efi.ipxe.sig>... ok
   Could not verify: Permission denied (<https://ipxe.org/0216eb8f>)
   Error occured, press any key to return to menu ...
   ```

1. Set the "console=" kernel command line parameter for the platform you are using.  Note, the console setting may vary depending on the OS.
   1. Valencia example:
      - Fedora: console=ttyS4,115200n8
      - Ubuntu: console=ttyS0,115200n8
      - If needed, earlycon=uart8250,mmio,0xfe03e000,115200n8

   1. Set the kernel command line param in Tools - Utilities

      ```shell
      Tools:
         Utilities (UEFI)
             netboot.xyz tools:
                Kernel cmdline params: [console=ttyS4,115200n8]
      ```

1. Install an OS
   - Example Install of Fedora with the console command line set to ttyS0 (see above)

   ```shell
   Distributions:
       Linux Network Installs (64-bit)
           Linux Distros:
              AlmaLinux
              ...
              Fedora
                  Latest Releases
                         Fedora 40
                               Fedora 40 Server
                                    Fedora 40 Server text install
   ```

## About HTTPS

iPXE HTTPS certificates work with a single ipxe.org/ca.crt certificate include in the binary that is used to validate all other certificates encountered. The device must be able to access ipxe.org to validate all HTTPS transactions.  In addition, it is possible to add custom certificates for self-signed sites and not have internet access. <https://ipxe.org/crypto>

Users can load the certs on the iPXE command line with the certstore command. This allows a "local" network copy of the certstore without rebuilding the binary.
<https://ipxe.org/cmd/certstore> A custom iPXE build can also hold the self-signed certs without a need for the external cert store.

## Self-hosted custom iPXE menu script without DHCP

For testing purposes, you may want to make a custom menu.ipxe without manipulating the DHCP server.

Requirements: HTTP server or TFTP server

1. Create a menu.ipxe and save it on your local web or tftp server.

   Here is a simple hello world that will print the SMBIOS serial number. See <https://ipxe.org/scripting> and <https://ipxe.org/cmd>

   ```script
   #!ipxe

   echo Hello World
   echo SMBIOS Serial: ${smbios/serial}
   ```

1. Use the iPXE shell to load your local menu.ipxe script (adjust the IP address as needed)

   ```shell
   iPXE> chain http://192.168.0.1/menu.ipxe
   ```

   - Note: HTTPS would require a self-signed certstore setup. See HTTPS section above.

## Silicom iPXE Build

Based on the SBL NetBoot instructions: <https://slimbootloader.github.io/how-tos/boot-netboot.html>

iPXE Build instructions: <https://ipxe.org/appnote/buildtargets>

1. Get Silicom iPXE

   ```bash
   git clone git@github.com:silicom-ltd/ipxe-silic.git
   cd ipxe-silic
   ```

1. Edit string settings in netboot.xyz.menu as needed:

   - set site_name {{ site_name }}
   - set boot_domain {{ boot_domain }}
   - set ipxe_version ${version}
   - set version {{ boot_version }}

1. Edit iPXE feature support in `src/config/local/general.h`. ex: HTTPS support

1. Get iPXE `ca.crt` cert file from ipxe.org. ***Required for HTTPS support.*** *See HTTPS section above for details.*

   ```bash
   wget https://ipxe.org/_media/certs/ca.crt
   ```

1. Build iPXE and copy it to the Silicom UefiPayload

   ```bash
   cd src
   make EMBED=../netboot.xyz.menu TRUST=../ca.crt  bin-x86_64-efi/ipxe.efi
   cp bin-x86_64-efi/ipxe.efi ${UEFIPAYLOAD}/PldPlatform/RplPlatformPkg/Binaries/UefiDriver/NetBoot/X64/NetBoot.efi
   ```

   - ex: Debug build

   ```bash
     make clean
     make EMBED=../netboot.xyz.menu TRUST=../ca.crt DEBUG=httpcore,tls bin-x86_64-efi/ipxe.efi
     cp bin-x86_64-efi/ipxe.efi ${UEFIPAYLOAD}
   ```

   - ex: Add custom certs. *See HTTPS section above for details.*

   ```bash
     make clean
     make EMBED=../netboot.xyz.menu TRUST=../ca.crt,../custom.crt CERT=../custom.crt bin-x86_64-efi/ipxe.efi
     cp bin-x86_64-efi/ipxe.efi ${UEFIPAYLOAD}
   ```
