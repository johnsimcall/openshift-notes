# Messing around with NVMe namespaces

## Acknowledgements

I got some good ideas from these authors:
https://nvmexpress.org/resource/nvme-namespaces/
https://www.drewthorst.com/posts/nvme/namespaces/readme/ (Drew, please renew your TLS certificate! :)
https://narasimhan-v.github.io/2020/06/12/Managing-NVMe-Namespaces.html
https://www.linkedin.com/pulse/linux-nvme-cli-cheat-sheet-frank-ober/
https://forum.level1techs.com/t/nvme-namespaces-little-known-cool-features-of-most-nvme-and-user-programmable-endurance/172660

## Script

This is the script I created to automatically query the available space / max space and create one giant namespace.
FYI - the "BEST" block size is 4096 as demonstrated above.

```bash
#!/bin/bash

DEVICE="/dev/nvme0"
BLOCK_SIZE="4096"

CONTROLLER_ID=$(nvme id-ctrl $DEVICE | awk -F: '/cntlid/ {print $2}')
MAX_CAPACITY=$(nvme id-ctrl $DEVICE | awk -F: '/tnvmcap/ {print $2}')
AVAILABLE_CAPACITY=$(nvme id-ctrl $DEVICE | awk -F: '/unvmcap/ {print $2}')
let "SIZE=$MAX_CAPACITY/$BLOCK_SIZE"

echo
echo "max is $MAX_CAPACITY bytes, unallocated is $AVAILABLE_CAPACITY bytes"
echo "block_size is $BLOCK_SIZE bytes"
echo "max / block_size is $SIZE blocks"
echo "making changes to $DEVICE with id $CONTROLLER_ID"
echo

# LET'S GO!!!!!
# nvme create-ns $DEVICE -s $SIZE -c $SIZE -b $BLOCK_SIZE
# nvme attach-ns $DEVICE -c $CONTROLLER_ID -n 1
```

## Odds and ends...

Some commands I used to gather additional info/details...

```bash
# FIGURE OUT THE CONTROLLER'S ID IN HEXADECIMAL
# nvme id-ctrl /dev/nvme0 | grep cntlid   
cntlid    : 0x21


# FIGURE OUT THE MAX AND AVAILABLE BYTES (-H for Human-Readable)
# nvme id-ctrl /dev/nvme0 -H
Total NVM Capacity (TNVMCAP)
tnvmcap   : 6,401,252,745,216
[127:0]   : 6,401,252,745,216

Unallocated NVM Capacity (UNVMCAP)
unvmcap   : 817,795,260,416
[127:0]   : 817,795,260,416


# CREATE A 1,000MB NAMESPACE AND QUERY THE BLOCK SIZES (Good, Better, Best & Degraded)
# PLEASE NOTE:
       "-b 512      "  is the same as  "-f 0     "  which is "BETTER"
       "-b 512  -d 1"  is the same as  "-f 1 -d 1"  which is "DEGRADED"
       "-b 4096     "  is the same as  "-f 2     "  which is "BEST"
       "-b 4096 -d 1"  is the same as  "-f 3 -d 1"  which is "GOOD"


# nvme create-ns /dev/nvme0 -s $((10**9)) -c $((10**9)) -b 512
# nvme attach-ns /dev/nvme0 -c 0x21 -n 1
# nvme id-ns /dev/nvme0 -n 1 -H | grep -e "^LBA Format"
LBA Format  0 : Metadata SIZE: 0   bytes - Data SIZE: 512 bytes - Relative Performance: 0x1 Better (in use) ######
LBA Format  1 : Metadata SIZE: 8   bytes - Data SIZE: 512 bytes - Relative Performance: 0x3 Degraded
LBA Format  2 : Metadata SIZE: 0   bytes - Data SIZE: 4096 bytes - Relative Performance: 0 Best
LBA Format  3 : Metadata SIZE: 8   bytes - Data SIZE: 4096 bytes - Relative Performance: 0x2 Good

# nvme attach-ns /dev/nvme0 -c 0x21 -n 2
# nvme create-ns /dev/nvme0 -s $((10**9)) -c $((10**9)) -b 4096
# nvme id-ns /dev/nvme0 -n 2 -H | grep -e "^LBA Format"
LBA Format  0 : Metadata SIZE: 0   bytes - Data SIZE: 512 bytes - Relative Performance: 0x1 Better
LBA Format  1 : Metadata SIZE: 8   bytes - Data SIZE: 512 bytes - Relative Performance: 0x3 Degraded
LBA Format  2 : Metadata SIZE: 0   bytes - Data SIZE: 4096 bytes - Relative Performance: 0 Best (in use)  ######
LBA Format  3 : Metadata SIZE: 8   bytes - Data SIZE: 4096 bytes - Relative Performance: 0x2 Good
```

I find it weird that these commands are the same

```bash
# nvme create-ns /dev/nvme1 -s 1562805846 -c 1562805846 -b 4096
create-ns: Success, created nsid:1


  # nvme delete-ns /dev/nvme1 -n 1
  delete-ns: Success, deleted nsid:1


# nvme create-ns /dev/nvme1 -s 1562805846 -c 1562805846 -f 2
create-ns: Success, created nsid:1
```