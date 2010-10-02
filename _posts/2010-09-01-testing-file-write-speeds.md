---
layout: article
uuid: 3a3172d5-2abe-4d40-93cf-80f0c67e0d22
title: testing file write speeds
categories: uncategorized unfinished
updated_at: 2010-09-01
---
Goal
====

Get a good comparison between various SD Cards of various sizes, brands, and classes.

Determine if write bottlenecks are due to the Overo's data bus, or the microSDHC. 

Results
====

Write and Read speeds are with booting from NAND.

TODO try `jfs` and `ext4`

    Manufacturer      Size      Class     Write Speed     Read Speed    Filesystem
    ------------      ----      -----     -----------     ----------    ----------
    
    KingMax           8GB       10        4.7 MB/s        14.1 MB/s     ext3,defaults

    Kingston          16GB      10        4.9 MB/s        13.2 MB/s     ext3,defaults
    
    Kingston          16GB      10        2.8 MB/s        13.5 MB/s     **ext2**,defaults

    Kingston          16GB      4         4.9 MB/s        12.6 MB/s     ext3,defaults

    SanDisk           16GB      2         3.1 MB/s        13.1 MB/s     ext3,defaults
    
    ? (Gumstix)       2GB       ?         3.2 MB/s        9.7 MB/s      ext3,defaults

Comparison: regular desktop with Kingston Multi-reader USB 2.0

    Kingston          16GB      10        5.3 MB/s        137.0 MB/s    **ext4**,defaults

    Kingston          16GB      10        3.3 MB/s        36.0 MB/s     ext3,defaults

    Kingston          16GB      10        5.5 MB/s        91.2 MB/s     **ext2**,defaults

Procedure
====

I would like to try both configurations of booting from the NAND and booting from the microSDHC.

**NOTE**: The results above are only from booting the NAND.

Booting from the NAND
----

  0. Connect via Serial: 
    0. `ls /dev/tty*[uU][sS][bB]*` # Should list usb-serial devices equally well on Mac and Linux
    0. `sudo minicom -s` # Configure to use the port listed above
  0. Power on and enter u-boot by pressing the `any` key
  0. `run nandboot` 

If the card wasn't already formatted, it's formatted as ext3

    mount | grep mmcblk0 | grep -v ext3
    umount /media/mmcblk0p1/
    mkfs.ext3 /dev/mmcblk0p1
    mount /dev/mmcblk0p1 /media/mmcblk0p1
    
TODO: should have formatted all cards with filesystem before testing

Booting from the microSDHC
----

There are two other posts (possibly in the *related posts* section) about *partitioning* and *formatting* the microSDHC.
Once that's done, they'll boot.

Benchmark Procedure
----

What we need to know is if the card can stream the MB it boasts or not.

**NOTE**: `bitbake bonnie++` provides a good benchmarking tool for more realistic testing.

    # Get some random bytes from memory - say 40MB or so
    sync
    dd if=/dev/urandom of=/dev/shm/rand.dd bs=1M count=40

    # Emulate a continuous stream of data
    sync
    MMC=/media/mmcblk0p1
    #NOTE: This fills the cache buffers with data 
    cat /dev/shm/rand.dd /dev/shm/rand.dd | dd of=${MMC}/rand.dd

    # The overhead of the un`sync`d data above will make this more accurate
    cat /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd \
      /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd \
      /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd \
      /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd \
      /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd \
      /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd \
      /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd \
      /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd \
      /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd \
      /dev/shm/rand.dd /dev/shm/rand.dd /dev/shm/rand.dd | dd of=${MMC}/rand.dd

    # Test read speed just for fun
    sync
    dd if=${MMC}/rand.dd of=/dev/shm/rand.dd

**NOTE**: By default these devices are mounted with the `sync` option. You'll get a whopping 6.4KB/s unless you remount them.

**NOTE**: Due to caching, the speeds appear slightly faster than they really are.
If you just test with a small amount of data - 40MB, for example - you'll see that quite profoundly.

Appendix
====

Generating Random Bits

    sync
    dd if=/dev/urandom of=/dev/shm/rand.dd bs=1M count=40
    41943040 bytes (42 MB) copied, 42.311 s, 991 kB/s

Writing to the NAND (which only had 45 MB free)

    45474304 bytes (45 MB) copied, 414.42 s, 110 kB/s
