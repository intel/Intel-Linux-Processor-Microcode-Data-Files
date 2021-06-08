# Intel Processor Microcode Package for Linux

## About

The Intel Processor Microcode Update (MCU) Package provides a mechanism to release updates for security advisories and functional issues, including errata. In addition, MCUs are responsible for starting the SGX enclave (on processors that support the SGX feature), implementing complex behaviors (such as assists), and more. The preferred method to apply MCUs is using the system BIOS. For a subset of Intel's processors, the MCU can also be updated at runtime using the operating system. The Intel Microcode Package shared here contains updates for those processors that support OS loading of MCUs.

## Why update the microcode?
Updating your microcode can help to mitigate certain potential security vulnerabilities in CPUs as well as address certain functional issues that could, for example, result in unpredictable system behavior such as hangs, crashes, unexpected reboots, data errors, etc. To learn more about applying MCUs to an Intel processor, see [Microcode Update Guidance](https://software.intel.com/security-software-guidance/insights/microcode-update-guidance).

## Loading microcode updates

This package is provided for Linux distributors for inclusion in their OS releases. Intel recommends obtaining the latest MCUs using the OS vendor update mechanism. A good starting point is [OS and Software Vendor](https://software.intel.com/security-software-guidance/insights/guidance-system-administrators-mitigate-transient-execution-side-channel-issues). Expert users can update their microcode directly outside the OS vendor mechanism. However, this method is complex and could result in errors if performed incorrectly. Such errors could include but are not limited to system freezes, inability to boot, performance impacts, logical processors loading different updates, and some updates not taking effect. As a result, this method should be attempted by expert users only.

MCUs are best loaded from the BIOS. Certain MCUs must only be applied from the BIOS. Such MCUs are never packaged in this package since they are not appropriate for OS distribution. An OEM may receive microcode update packages that are a superset of what is contained in this package for inclusion in a BIOS.

OS vendors may choose to provide an MCU that the kernel can consume for early loading. For example, Linux can apply an MCU very early in the kernel boot sequence. In situations where a BIOS update isn't available, early loading is the next best alternative to updating processor microcode. **Microcode states are reset on a power reset, hence its required that the MCU be loaded every time during boot process.**

## Recommendation

Using the initrd method to load an MCU is recommended as this method will load the MCU at the earliest time for the most coverage. Systems that cannot tolerate downtime may use the late-load method to update a running system without a reboot.

## About Processor Signature, Family, Model, Stepping and Platform ID

The Processor Signature is a number identifying the model and version of an Intel processor. It can be obtained using the *CPUID instruction*, via the command *lscpu*, or from the content of */proc/cpuinfo*. It's usually presented as 3 fields: Family, Model, and Stepping.

For example, if a processor returns a value of "0x000906eb" from the *CPUID instruction*:

| Reserved | Extended Family | Extended Model | Reserved | Processor Type | Family Code | Model Number | Stepping ID |
|:---------|:----------------|:---------------|:---------|:---------------|:------------|:-------------|:------------|
| 31:28    | 27:20           | 19:16          | 15:14    | 13:12          | 11:8        | 7:4          | 3:0         |
| xxxx     | 00000000b       | 1001b          | xx       | 00b            | 0110b       | 1110b        | 1011b       |


The corresponding Linux formatted file name will be "06-9e-0b", where:  
- Extended Family + Family  = 0x06  
- Extended Model + Model Number = 0x9e  
- Stepping ID  = 0xb

A processor may be implemented for multiple platform types. Intel processors have a 3bit Platform ID field in MSR(17H) that specifies the platform type for up to 8 types. An MCU file for a specified processor model may support multiple platforms. The Platform ID(s) supported by an MCU is an 8bit mask where each set bit indicates a platform type that the MCU supports. The Platform ID of a processor can be read in Linux using rdmsr from [msr-tools](https://github.com/intel/msr-tools).

## Microcode update instructions

The [intel-ucode](https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files/tree/main/intel-ucode) directory contains binary MCU files named in the `family-model-stepping` format. This file format is supported by most modern Linux distributions. It's generally located in the /lib/firmware directory and can be updated through the microcode reload interface following the late-load update instructions below.

### Early-load update
To update early loading initrd, consult your Linux distribution on how to package MCU files for early loading. Some distributions use `update-initramfs` or `dracut`. Use the OS vendors recommended method to help ensure that the MCU file is updated for early loading before attempting the late-load procedure below.

### Late-load update
To update the intel-ucode package to the system:
1. Ensure the existence of `/sys/devices/system/cpu/microcode/reload`
2. Download the latest microcode firmware</br> `$ git clone https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files.git` or</br> `$ wget https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files/archive/main.zip`
3. Copy `intel-ucode` directory to `/lib/firmware`, overwriting the files in /lib/firmware/intel-ucode/
4. Write the reload interface to 1 to reload the microcode files, e.g.</br>
  `$ echo 1 > /sys/devices/system/cpu/microcode/reload`</br>
  Microcode updates will be applied automatically without rebooting the system.
5. Update an existing initramfs so that next time it gets loaded via kernel:</br>
`$ sudo update-initramfs -u`</br>
`$ sudo reboot`
6. Verify that the microcode was updated on boot or reloaded by echo command:</br>
`$ dmesg | grep microcode` or</br>
`$ cat /proc/cpuinfo | grep microcode | sort | uniq`

If you are using the OS vendor method to apply an MCU, the above steps may have been done automatically during the update process.

The [intel-ucode-with-caveats](https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files/tree/main/intel-ucode-with-caveats) directory contains MCUs that need special handling. The BDX-ML MCU is provided in this directory because it requires special commits in the Linux kernel otherwise updating it might result in unexpected system behavior. OS vendors must ensure that the late loader patches (provided in [linux-kernel-patches](https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files/tree/main/linux-kernel-patches)) are included in the distribution before packaging the BDX-ML MCU for late-loading.

The [linux-kernel-patches](https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files/tree/main/linux-kernel-patches) directory consists of kernel patches that address various issues related to applying MCUs.

## Notes

* You can only update to a higher MCU version (downgrade is not possible with the provided instructions)
* To calculate Family-Model-Stepping, use Linux command:</br>
`$ printf "%x\n" <number_to_convert_to_hex>`
* There are multiple ways to check the MCU version number BEFORE update. After cloning this Intel Microcode update repo , run the following:
  - `$ iucode_tool -l intel-ucode | grep -wF sig` ([iucode_tool](https://gitlab.com/iucode-tool/iucode-tool/-/wikis/home) package is required)
  - `$ od -t x4 <Family-Model-Stepping>` will read the first 16 bytes of the microcode binary header specified in \<Family\-Model\-Stepping\>. The third block is the microcode version. For example:
`$ od -t x4 06-55-04`</br>
`0000000 00000001 *02000065* 09052019 00050654`

## License

See the [license](https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files/blob/main/license) file for details.

## Security Policy

See the [security.md](https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files/blob/main/security.md) file for details.

## Release Note

See the [releasenote.md](https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files/blob/main/releasenote.md) file for details.

## Disclaimers 

Intel technologies’ features and benefits depend on system configuration and may require enabled hardware, software, or service activation. Performance varies depending on system configuration. Check with your system manufacturer or retailer or learn more at [www.intel.com](https://www.intel.com).

No product or component can be absolutely secure.

All information provided here is subject to change without notice. Contact your Intel representative to obtain the latest Intel product specifications and roadmaps.

The products and services described may contain defects or errors known as errata which may cause deviations from published specifications. Current characterized errata are available on request.

Intel provides these materials as-is, with no express or implied warranties.

© Intel Corporation.  Intel, the Intel logo, and other Intel marks are trademarks of Intel Corporation or its subsidiaries.

*Other names and brands may be claimed as the property of others.