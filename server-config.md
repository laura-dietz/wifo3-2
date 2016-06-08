wifo3-2 Config
==============

# Motherboard SATA

|      |      |     |      |   |      |      |      |      |
|------|------|-----|------|---|------|------|------|------|
|      |      |     |      |   |      |      |      |      |
|      |      |     |      |   |      |      |      |      |
| R1C2 | R1C3 |     |      |   |      |      |      |      |
| R1C0 | R1C1 |     |      |   |      |      |      |      |
|      |      |     |      |   |      |      |      |      |
|      |      | SSD |      |   | R0C3 | R0C2 | R0C1 | R0C0 |


## SAS controller

LSI SAS 2008

- SN SP51115003
- H5-25249-01K
- P/N LSI00194

|         |      |
|---------|------|
| Top     | R3C* |
| Bottom  | R2C* |


|      |         |              |             |
|------|---------|--------------|-------------|
|Slot0 | lemons  | ST3000VN000  | Z31004EY    |
|Slot1 | lemons  | ST3000VN000  | Z3100EJJ    |
|Slot4 |         |              |             |
|Slot5 | lemons  | ST3000VN000  | ST3100EQJ   |
|Slot6 | lemons  | ST3000VN000  | ST3100DCL   |
|Slot7 | lemons  | ST3000VN000  | ST3100FIV   |


# Bay layout


   | C0 | C1 | C2 | C3  
---|---|--- |--- | ---
R3 |   |    |    | L8
R2 | L9|    |    |    
R1 | L5| L6 | L7 |  Sys
R0 | L1| L2 | L3 |  L4

L: Lemons  (4TB)
Sys: system drive SSD


# S/N of drives in bays

WD40EFRX
* r0
   * r0c0 WCC4E2NH22ND (L1)
   * r0c1 WCC4E5AU8Y8K (L2)
   * r0c2 WCC4E5SJCE4N (L3)
   * r0c3 WCC4E3ZH0P1K (L4)
* r1
   * r1c0 WCC4E5LSLTDN (L5)
   * r1c1 WCC4E5SJCUTS (L6)
   * r1c2 WCC4E5LSL184 (L7)
   * r1c3 WCC4E5LSLSUS (L8)
* r2
   * r2c0 WCC4E7ARPUY0 (L9)
* r3
   * r3c3  system drive (S1)


Raw: 32TB + spare

Raid6: 24TB


Bonnie++ Benchmarks
=====================

````

Version  1.97       ------Sequential Output------ --Sequential Input- --Random-
Concurrency   1     -Per Chr- --Block-- -Rewrite- -Per Chr- --Block-- --Seeks--
Machine        Size K/sec %CP K/sec %CP K/sec %CP K/sec %CP K/sec %CP  /sec %CP
wifo3-02        63G  1842  89 347029  24 175829  17  3404  89 495890  20 447.9  11
Latency              8735us   16764us   79061us   18280us   32499us     228ms
Version  1.97       ------Sequential Create------ --------Random Create--------
wifo3-02            -Create-- --Read--- -Delete-- -Create-- --Read--- -Delete--
              files  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP
                 16  7555  17 +++++ +++ 18185  27  7097  12 +++++ +++ 11216  21
Latency               349ms     127us   12376us     382ms      14us     490ms
1.97,1.97,wifo3-02,1,1438369124,63G,,1842,89,347029,24,175829,17,3404,89,495890,20,447.9,11,16,,,,,7555,17,+++++,+++,18185,27,7097,12,+++++,+++,11216,21,8735us,16764us,79061us,18280us,32499us,228ms,349ms,127us,12376us,382ms,14us,490ms
bonnie (END)
````



History
===========

## 2015-07-03

 * wifo3-2 installed with peaches drives and 4 lemon drives;
 * copied data from peaches onto lemons
 * checked data with md5

## 2015-07-08

 * removed peaches drives; added 5 more lemon drives (total lemon drives see above)
 * Added remaining lemons with,
   `mdadm --grow /dev/md/lemons --raid-devices=8 --add /dev/sd[abcdi]1`
 * this leaves one drive as a spare
 * Unfortunately I forgot to also set `--level=raid6` when I did this, will need to revisit this
 * This results in (from `cat /proc/mdstat`),
   ```
   md127 : active raid5 sdi1[9] sdd1[8] sdc1[7] sdb1[6] sda1[5](S) sdh1[4] sdg1[2] sde1[0] sdf1[1]
      11720647680 blocks super 1.2 level 5, 512k chunk, algorithm 2 [8/8] [UUUUUUUU]
   ```
   
 * Tuned filesystem for raid parameters,
   * Following https://raid.wiki.kernel.org/index.php/RAID_setup#ext2.2C_ext3.2C_and_ext4
   * chunk size = 512kB (from `mdstat`)
   * block size = 4kB (from `tune2fs -l`)
   * stride = chunk size / block size = 128
   * raid_devices = 8 (from `mdstat`)
   * stripe width = stride * (raid_devices - 2) = 768 (for raid6)
   * set with (after unmounting),
     `tune2fs -E stride=128,stripe-width=768 /dev/md/lemons1`

 * Upp'd the raid level to raid6 and add a write-intent bitmap:
     `mdadm /dev/md/lemons --grow --level=raid6 --backup-file=/root/md.backup`
 * This results in (from `cat /proc/mdstat`),
   ```
md127 : active raid6 sdi1[9] sdd1[8] sdc1[7] sdb1[6] sda1[5] sdh1[4] sdg1[2] sde1[0] sdf1[1]
      27348177920 blocks super 1.2 level 6, 512k chunk, algorithm 18 [9/8] [UUUUUUUU_]
   ```
 * Grow partition with parted
    ```
    ldietz@wifo3-02:/mnt$ sudo parted /dev/md/lemons
    GNU Parted 3.2
    Using /dev/md/lemons
    Welcome to GNU Parted! Type 'help' to view a list of commands.
    (parted) resizepart                                                       
    Partition number? 1                                                       
    End?  [12.0TB]? -1                                                        
    (parted) print                                                            
    Model: Linux Software RAID Array (md)
    Disk /dev/md/lemons: 28.0TB
    Sector size (logical/physical): 512B/4096B
    Partition Table: gpt
    Disk Flags: 

    Number  Start   End     Size    File system  Name    Flags
     1      1573kB  28.0TB  28.0TB  ext4         lemons
    ```

     
 * What next:
   * --- done grow the partition to fill the new space: `parted /dev/md/lemons resizepart 1 -1` (need to check this)
   * grow the filesystem in the partition: `resize2fs /dev/md/lemons1`

  * can view the status of the md volume with `cat /proc/mdstat`

## 2015-07-28

 * Finally have both peaches and lemons present due to installation of LSI
   SAS2008 controller
 * Unfortunately the lemons md volume was grown incorrectly,
   with `raid-devices = 9` whereas it should have been 8
 * Going to start again,
   ```
   $ mdadm --create /dev/md/lemons /dev/sd[lmnfbedac]1 --name=lemons --raid-devices=8 --spare-devices=1 --level=raid6 --bitmap=internal
   $ cat /proc/mdstat
   Personalities : [raid0] [raid6] [raid5] [raid4] 
   md126 : active raid6 sdn1[8](S) sdm1[7] sdl1[6] sdf1[5] sde1[4] sdd1[3] sdc1[2] sdb1[1] sda1[0]
         23441295360 blocks super 1.2 level 6, 512k chunk, algorithm 2 [8/8] [UUUUUUUU]
         [>....................]  resync =  0.0% (362012/3906882560) finish=1978.3min speed=32910K/sec
         bitmap: 30/30 pages [120KB], 65536KB chunk
   
   md127 : active raid0 sdh1[0] sdj1[4] sdi1[3] sdg1[2] sdk1[1]
         14651322880 blocks super 1.2 512k chunks
         
   unused devices: <none>
   ```
 * Create partition,
   ```
   $ parted -a optimal /dev/md/lemons
   (parted) mklabel gpt
   (parted) mkpart lemons xfs 0% 100%
   (parted) print
   Model: Linux Software RAID Array (md)
   Disk /dev/md/lemons: 24.0TB
   Sector size (logical/physical): 512B/4096B
   Partition Table: gpt
   Disk Flags: 
   
   Number  Start   End     Size    File system  Name    Flags
    1      3146kB  24.0TB  24.0TB  xfs          lemons
   ```
 * Create filesystem,
   ```
   $ sudo mkfs.xfs /dev/md/lemons1 
   ```
* Bonnie ++ (while rebuilding RAID6)

```
Version  1.97       ------Sequential Output------ --Sequential Input- --Random-
Concurrency   1     -Per Chr- --Block-- -Rewrite- -Per Chr- --Block-- --Seeks--
Machine        Size K/sec %CP K/sec %CP K/sec %CP K/sec %CP K/sec %CP  /sec %CP
wifo3-02        63G  1905  91 364656  24 119264  12  3457  86 306391  14 439.5  17
Latency              8631us   18055us     184ms   23985us   57565us   83422us
Version  1.97       ------Sequential Create------ --------Random Create--------
wifo3-02            -Create-- --Read--- -Delete-- -Create-- --Read--- -Delete--
              files  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP
                 16  8891  13 +++++ +++ 17536  23 11535  17 +++++ +++ 19876  28
Latency               391us     125us     128us     375us      20us      90us
1.97,1.97,wifo3-02,1,1438076799,63G,,1905,91,364656,24,119264,12,3457,86,306391,14,439.5,17,16,,,,,8891,13,+++++,+++,17536,23,11535,17,+++++,+++,19876,28,8631us,18055us,184ms,23985us,57565us,83422us,391us,125us,128us,375us,20us,90us
l
```


 * Copy peaches to lemons,
   ```
   $ rsync -a /mnt/peaches /mnt/lemons/peaches --info=progress2
   ```

## 2015-07-30

* RAID rebuilt, drive failed
````
ldietz@wifo3-02:~$ cat /proc/mdstat
Personalities : [raid0] [raid6] [raid5] [raid4] 
md126 : active raid6 sdn1[8] sdm1[7] sdl1[6](F) sdf1[5] sde1[4] sdd1[3] sdc1[2] sdb1[1] sda1[0]
      23441295360 blocks super 1.2 level 6, 512k chunk, algorithm 2 [8/8] [UUUUUUUU]
      bitmap: 12/30 pages [48KB], 65536KB chunk

md127 : active raid0 sdh1[0] sdj1[4] sdi1[3] sdg1[2] sdk1[1]
      14651322880 blocks super 1.2 512k chunks
      
unused devices: <none>
````
* suspecting issues with new LSI controller , [see blog](http://blog.disksurvey.org/blog/2014/03/27/sata-handling-of-medium-errors-log-info-0x0x31080000/)

* removed drive /dev/sdl1 and readded as spare
````
 sudo madm /dev/md/lemons --remove /dev/sdl1

 sudo mdadm --manage --add /dev/md/lemons /dev/sdl1
````

 
* ldietz@wifo3-02:~/wifo3-2$ sudo sudo mdadm --detail /dev/md/lemons
````
/dev/md/lemons:
        Version : 1.2
  Creation Time : Tue Jul 28 12:24:38 2015
     Raid Level : raid6
     Array Size : 23441295360 (22355.36 GiB 24003.89 GB)
  Used Dev Size : 3906882560 (3725.89 GiB 4000.65 GB)
   Raid Devices : 8
  Total Devices : 9
    Persistence : Superblock is persistent

  Intent Bitmap : Internal

    Update Time : Thu Jul 30 13:51:58 2015
          State : active 
 Active Devices : 8
Working Devices : 9
 Failed Devices : 0
  Spare Devices : 1

         Layout : left-symmetric
     Chunk Size : 512K

           Name : wifo3-02:lemons  (local to host wifo3-02)
           UUID : 116751fa:df64959f:cb3098a9:17f4e326
         Events : 24801

    Number   Major   Minor   RaidDevice State
       0       8        1        0      active sync   /dev/sda1
       1       8       17        1      active sync   /dev/sdb1
       2       8       33        2      active sync   /dev/sdc1
       3       8       49        3      active sync   /dev/sdd1
       4       8       65        4      active sync   /dev/sde1
       5       8       81        5      active sync   /dev/sdf1
       8       8      209        6      active sync   /dev/sdn1
       7       8      193        7      active sync   /dev/sdm1

       9       8      177        -      spare   /dev/sdl1
```
## 2015-07-31

* reseating cables and drives. trying to reproduce drive error

* Bonnie++ (without rebuilding RAID)

````
Version  1.97       ------Sequential Output------ --Sequential Input- --Random-
Concurrency   1     -Per Chr- --Block-- -Rewrite- -Per Chr- --Block-- --Seeks--
Machine        Size K/sec %CP K/sec %CP K/sec %CP K/sec %CP K/sec %CP  /sec %CP
wifo3-02        63G  1842  89 347029  24 175829  17  3404  89 495890  20 447.9  11
Latency              8735us   16764us   79061us   18280us   32499us     228ms
Version  1.97       ------Sequential Create------ --------Random Create--------
wifo3-02            -Create-- --Read--- -Delete-- -Create-- --Read--- -Delete--
              files  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP
                 16  7555  17 +++++ +++ 18185  27  7097  12 +++++ +++ 11216  21
Latency               349ms     127us   12376us     382ms      14us     490ms
1.97,1.97,wifo3-02,1,1438369124,63G,,1842,89,347029,24,175829,17,3404,89,495890,20,447.9,11,16,,,,,7555,17,+++++,+++,18185,27,7097,12,+++++,+++,11216,21,8735us,16764us,79061us,18280us,32499us,228ms,349ms,127us,12376us,382ms,14us,490ms
bonnie (END)
````

## 2015-09-03

* Moving system drive into bay R3C3
* Rebooting system - no changes needed.
````
ldietz@wifo3-02:~$  cat /proc/mdstat 
Personalities : [raid0] [raid6] [raid5] [raid4] 
md126 : active raid6 sdf1[5] sdm1[9](S) sdo1[7] sde1[4] sdp1[8] sdc1[2] sda1[0] sdb1[1] sdd1[3]
      23441295360 blocks super 1.2 level 6, 512k chunk, algorithm 2 [8/8] [UUUUUUUU]
      bitmap: 0/30 pages [0KB], 65536KB chunk

md127 : active raid0 sdi1[0] sdk1[4] sdj1[3] sdg1[2] sdl1[1]
      14651322880 blocks super 1.2 512k chunks
      
unused devices: <none>
````

## 2015-09-15
* Labeling hard drives (on bay)
* Running bonnie++ again
   * bonnie++ on system drive

````
Version  1.97       ------Sequential Output------ --Sequential Input- --Random-
Concurrency   1     -Per Chr- --Block-- -Rewrite- -Per Chr- --Block-- --Seeks--
Machine        Size K/sec %CP K/sec %CP K/sec %CP K/sec %CP K/sec %CP  /sec %CP
wifo3-02        63G   991  99 505650  48 248311  22  4198  99 686466  27 +++++ +++
Latency             18813us     159ms     397ms    3930us    7022us     169ms
Version  1.97       ------Sequential Create------ --------Random Create--------
wifo3-02            -Create-- --Read--- -Delete-- -Create-- --Read--- -Delete--
              files  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP
                 16 +++++ +++ +++++ +++ +++++ +++ +++++ +++ +++++ +++ +++++ +++
Latency               693us     587us     655us     936us      16us     923us
1.97,1.97,wifo3-02,1,1442285956,63G,,991,99,505650,48,248311,22,4198,99,686466,27,+++++,+++,16,,,,,+++++,+++,+++++,+++,+++++,+++,+++++,+++,+++++,+++,+++++,+++,18813us,159ms,397ms,3930us,7022us,169ms,693us,587us,655us,936us,16us,923us

````
    * bonnie ++ on lemons
````
Version  1.97       ------Sequential Output------ --Sequential Input- --Random-
Concurrency   1     -Per Chr- --Block-- -Rewrite- -Per Chr- --Block-- --Seeks--
Machine        Size K/sec %CP K/sec %CP K/sec %CP K/sec %CP K/sec %CP  /sec %CP
wifo3-02        63G   910  42 226340  16 128574  12  3216  86 527467  21 335.7  11
Latency              7421us   43874us     127ms   40307us   84960us   80635us
Version  1.97       ------Sequential Create------ --------Random Create--------
wifo3-02            -Create-- --Read--- -Delete-- -Create-- --Read--- -Delete--
              files  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP
                 16  6978  14 +++++ +++ 17601  28  7027  15 +++++ +++ 13611  21
Latency               310ms     133us   23978us     395ms      16us     808ms
1.97,1.97,wifo3-02,1,1442286737,63G,,910,42,226340,16,128574,12,3216,86,527467,21,335.7,11,16,,,,,6978,14,+++++,+++,17601,28,7027,15,+++++,+++,13611,21,7421us,43874us,127ms,40307us,84960us,80635us,310ms,133us,23978us,395ms,16us,808ms
````
## 2016-02-08

* System drive read-only, possibly related to excessive memory use and thrashing. (System drive on SAS controller)

* Reconsidering firmware issues, where controller is not truthful to the features it supports, therefore, when the OS uses the features, we get a ton of errors.

* SMART Tests run: long and short, both without errors

* Trying to run conveyance tests, but not supported. Possibly due to SCSI controller.

* Updating Linux kernel to 4.3, to enable the workaround for [bug 72341](https://bugzilla.kernel.org/show_bug.cgi?id=72341) (regarding Samsung EVO / System drive)

Still seeing CRC errors on system drive

````
[   14.513858] sd 0:0:1:0: [sdi] FAILED Result: hostbyte=DID_OK driverbyte=DRIVER_SENSE
[   14.513863] sd 0:0:1:0: [sdi] Sense Key : Aborted Command [current] 
[   14.513867] sd 0:0:1:0: [sdi] Add. Sense: Information unit iuCRC error detected
[   14.513870] sd 0:0:1:0: [sdi] CDB: Read(10) 28 00 02 c0 17 78 00 00 88 00
[   14.513872] blk_update_request: I/O error, dev sdi, sector 46143352
[   14.520939] mpt2sas0: log_info(0x31080000): originator(PL), code(0x08), sub_code(0x0000)
[   14.520943] mpt2sas0: log_info(0x31080000): originator(PL), code(0x08), sub_code(0x0000)

````

* Switching L8 with Sys bay to have system drive directly on the motherboard.

````
ldietz@wifo3-02:~$ cat /proc/mdstat
Personalities : [raid0] [raid6] [raid5] [raid4] 
md126 : active raid6 sdi1[3] sdn1[9](S) sdb1[1] sdg1[7] sde1[4] sdf1[5] sda1[0] sdo1[8] sdc1[2]
      23441295360 blocks super 1.2 level 6, 512k chunk, algorithm 2 [8/8] [UUUUUUUU]
      bitmap: 0/30 pages [0KB], 65536KB chunk

md127 : active raid0 sdj1[0] sdl1[4] sdk1[3] sdh1[2] sdm1[1]
      14651322880 blocks super 1.2 512k chunks
      
unused devices: <none>
````

### 2016-06-08

* Took out peaches drives (md126)

* Still errors in dmesg

````
[   14.403577] Installing knfsd (copyright (C) 1996 okir@monad.swb.de).
[   14.653388] ioatdma 0000:00:04.1: ioat_timer_event: Channel halted (0)
[   17.331809] ixgbe 0000:04:00.0 eth0: NIC Link is Up 1 Gbps, Flow Control: None
[   17.551721] ixgbe 0000:04:00.1 eth1: NIC Link is Up 1 Gbps, Flow Control: None
[   52.867144] usb 3-1: USB disconnect, device number 2
[ 4077.269050] mpt2sas0: log_info(0x31080000): originator(PL), code(0x08), sub_code(0x0000)
[ 4077.269076] sd 0:0:0:0: [sdh] FAILED Result: hostbyte=DID_SOFT_ERROR driverbyte=DRIVER_OK
[ 4077.269082] sd 0:0:0:0: [sdh] CDB: Read(16) 88 00 00 00 00 01 36 b2 4a 78 00 00 00 08 00 00
[ 4077.269087] blk_update_request: I/O error, dev sdh, sector 5212621432
[ 4077.269151] sd 0:0:0:0: [sdh] FAILED Result: hostbyte=DID_OK driverbyte=DRIVER_SENSE
[ 4077.269156] sd 0:0:0:0: [sdh] Sense Key : Aborted Command [current] 
[ 4077.269164] sd 0:0:0:0: [sdh] Add. Sense: Information unit iuCRC error detected
[ 4077.269169] sd 0:0:0:0: [sdh] CDB: Read(16) 88 00 00 00 00 01 36 b2 4a 50 00 00 00 28 00 00
[ 4077.269172] blk_update_request: I/O error, dev sdh, sector 5212621392
````
