# SIFT Workstation Notes

## Installation of Sift-Workstation
- Download the .ova file on the SANS website
   - This file is a VM file containing Ubuntu and SIFT in one package

## Install Sift-CLi
- After opening the .ova file and importing the VM, open a terminal and download these 3 sift-cli files from Github (https://github.com/teamdfir/sift-cli):
   - If Ubuntu asks you to upgrade, first view the Support Distros at https://github.com/teamdfir/sift to see if the new version is supported. I received an error using the sift-cli after upgrading Ubuntu. Must reinstall original version.
   - DO NOT RUN SUDO APT UPDATE OR UPGRADE: ERRORS OCCUR
   ```bash
   cd /usr/local
   sudo wget https://github.com/teamdfir/sift-cli/releases/download/v1.14.0-rc1/sift-cli-linux &&
   sudo wget https://github.com/teamdfir/sift-cli/releases/download/v1.14.0-rc1/sift-cli-linux.sig &&
   sudo wget https://github.com/teamdfir/sift-cli/releases/download/v1.14.0-rc1/sift-cli.pub
   ```
   - sift-cli allows you to update the sift workstation using:
   ```bash
     sudo sift install
     sudo sift update
     sudo sift upgrade
     ```

## Install cosign
- Cosign is used to sign and validate signatures for containers easily
1. Install from Github links latest version of file cosign-linux-*****:
   - `cd /usr/local`
   - To check your linux architecture: `uname -m`
     - This outputs X86_64 which AMD64 bit architecture
     - Find the associated cosign-linux-**** file in Github and copy link
   - `sudo wget https://github.com/sigstore/cosign/releases/download/v1.13.1/cosign-linux-amd64`
   - **attention to version number**
2. Next, move the Cosign binary to your bin folder.
   - `sudo mv cosign-linux-amd64 /usr/local/bin/cosign`
3. Finally, update permissions so that Cosign can execute within your filesystem.
   - `sudo chmod +x /usr/local/bin/cosign`

## Validate SIFT signature files
1. Validate the signature:
```bash
   cosign verify-blob \
   --key sift-cli.pub \
   --signature sift-cli-linux.sig \
   sift-cli-linux
```
2. Move the file to <br>
   `sudo mv sift-cli-linux /usr/local/bin/sift`
3. Run <br>
   `chmod 755 /usr/local/bin/sift`
4. Type `sift --help` to see its usage

## To Do:
1. Show example tools and commands to use for each artifact in Posters.
2. Update Creating Timelines, Memory Analysis, Registry Analysis sections

## Copying Evidence to SIFT
1. Use a USB
2. Copy and Paste files
3. Copy over the network:
    - Run `ifconfig` and grab the IP address
    - Go to the system with the evidence and open Windows Explorer
    - In address bar type `\\<SIFT_IP address>`
    - Copy files to `/cases/` directory on SIFT


## Mounting Evidence

1.	To mount an E01 file:
	- The /mnt directory is setup in SIFT with multiple folder names for different mounts
	- This is a compressed image file and should be in your `/cases/` directory
	- `ls -lh` and note the size of the file
	- Elevate privileges to root:  `sudo su -`
	- Expose the .E01 compressed image file as an uncompressed raw disk image
	    - Use the `ewfmount` command to create a virtual representation of the uncompressed image
	    - This will create a file named **ewf1** in the target directory
	    - Run this command then note the file size: <br>
    -   ```bash
        ewfmount <E01_file> /mnt/ewf_mount/
	    cd /mnt/ewf_mount
	    ls -lh
        ```

    - Now you have a representation of a raw disk image and can use mount to access the file system.
    - Use the alias `mountwin` since this has several preconfigured options
    - View alias definitions with:  `alias`
    - Commands to mount the ewf1 image:
    - ```bash
        mkdir /mnt/<folder_name>
        mountwin /mnt/ewf_mount/ewf1 /mnt/<folder_name>
        cd /mnt/<folder_name>
        ls
        ```
    - This should show the entire contents of the c-drive

2.	To unmount a drive use:
```bash
    umount /mnt/<folder_name>
    umount /mnt/ewf_mount/ewf1
```

3.	Step 1 above is a two-step process.  The SIFT station has a python program `imagemounter.py` that can combine the two steps above into one process.  This will also mount MS-DOS or GPT or other physical disk partitions it finds.
    - For a single partition to mount, use:
`imagemounter.py -s <file.E01> /mnt/windows_mount`
    - `-s` is for a single partition
    - This also has the option to mount a Bitlocker encrypted drive, but you need the recovery key.

4.	Another tool is `bdemount`, which is used to mount a Bitlocker Drive Encrypted (BDE) volume
    - If you don't have a recovery key you can use something like `passware` to try and brute force it

## Volume Shadow Mounting

- Past VSS snapshots are great for retrieving files that have been deleted or reviewing the registry

- To mount volume shadows, you need the raw image of a file.  
    - You can use `ewfmount` to make the ewf1 raw image

- To find out how many snapshot stores exist use:
    `vshadowinfo /mnt/ewf_mount/ewf1`

- Mount the raw image using `vshadowmount` to expose each of these stores as their own disk image
```bash
vshadowmount /mnt/ewf_mount/ewf1 /mnt/vss
cd /mnt/vss
ls -lh
```
- If there are multiple snapshots, they will be labeled vss1, vss2, etc.

- Mount the vss images in shadow_mount directory:
    - For a single image use:  
    `mountwin vss1 /mnt/shadow_mount/vss1`
    - For multiple images, use a `for` loop:
```bash
for i in vss*; do mountwin $i /mnt/shadow_mount/$i; done
```
- If you receive any errors or failure, its most likely because a vss is corrupted

## Creating Timelines

- `log2timeline.py` will create a supertimeline from a raw disk image like **ewf1** and from all snapshots as well.  It will also perform deduplication.
    - `log2timeline.py /cases/plaso.dump ewf1`
- Installing this on a desktop version instead of the virtual machine will increase the speed of creating the timeline as more processors would be available.

***Timeline section needs more info (in progress)***

## Memory Analysis

- Use Volatility & ReKall  
***This section in progress***

## Registry Analysis
1. RegRipper
    - Parses Windows Registry files using `rip.pl` (a Perl script)
- Change to users directory first:  
`cd /mnt/windows_mount/Users/nromanoff`
- `ls` should show the hive file NTUSER.DAT
- `rip.pl -r NTUSER.DAT -f ntuser | less`
    - `-f` specifies the plugin to use
    - Also run from: 
    - ```bash
        cd /Windows/System32/config
        rip.pl -r SAM -f sam
        ```
    - extracts files from Security Accounts Manager

2. ShimCacheParser
- Parses application shim cache data from the System hive file for executables that have run or potentially ran on the system.
- `ShimCacheParser.py -i SYSTEM --bom`
    - `--bom` used to import into csv files
    - the timestamp output is modification time (M) of the file, not execution time.

