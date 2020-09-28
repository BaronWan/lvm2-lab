# LVM2 Lab

> 實作者 : Wan Pei Chih 



## * RAID5 + LVM2, use xfs filesystem, and mounted to (/) root-directory, now happen the metadata I/O error in "xfs_trans_read_buf_map" error!

> 環境資料：
>
> - RAID 5 + LVM2
> - formatted xfs filesystem
> - mount to (/) root-directory

### - start to lab:

#### - 檢查發現：

1. `cat /proc/mdstat` happen ...

   ```
   md126: active raid5 sda1[0] sdb1[1] 
                   ... [3/2][UU_]
   md127: active raid5 sda2[0] sdb2[1]
                   ... [3/2][UU_]
   ```

   以上資訊說明了其中一顆硬碟找不到了

2. `mdadm --detail /dev/md126` happen ...

   ```
   ... active sync    /dev/sda1
   ... active sync    /dev/sdb1
   ... removed
   ```

   某一顆被宣告已遭移除!

3. 檢查 pvscan , vgscan , 等 lvm 都無標示任何錯誤。

#### - 問題分析與解決辦法：

- 問題分析：

  1. 可能為第三顆硬碟 sdc 發生故障，須更換之。並將原故障硬碟帶回進行檢測確認。
  2. 依據檢查所有可查詢的資源都顯示 LVM 是正常的，因此我們將不對 LVM 進行操作。磁碟的狀況已為 mdadm (soft raid) 確實掌握，因此我們只須就 新的裝置，正確的 disk to partition 後，重新讓 mdadm 管理到正確對應的 md 裝置中，然後對其 xfs filesystem 進行修復的工程，如果一切順利，將與 LVM 無任何事。

- 解決方式 (步驟實錄)：

  ※ 由於此雖是屬於掛載在 root filesystem (/) 根，無法直接進行 **umount** 等操作，須由救援光碟開機後，載入 lvm service，並進行相關作業。

  1. `fdisk -l /dev/sda` 查看並紀錄好 sda partition 資訊。

  2. 關機，並確認 sdc 對應的實體裝置位置 (留意主機板上或是磁碟裝置卡上標示的裝置順序 1,2,3,4, **最好的作法是在對應的磁碟機上一樣貼上 1,2,3,4 來確認順序**)，依系統顯示的裝置環境順序依序是 sda, sdb, sdc, 所以 sdc 應該是標示 3 的位置。將 3 位置的**磁碟機排線及電源排除後，主機重新上電開機確認是否可正常開機無誤**，如此可進一步確認拆下的裝置是正確的。確認無誤後，替換上新的硬碟後，主機再次上電開機並進入系統。
     **磁碟位置確認是相當重要的，拔錯正常的磁碟裝置，嚴重是會造成系統及資料無可挽回的危機。因此最好從一開始的佈建工程就已確認並標示完善，以免後續其他工程人員施做的困難。**

  3. 確認新的硬碟裝置理論上應該一樣是 sdc , 進行 disk partition 作業：

     ```shell
     $> parted /dev/sdc
     (parted)> p		# 確認空間無誤
     
     # 此例為 xfs 屬性, Start 為 1049KB, End 為 264MB
     (parted)> mkpart primary xfs 1049KB 264MB
     
     # 設定一樣的旗標值, 此例 sdc1 為 (boot, raid)
     (parted) set 1 boot
     (parted) set 1 raid
     
     # 此例 sdc2 的 start 為 264MB, end 為 21.5GB
     (parted)> mkpart primary xfs 264MB 21.5GB
     
     # 一樣為 sdc2 標示好 flag: raid
     (parted)> set 2 raid
     (parted)> p		# 再次確認
     (parted)> quit		# 退出 parted 
     
     ```

     

  4. 重新掛載進相關的 raid md:

     ```shell
     # 此例需要分別將 sdc1, sdc2 加入 md126, md127 
     # 此部份操作務必正確, 否則後續麻煩一籮筐
     mdadm --manage /dev/sdc1 --add /dev/md126
     mdadm --manage /dev/sdc2 --add /dev/md127
     ```

  5. 檢查當前 raid 狀態 :
     `mdadm --detail /dev/md126`

     ```
     ... active sync		/dev/sdc1	(成功!)
     ```

     `cat /proc/mdstat`

     ```
     ... [3/3] [UUU]		(表示都載入完成)
     ```

  6. 接下來要對 xfs 進行<u>修復</u>操作(**此部份必須在 single user mode 下才能完成操作 ! 掛載為 (/) root-directory 根位置，此時的系統將不允許對其進行 remount 的安全動作，故 必須經由救援光碟開機進入後進行操作**):

     - 開機選擇 [troubleshooting] --> [Rescue a CentOS system]
       選擇 1. [continue] 或 2. [read-only] 皆可，若選擇 1，則需要手動將該裝置重新掛載為 readonly mode。
       `mount -o remount,ro /dev/mapper/vg00root`
     - 確認掛載的裝置名稱資訊
       官方的救援光碟正常應該是會自動取得 local OS 的裝置掛載資訊並正確掛載之

     ```shell
     $> df 
     /dev/mapper/vg00root	xfs  /mnt/sysimage
     ```

     - 使用 `xfs_repair` 來修復 xfs 錯誤：
       **當前磁碟裝置必須為 read-only mode**

       ```shell
       $> xfs_repair -d /dev/mapper/vg00root
       ...
       # 細節務必瀏覽確認, 確認修復無誤
       ...
       Repair of readonly mount complete. Immediate reboot encouraged.
       
       ```

       (完成後，執行 `exit` 後重新開機進入正常系統)

  7. 首次重新開機會提示錯誤後，自動重新開機，之後在進入系統時，系統也會發出警告，登入後檢查:

     - LVM 的磁碟檢查確認：

       `lvmdiskscan`

       ```
       /dev/md126 [       498.00 MiB]
       /dev/md127 [       <39.45 GiB] LVM physical volume
       0 disks
       1 partition
       0 LVM physical volume whole disks
       1 LVM physical volume
       ```

       (其他相關 `lvs`, `vgs`, `pvs`... 等最好都檢查確認)

(此部份 lab 已完成～)

-----

## * 建立 LVM Configuration Metadata 備份/還原 

預設備份將放於 "/etc/lvm/backup/" 中。

### - 備份

`vgcfgbackup`

### - 還原

`vgcfgrestore`

----

## * LVM 訊息診斷工具

`lvmdump`

此外，可另外執行相關確認工作：

`lvs -v`

`pvs -a`

`dmsetup info -c`

- 確認目前 LVM 的配置資訊:
  `lvm dumpconfig`
- 檢測 /etc/lvm/backup/ 下的 metadata 配置文件資訊。



------

