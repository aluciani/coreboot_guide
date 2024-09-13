# coreboot_guide Coreboot and me_cleaner: Free Your BIOS ( updated and translated https://connect.ed-diamond.com/GNU-Linux-Magazine/GLMF-220/Coreboot-et-me_cleaner-liberez-votre-BIOS)

Replacing Your BIOS with Coreboot and Neutralizing Intel ME

This article aims to release recent hardware at a low level. Currently, only Libreboot (a distribution of Coreboot) allows for completely removing Intel's "Management Engine (ME)" and other proprietary blobs. However, it is possible to neutralize the ME with me_cleaner and reduce it to its most basic functions.

The most advanced hardware compatible with Libreboot in Intel dates back to 2008 (Lenovo X200/T) and 2010 for AMD (Asus KPGE-D16), so rather old hardware that is still good for basic office or multimedia use. However, it is possible to neutralize the ME with me_cleaner and reduce it to its most basic functions. Here we will see how to use this software not on a standard BIOS (which is possible), but with Coreboot and SeaBIOS. The goal is to have a new free BIOS image with as few proprietary blobs as possible. Note that the manipulation was done with a Lenovo X230. I have no affiliation with them, but it's a brand whose laptops are particularly well supported by the Coreboot/Libreboot community, and for which we find many free drivers and good technical documentation. Everything presented in this article is therefore valid for most recent Thinkpads.

The Problem with Recent Hardware

The problem with newer hardware is that it is impossible to completely remove the ME; otherwise, the computer shuts down after 30 minutes due to a security measure imposed by Intel's proprietary software. However, there is a solution: me_cleaner and Coreboot. This small software allows neutralizing the ME and, when used with Coreboot, significantly reducing its size. We will see the details in a moment.

To recap, Coreboot (formerly LinuxBIOS) is a free boot software project created in 1999. It aims to replace proprietary BIOS found in most computers with a system whose sole function is to load a modern 32 or 64-bit operating system.

It is written in C and x86 assembly. me_cleaner is a Python script that can modify Intel's ME in a compiled BIOS image, ultimately reducing this firmware's capacity to interact with the system to its bare minimum. Below is a risk matrix for non-free parts contained in a BIOS:

| BIOS Region              | Risk Level                    |
|--------------------------|-------------------------------|
| Intel ME                 | High                          |
| VGA (Optional)           | Medium                        |
| CPU microcode            | Low                           |
| EC (Embedded Controller) | Low                           |
| GbE                      | Very Low (almost nonexistent) |

    Coreboot generally replaces the VGA and EC parts and does not touch the CPU microcode. By default, it integrates the original ME but can take a neutralized and reduced ME via me_cleaner, as we will see later.

# 1. Clarifications
# 1.1 What is the Management Engine?

The Intel Management Engine (ME) is an autonomous subsystem incorporated in almost all Intel processor chipsets since 2008. The subsystem mainly consists of proprietary firmware running on a separate microprocessor that performs tasks during boot, while the computer is running, and while it is in sleep mode. As long as the chipset or SoC is connected to power (via battery or power supply), it continues to operate even when the system is off. Intel claims that the ME is necessary to provide full performance. Its exact workings are undocumented, and its code is obscured using confidential Huffman tables stored directly in the hardware, so the firmware does not contain the information needed to decode its content. However, low-level reverse engineering has allowed us to understand a good portion of ME's mechanisms. Intel's main competitor, AMD, has incorporated the equivalent, AMD Secure Technology (formerly called Platform Security Processor), into almost all post-2013 processors.

The Management Engine is often confused with Intel AMT. AMT is based on the ME but is only available on processors with vPro technology. AMT allows owners to remotely manage their computer, such as turning it on or off and reinstalling the operating system.

However, the ME itself is integrated into all Intel chipsets since 2008, not just those with AMT. While AMT can be unprovisioned by the owner, there is no official and documented way to disable the ME.

The Electronic Frontier Foundation (EFF) and security expert Damien Zammit accuse the ME of being a backdoor and a privacy problem. Zammit states that the ME has full access to memory (without the parent CPU being aware), full access to the TCP/IP stack, and can send and receive network packets independently of the operating system, thus bypassing its firewall. Intel claims that it "does not dispute backdoors in its products" and that its products "do not allow Intel to control or access computer systems without the explicit permission of the end user."

Several vulnerabilities have been found in the ME. On May 1, 2017, Intel confirmed the existence of a Remote Elevation of Privilege bug (SA-00075) in its management technology. Every Intel platform with standard management technology, active management, or small Intel-provided technologies, from Nehalem in 2008 to Kaby Lake in 2017, has a remotely exploitable security flaw in the ME. Several ways to disable the ME without authorization that could allow ME functions to be sabotaged have been found. Additional significant security vulnerabilities in the ME affecting a large number of computers with ME firmware, Trusted Execution Engine (TXE), and Server Platform Services (SPS), from Skylake in 2015 to Coffee Lake in 2017, were confirmed by Intel on November 20, 2017 (SA-00086). Unlike SA-00075, this bug is present even if AMT is absent, unprovisioned, or if the ME has been "disabled" by one of the known unofficial methods.

# 1.2 Why is Intel's ME so harmful, and how is it made?

Intel's Management Engine (ME) is opaque proprietary software whose exact function is not well known. It has unlimited access to several branches of the computer (network, memory, hard drive...) and is completely transparent to the rest of the system (so what it does is undetectable from the operating system). Among other harmful things, the ME can remotely control almost the entire computer, which is a significant security flaw (see figure 1).


![Coreboot_and_me_cleaner_figure_01](https://github.com/123ahaha/coreboot_guide/assets/97607910/0dd5b804-706f-41fa-9b6e-bfe1a4067348)

Fig. 1: Diagram showing what the ME can control (full control) without any countermeasures by the end user (no interface).

The goal of me_cleaner is to disable the ME after the boot phase and limit the ME to the strict initialization of the hardware. Thus, no memory, disk, or network access can take place after the system boots, and no access to the end user's private data can occur. Additionally, me_cleaner has a function that allows reducing the space occupied by the ME, thereby increasing that of Coreboot (and SeaBIOS), allowing for additional features (we will see this in another part of the article).

Note: What is SeaBIOS?
SeaBIOS is an open-source implementation of a 16-bit x86 BIOS, serving as free firmware for x86 systems. Aiming for compatibility, it supports standard BIOS features and call interfaces implemented by a typical proprietary x86 BIOS. SeaBIOS can be used as a payload by Coreboot or directly in emulators such as QEMU and Bochs. Initially, SeaBIOS was based on the open-source BIOS implementation included with the Bochs emulator. The project was created to enable native use on x86 hardware and be based on an improved and more easily extensible internal source code implementation.
SeaBIOS is not the only payload that can be used with Coreboot. It is also possible to install a GRUB (1 or 2), a micro-Linux... Several alternatives can be found easily on the Web. However, it is clearly the one that installs and configures the most simply. It can also be modified from the operating system with the nvramtools software.

# 1.3 How is a BIOS made?
Since the test was done on a Lenovo X230, I will mainly explain how this BIOS works. It has the particularity of having two physical chips for one virtual one, but in principle, all x86 BIOSes work on the model of the virtual chip (see figure 2). In our case, we have two SPI flash chips hidden under the black plastic, labeled "SPI1" and "SPI2". Visually, the first one is 4 MB and contains the BIOS and the reset vector (which allows the "Reset" button to work). The bottom one is 8 MB, containing the Intel Management Engine (ME), the network bootloader (GbE), and the flash descriptor. The two chips are concatenated into a virtual 12 MB chip. We talk about BIOS regions.


![Coreboot_and_me_cleaner_figure_021](https://github.com/123ahaha/coreboot_guide/assets/97607910/7e3efd1c-5c42-4fb2-98c7-67168d38f884)

Fig. 2: Diagram of a BIOS with two physical chips read as one. When there is only one physical chip, the regions simply follow one another.

# 1.4 How do Coreboot and me_cleaner act on this structure?
If we take the structure again (without talking about occupied space for now), the Intel Flash Descriptor (IFD) is modified by me_cleaner to compensate for the loss of ME space; the BIOS, without VGA blobs, is replaced by Coreboot + SeaBIOS (we will see later what this is), and the GbE remains unchanged (see figure 3).


![Coreboot_and_me_cleaner_figure_03](https://github.com/123ahaha/coreboot_guide/assets/97607910/82082486-4dfc-426f-88ee-92fd05df2d8b)

Fig. 3: Diagram showing the regions affected (and how) by the use of Coreboot and me_cleaner. The network region remains unchanged.

# 2. Preparing for Compilation
# 2.1 Software and Hardware Prerequisites

On the software level, we need a host computer with a GNU/Linux system (I used debian 12 for this test, but the operation works on any distribution). Be as up-to-date as possible to avoid hardware compatibility, compilation, or other issues. To manipulate BIOS images, you will need flashrom installed in its latest version, git, and GCC up-to-date as well.

For the hardware, several solutions are possible, depending on the type of BIOS to flash. In the most common cases, the BIOS chips have 8 or 16 pins. I recommend a CH341A chip programmer (very affordable and easy to use). There are plenty of other programmers available on the market, but generally quite expensive and complex to use.

To connect the programmer to the BIOS, you will necessarily need an SOIC 8 or 16 clip, depending on the size of the chip, and its cable to connect it to the programmer. I recommend taking the shortest possible length to avoid problems during BIOS operations. You will, of course, need all the traditional tools to disassemble a desktop or laptop computer (screwdrivers, clips of all kinds, etc.).

# 2.2 Preparing the Linux Host

The first step is to install on the computer that will compile Coreboot all the necessary software and source codes. Proceed as follows:
```
$ sudo apt install flashrom bison build-essential curl flex git gnat libncurses5-dev libssl-dev m4 zlib1g-dev pkg-config wget
```
Then retrieve the Coreboot and me_cleaner sources and compile the various tools (place yourself at the home base to keep it simple). You also need to create 3 folders: mainboard (generic), a folder for the motherboard brand (for the test Lenovo), and the model (for the test X230). Examples are available in the official Coreboot documentation.
```
$ cd ~/
$ git clone https://review.coreboot.org/coreboot
$ cd coreboot
$ git submodule update --init --checkout
$ make crossgcc-i386 CPUS=$(nproc)
$ cd util/ifdtool
$ make CPUS=$(nproc) && sudo make install
$ cd ../cbfstool
$ make CPUS=$(nproc) && sudo make install
$ cd ~/coreboot
$ mkdir -p 3rdparty/blobs/mainboard/lenovo/x230/
```

# 2.3 Preparing the Target (the Computer to be Flashed)

The preparation has two main phases: making the BIOS accessible (disassembling the PC to access the BIOS on the motherboard see figure 4) and installing the clip (figure 5) on the BIOS (be careful with the orientation, place pin number 1 in the same spot on the programmer as on the BIOS). Then, connect the clip to the programmer and the programmer to the PC (see figure 6).


![Coreboot_and_me_cleaner_figure_04](https://github.com/123ahaha/coreboot_guide/assets/97607910/e92a1b02-b031-4e4e-873a-f366c9110ad0)

Fig. 4: Photo of a BIOS, in this case, a Lenovo X230 with two SOIC 8 type chips.


![Coreboot_and_me_cleaner_figure_05](https://github.com/123ahaha/coreboot_guide/assets/97607910/a3de1c1b-d287-4961-9d18-9d6ce02fb976)

Fig. 5: Photo of the type of clip needed for SOIC 8 or 16 chips.


![Coreboot_and_me_cleaner_figure_06](https://github.com/123ahaha/coreboot_guide/assets/97607910/6412d8e5-4681-4630-bd26-98e204118047)

Fig. 6: Photo of the wired and connected programmer. The number 1, which must be connected to the corresponding pin on the BIOS, is clearly visible.

Be sure to back up the original BIOS image multiple times to revert in case of error with flashrom and the following command (adapt according to the type of BIOS):
```
$ sudo flashrom -p ch341a_spi -r backup.rom
```
Note that if there are two physical chips for one virtual chip, back up both and concatenate them into one complete.rom using the cat command for the remaining operations.
```
$ sudo flashrom -p ch341a_spi -r 8MiB.rom
$ sudo flashrom -p ch341a_spi -r 4MiB.rom
$ cat 8MiB.rom 4MiB.rom > complete.rom
```

Go to the location of the complete.rom image. Now you need to extract all the regions of the original BIOS to recover blobs such as IME or GbE or IFD. Run the following command:

```
$ ifdtool -x complete.rom
```

You should find the following files (remember the first part about BIOS structure):

    flashregion_0_flashdescriptor.bin
    flashregion_1_BIOS.bin
    flashregion_2_intel_me.bin
    flashregion_3_gbe.bin

Copy flashregion_3_gbe.bin to the folder 3rdparty/blobs/mainboard/lenovo/x230 and rename it as follows:
```
$ cp flashregion_3_gbe.bin 3rdparty/blobs/mainboard/lenovo/x230/gbe.bin
```
Now neutralize and reduce the ME using me_cleaner. This process will reduce the size of the ME region by five times compared to the original. Run the following command:
```
$ python3 ~/coreboot/util/me_cleaner/me_cleaner.py -S -r -t -d -O out.bin -D ifd_shrinked.bin -M me_shrinked.bin ./complete.rom
```
We will pay more attention to this command in the next part to explain in detail the operation of me_cleaner and its effect on the final ROM. For now, note that me_shrinked.bin corresponds to the flashregion_2_intel_me.bin of the original ROM, but with a neutralized ME and reduced size (about five times smaller) and ifd_shrinked.bin corresponds to the new flash descriptor (region) indicating how the regions are placed relative to each other. In the end, we now have three new files:

    out.bin, which is useless
    ifd_shrinked.bin, the new descriptor
    me_shrinked.bin, the new neutralized and reduced ME

Copy the new ME and descriptor to the blobs folder:
```
$ cp ifd_shrinked.bin 3rdparty/blobs/mainboard/lenovo/x230/descriptor.bin
$ cp me_shrinked.bin 3rdparty/blobs/mainboard/lenovo/x230/me.bin
```
At this stage of the operations, we have all the elements necessary to build a new ROM based on Coreboot and SeaBIOS.

# 3. Compile a New Coreboot ROM and Flash the Thinkpad
# 3.1 Compiling the Coreboot ROM

Now, we will very simply compile a new Coreboot ROM. Start by going to the coreboot folder:
```
$ cd ~/coreboot
```
Since all the blobs have been copied, we can enter the menu to configure the future Coreboot ROM and create a .conf file that will serve to give compilation instructions. Since version 4.8 of Coreboot, SeaBIOS is downloaded, configured, and compiled into the final ROM automatically.
```
$ make nconfig
```
A window opens in the terminal; navigate it using the arrow keys and use the space bar to select a value.

In the General Setup menu: Set "Set CMOS for configuration values".
In the Payloads menu: Set "Add a payload" → SeaBIOS → git version.
In the Mainboard menu: Set "Mainboard Vendor" to “Lenovo” and Set "Mainboard model" to "X230".
In the Chipset menu (the complete paths are automatically added):

    Set "Add Intel descriptor.bin file",
    Set "Add Intel ME firmware" and
    Set "Add Gigabit Ethernet firmware".

In the Devices menu: Set "Use native graphics initialization".

The configuration is complete; now compile the ROM:
```
$ make CPUS=$(nproc)
```
The ROM is generated in ~/coreboot/builds/coreboot.rom.

# 3.2 Flashing the Laptop with the New ROM

Note: Flashing a BIOS

Be careful! The manipulation is risky and can render the computer inoperable. Building and replacing a BIOS is not a simple operation. It requires time, courage, and a lot of patience. Several verification steps are necessary, as well as suitable hardware. Avoid soldering or anything that can physically alter the hardware.

Before flashing hardware-wise, prepare the two ROMs that will go on the 4 MB and 8 MB chips. Recall (see part 2) that there are two chips in a specific order: 8 MB first and 4 MB second. Now, separate the recently created ROM. Run the following commands:
```
$ dd of=new_top.rom bs=1M if=~/coreboot/builds/coreboot.rom skip=8
$ dd of=new_bottom.rom bs=1M if=~/coreboot/builds/coreboot.rom count=8
```
The argument bs (block size) 1M means we take steps of 1 MB; the argument skip=8 means we skip the first 8 blocks of 1 MB, so we only take the final 4 MB part; the argument count is the inverse, we only take the first 8 blocks of 1 MB. In the end, we have two images, one of 4 MB and one of 8 MB. Since the two images are then assembled into a virtual 12 MB image, the allocation of regions within the two chips does not matter as long as the descriptor is in the 8 MB chip (at the beginning). We can now flash the BIOS with the same command for extraction or almost (with the same hardware constraint as in the previous part). You can then restart the PC.
```
$ sudo flashrom -p ch341a_spi -r new_bottom.rom -c 
$ sudo flashrom -p ch341a_spi -r new_top.rom
```

# 4. Operation Results and Explanations

Let's see what we have in the new ROM compared to the old one with a simple ls -al (I did an idftools -x on the original complete.rom and on coreboot.rom which is the new BIOS image (with the ME region reduced and neutralized by me_cleaner)):
```
ThinkPad-X230:~/coreboot/local_BIOS$ ll *.bin
-rw-r--r-- 1 user user 12472320 Jul  9 21:33 BIOS_local.bin
-rw-r--r-- 1 user user 7340032 Jul  9 21:39 BIOS_origine.bin
-rw-r--r-- 1 user user 98304 Jul  9 21:33 me_local.bin
-rw-r--r-- 1 user user 5230592 Jul  9 21:39 me_origine.bin
```

We immediately notice two things: the ME region and the BIOS region have changed in size. Indeed, me_cleaner has reduced the size of the ME to keep only the functions essential for component initialization. The size went from 5.230592 MB to 0.098304 MB, a reduction of over 5 times! The freed space is then recovered by the BIOS, hence the generation by me_cleaner of 2 binaries instead of one. The GbE and the descriptor remain unchanged in size.

As we saw at the beginning of this document, the initial BIOS structure corresponds to the figure 7.


![Coreboot_and_me_cleaner_figure_07](https://github.com/123ahaha/coreboot_guide/assets/97607910/e09925e5-eb95-4320-b1d8-d5d155d525b2)

Fig. 7: Image showing the standard structure of a BIOS before using me_cleaner.

Using the command to neutralize and reduce the ME affected the ME structure and therefore forced the descriptor to be modified to maintain ROM consistency and allow re-flashing the chip without impact (see figure 8).


![Coreboot_and_me_cleaner_figure_08](https://github.com/123ahaha/coreboot_guide/assets/97607910/7565041a-e806-4007-9a30-f29234167da6)

Fig. 8: Image showing the structure of a BIOS after ME reduction and neutralization.

# Conclusion

In conclusion, we have seen that it is possible and quite simple to replace your BIOS with a free alternative and limit security flaws on Intel platforms while waiting for better days. I invite you to look at the various links in the references and read the additional documents to better understand low-level security issues, how flaws are exploited, and how to go even further in securing your computer with Coreboot.
References

[1] Coreboot Official Documentation : https://coreboot.org/users.html (https://coreboot.org/users.html)

[2] me_cleaner Official Page : https://github.com/corna/me_cleaner

[3] Flashrom Supported Hardware Page : https://www.flashrom.org/Supported_hardware

To Learn More

    Intel ME Secrets by Igor Skochinsky at RECON in Montreal (Canada), 2014. 
        (https://recon.cx/2014/slides/Recon%202014 %20Skochinsky.pdf (https://recon.cx/2014/slides/Recon%202014 %20Skochinsky.pdf)
    
    Intel ME: The Way of the Static Analysis by the Positive Technologies research team, 2017, Heidelberg (Germany). 
        (https://www.troopers.de/downloads/troopers17/TR17_ME11_Static.pdf (https://www.troopers.de/downloads/troopers17/TR17_ME11_Static.pdf)
        
    The Intel Management Engine: An Attack on Computer Users' Freedom with contributions from Denis GNUtoo Carikli and Molly de Blanc, January 10, 2018. 
        (https://www.fsf.org/blogs/sysadmin/the-management-engine-an-attack-on-computer-users-freedom)
        
    Heads Official Site, a highly secure Coreboot distribution. 
        (http://osresearch.net/)
