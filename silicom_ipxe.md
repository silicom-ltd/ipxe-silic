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

## Secure Boot

NetBoot.efi is in the UefiPayload firmware volume. The firmware trusts it because SBL verifies the payload. The firmware does not check NetBoot.efi against db, so NetBoot.efi needs no signature and no db entry. The boot manager starts NetBoot as a boot option, after EndOfDxe and SMM ReadyToLock.

### iPXE command line and menu limits

The iPXE shell, the embedded menu, and downloaded scripts all use the same commands. Under Secure Boot:

- iPXE has no command that writes UEFI variables, memory, I/O ports, MSRs, or PCI configuration space. It cannot change the Secure Boot keys or the Secure Boot state.
- iPXE loads every EFI image with the firmware `LoadImage()`. The firmware checks the image against db and dbx. An unsigned image does not run.
- On Intel NICs, the `set` command changes iPXE settings in RAM only. Silicom platforms use only Intel NICs at this time. The Realtek, Solarflare, Myricom, and ConnectX-3 drivers can store settings in the card EEPROM.

### Residual risks

1. **SBAT revocation with shim.** By default, the iPXE `shim` command blanks the SbatLevel data that shim reads. Shim then does not apply SBAT revocations. `shim --allow-sbat` keeps the SBAT checks, but only when shim loads GRUB, which has an `.sbat` section. A kernel that shim loads directly does not boot with `--allow-sbat`. See <https://ipxe.org/cmd/shim>.
1. **Kernel command line and initrd.** iPXE can boot a signed kernel with any kernel command line and an initrd that is not verified. A disk key sealed only to TPM PCR7 can then be unsealed. Seal to PCR4 as well as PCR7, or use a Unified Kernel Image (UKI) and seal to PCR11.
1. **DHCP autoboot.** A NetBoot.efi built without `EMBED=` (the vanilla build) offers the Ctrl-B command line and autoboots from DHCP. The LAN DHCP server then decides which script or image iPXE runs. To mitigate, build with `EMBED=` set to a script that loads its next stage over HTTPS from a fixed server.
1. **Unsigned scripts.** iPXE runs any script that it downloads. The firmware checks EFI images only, not iPXE scripts. To mitigate, run `imgtrust --permanent` in the embedded script, and verify each downloaded script and image with `imgverify`. The signing certificate must be a code-signing certificate that chains to a `TRUST=` root. `--permanent` stops a later `imgtrust --allow` from turning the check off. `imgtrust` checks only images that iPXE executes. It does not check an initrd, or a command line that `imgargs` changes. See <https://ipxe.org/cmd/imgtrust>.
1. **UEFI variable read.** A script can read any UEFI variable with `${efi/...}`. Scripts use `${efi/SecureBoot}` to detect Secure Boot. This includes the Setup admin password record (a salted PBKDF2 hash), and a script can send the value to a server. Any code that can read the SPI flash, such as the OS, can also read the record, so treat it as public and rely on the strong Setup passwords that the firmware enforces. Do not keep secrets in UEFI variables. To remove `${efi/...}`, add `#undef EFI_SETTINGS` to `src/config/local/settings.h`.

### HTTPS trust and db

- `ca.crt` is the iPXE root CA. It is a TLS certificate only. Do not add it, or any other TLS CA, to the Secure Boot db.
- `ca.crt` trusts all public CAs through cross-signed certificates that iPXE downloads from `http://ca.ipxe.org/auto`. For a private boot server, build with `TRUST=<customer CA>` so that iPXE trusts only that CA.
- `TRUST=` sets the trusted root certificates at build time. DHCP and the iPXE command line cannot change them.
- iPXE does not check whether a server certificate has been revoked. The Silicom build turns off `OCSP_CHECK` in `src/config/local/crypto.h`. To mitigate, remove that `#undef` if the boot network can reach the OCSP responders, or use a private CA with short-lived server certificates.

### netboot.xyz with Secure Boot

Only images signed by a db certificate, or loaded by a shim that the Microsoft UEFI CA signed, will boot. Most netboot.xyz menu entries fail. netboot.xyz reports that only Debian, Devuan, and Kali boot end to end. See <https://netboot.xyz/docs/kb/hardware/secureboot>. On a Secure Boot system, set `boot_domain` to a server that the customer controls.

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
