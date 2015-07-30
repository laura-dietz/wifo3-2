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

|         |      |
|---------|------|
| Top     | R3C* |
| Bottom  | R2C* |




# Bay layout


   | C0 | C1 | C2 | C3  
---|---|--- |--- | ---
R3 | P4| P5 |    |  
R2 | L | P1 | P2 | P3 
R1 | L | L  | L  |  L
R0 | L | L  | L  |  L


L: Lemons  (4TB)
P: Peaches 1-5 (3TB)



# S/N of drives in bays

WD40EFRX
* r0
   * r0c0 WCC4E2NH22ND
   * r0c1 WCC4E5AU8Y8K
   * r0c2 WCC4E5SJCE4N
   * r0c3 WCC4E3ZH0P1K
* r1
   * r1c0 WCC4E5LSLTDN
   * r1c1 WCC4E5SJCUTS
   * r1c2 WCC4E5LSL184
   * r1c3 WCC4E5LSLSUS
* r2
   * r2c0 WCC4E7ARPUY0


Raw: 32TB + spare

Raid6: 24TB


Bonnie++ Benchmark
==================

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
 * Copy peaches to lemons,
   ```
   $ rsync -a /mnt/peaches /mnt/lemons/peaches --info=progress2
   ```

## 2015-07-30

* Raid reformat finishes
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

