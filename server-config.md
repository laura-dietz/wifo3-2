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
|      |      | SSD | R2C0 |   | R0C3 | R0C2 | R0C1 | R0C0 |



# Bay layout


   | C0 | C1 | C2 | C3  
---|---|--- |--- | ---
R3 |   |    |    |  
R2 | X |    |    | 
R1 | X | X  | X  |  X
R0 | X | X  | X  |  X




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

     
 * What next:
   * grow the partition to fill the new space: `parted /dev/md/lemons resizepart 1 -1` (need to check this)
   * grow the filesystem in the partition: `resize2fs /dev/md/lemons1`

  * can view the status of the md volume with `cat /proc/mdstat`
 
