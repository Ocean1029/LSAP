# LSAP HW2 作業

> 姓名：曾煥軒  
> 系級：資管三 B12705002  
> 日期：2025/10/01  
> 檔案來源：[HW2 作業說明](https://hackmd.io/@LiSeng/S11kuPRjlx)

---

## 作業環境資訊

- OS: Ubuntu / VM  stn09
- 必要套件：`nfs-kernel-server`、`nfs-common`、`autofs`

## Module 1 謎樣的硬碟


1. **用 `curl` 指令下載硬碟檔**
![image](https://hackmd.io/_uploads/r1xDlrKhel.png)

2. **掛載硬碟檔**
loop device 是一種虛擬的區塊裝置，可以把一個普通檔案「掛載」成像實體磁碟一樣使用。這樣，系統會把檔案當作一顆磁碟來看待。最常見的用途是掛載 ISO 映像檔 或 磁碟映像檔（.img 檔）。接下來我們把剛剛拿到的 .img 檔案，使用 `losetup -fP diskX.img`，掛載到 loop device 上面，並用 `losetup -a` 列出來對應關係。
![image](https://hackmd.io/_uploads/SJvilrFnll.png)
list block devices 時，就可以看到這四個虛擬磁碟被掛載。
![image](https://hackmd.io/_uploads/H1rD4mqheg.png)

3. **檢查每顆硬碟**
`sudo mdadm --examine /dev/loopX`
看它是不是有 RAID superblock，如果有，會顯示 RAID level（RAID 0、1、5、6…）、array UUID、slot number 等。
	* loop0 的硬碟如下，可以看到在這個 superblock 中顯示 raid level 是 raid5，也代表這顆硬碟不只沒有損壞：
![image](https://hackmd.io/_uploads/S1FkUN92ee.png)
	* loop1 找不到 superblock，則代表該硬碟損毀。
![image](https://hackmd.io/_uploads/rJhUUNq2lx.png)
	* loop2/3 的結果和 loop0 相似，並且他們有相同的 Array UUID，代表這三顆硬碟恰好組成了一個 RAID 5。
	
4. **組 RAID**
![image](https://hackmd.io/_uploads/B1ZL1Iqhxe.png)
用`cat /proc/mdstat` 可以看到 RAID 已經成功啟動。
![image](https://hackmd.io/_uploads/HkAjeL9nge.png)
`sudo mdadm --detail /dev/md1` 看 array 的狀態。
![image](https://hackmd.io/_uploads/rk-mW853xe.png)

5. **掛載 RAID**
`sudo mount /dev/md1 /mnt`。

6. **找 secret code**
![image](https://hackmd.io/_uploads/S1MPzU52ee.png)
`cat /mnt/treasure.txt`
![image](https://hackmd.io/_uploads/HkiOGIqneg.png)

### Module 1 Result
- The secret code is {IL0V3L5AP_HAPPYHOLIDAY9/28}
- The broken hard disk is loop1, which is disk2

7. **清理環境**

* `sudo umount /mnt`
* `sudo mdadm --stop /dev/md1`
* `sudo losetup -d /dev/loopX` 把所有硬碟解除。

## Module 2 NFS auto mounting 

### Server Side

1. **建立目錄**
![image](https://hackmd.io/_uploads/r1GXGF5nxe.png)

2. **設定分享規則**
`apt install nfs-kernel-server` 後，設定 `/etc/exports` 檔案。
這個設定檔包含了：哪些目錄要分享出去，要分享給誰（允許的 client IP 或網段）與存取權限（讀寫、root 權限如何處理…）等等資訊。此處設定 `/srv/nfsroot *(rw,sync,no_subtree_check,no_root_squash)`代表我們開放所有人都可以 access to `srv/nfsroot` ，並且每個 client 都有 `rw` 的權限。
![image](https://hackmd.io/_uploads/rkRYXK9hxl.png)
run `exportfs` 讓設定生效。
![image](https://hackmd.io/_uploads/HJMp4Y53le.png)

3. **啟動服務**
![image](https://hackmd.io/_uploads/B1TESK92ll.png)
從圖中我們可以知道，這個 systemctl 有正常運作，我們可以用 `showmount -e` 檢查 NFS export 的目錄，也如同我們的想像。
![image](https://hackmd.io/_uploads/HJtqHt92ll.png)

### Client Side

1. **編輯 Master map**
首先先 `install -y autofs nfs-common` 安裝必要套件，接著就可以編輯 master map
	```
	/mnt/nfs   /etc/auto.nfs   --timeout=60
	/-         /etc/auto.direct
	```	
	- `/mnt/nfs` → 交給 `/etc/auto.nfs` 管理，這是 indirect map。
	- `/-` → 交給 `/etc/auto.direct` 管理，這是 direct map。
	- `-\-timeout=60` → 如果 60 秒內沒人用這個掛載點，autofs 就會自動卸載它。

	接下來我們就需要設定 `etc/auto.nfs` 和 `/etc/auto.direct`，定義不同檔案的掛載點。

2. **編輯 Indirect Map**
Indirect map 是那些要掛載在 `/mnt/nfs` 下的的設定檔案。在作業中大部分的路徑都掛載在這。左邊的欄位定義 client 的路徑，右邊的欄位定義 server 的 ip 和目錄，這裡因為在本機模擬，所以用 localhost。
![image](https://hackmd.io/_uploads/H1mKg9cnxg.png)


3. **編輯 Direct Map**
Direct map 是那些要掛載在絕對路徑下的設定檔案。在作業中， `/webdata/` 和 `/backups/`都想要直接放在根目錄，而非掛載點。
![image](https://hackmd.io/_uploads/S1jhAFqhxl.png)



4. **啟動服務**
![image](https://hackmd.io/_uploads/H1qYnKc3xx.png)

5. **使用 NFS**
當進入到目錄 `/mnt/nfs` 時，會先發現沒有檔案。
![image](https://hackmd.io/_uploads/SybkAt5heg.png)
但我們可以 `cd` 到 `public`，連結到 NFS Server，我們就可以看到 README.txt，或新增修改檔案。
![image](https://hackmd.io/_uploads/SkceRK9nxl.png)
也因為我們在 mastermap 中設定 timeout 是 60 秒，因此 60 秒後就會自動取消掛載。
![image](https://hackmd.io/_uploads/rkDKCt9ngg.png)
掛在根目錄的檔案也可以正常連結進去。
![image](https://hackmd.io/_uploads/BJzoycqnlx.png)
而路徑中的其他資料夾也可以正常進入和顯示，如 `projects`：
![image](https://hackmd.io/_uploads/S1boe993gx.png)


