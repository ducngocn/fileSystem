## Nội dung

- [Demo flexible capacity](#flexible-capacity)

- [Demo Resize volume](#resize-volume)

- [Demo Di chuyển data từ PV này sang PV khác](#di-chuyển-dữ-liệu-từ-pv-này-sang-pv-khác-và-xoá-pv-mà-không-bị-downtime)

- [Giảm kích thước LV](#giảm-kích-thước-của-lv)
### Flexible capacity  

#### Tạo pv từ phân vùng/ổ

- **Cú pháp**
```
sudo pvcreate <phân vùng>
```

Tạo pv /dev/sda4 và /dev/sda5 (cả 2 đều chưa format filesystem)
```
ngocduc@linux:~$ sudo pvcreate /dev/sda4
  Physical volume "/dev/sda4" successfully created.

ngocduc@linux:~$ sudo pvcreate /dev/sda5
  Physical volume "/dev/sda5" successfully created.
```

#### Xem các Physical volume vừa tạo

- **Cú pháp**

```
sudo pvs
```

```
ngocduc@linux:~$ sudo pvs
  PV         VG        Fmt  Attr PSize   PFree
  /dev/sda3  ubuntu-vg lvm2 a--  <28.00g 14.00g
  /dev/sda4            lvm2 ---    2.00g  2.00g
  /dev/sda5            lvm2 ---    3.00g  3.00g
```

**Giải thích các trường** 

| Cột       | Ý nghĩa                                |
| --------- | -------------------------------------- |
| **PV**    | Thiết bị vật lý (phân vùng hoặc ổ đĩa) |
| **VG**    | Volume Group mà PV đó thuộc về         |
| **Fmt**   | Định dạng LVM metadata - cách LVM lưu trữ thông tin về  PV/VG/LV (thường là lvm2)           |
| **Attr**  | Trạng thái PV        |
| **PSize** | Tổng dung lượng của PV                 |
| **PFree** | Dung lượng trống còn lại trong PV      |

**Giải thích cụ thể các giá trị trong trường Fmt và Attr**:

- **Các giá trị fmt có thể gặp:**

| Fmt       | Ý nghĩa                                                                                           |
| --------- | ------------------------------------------------------------------------------------------------- |
| **lvm2**  | (Hiện nay phổ biến nhất) — dùng hệ thống metadata kiểu mới, hỗ trợ snapshot, thin pool, RAID, ... |
| **lvm1**  | (Cũ, gần như không dùng nữa) — định dạng LVM thế hệ đầu, chỉ còn để tương thích ngược.            |
| *(trống)* | Ổ này **chưa được khởi tạo làm PV** hoặc bị lỗi metadata.                                         |

- **Trạng thái Attr**

| Ký tự     | Vị trí | Ý nghĩa                                                                                                                                              | Ví dụ |
| --------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ----- |
| **a / x / -** | 1st    | **Trạng thái hoạt động của PV**:<br>• `a` = *active* (đang hoạt động, được dùng trong VG)<br>• `x` = *missing* (thiếu — ví dụ một ổ trong VG bị mất) •<br>• `-` = *không active* (là PV nhưng chưa được gán vào VG nào)<br> | `a--` |
| **m / -** | 2nd    | **Có được ghép mirror không**:<br>• `m` = chứa dữ liệu mirrored<br>• `-` = không                                                                     | `a-m` |
| **r / -** | 3rd    | **Read-only hay không**:<br>• `r` = PV chỉ đọc<br>• `-` = đọc/ghi bình thường                                                                        | `a-r` |


**Kí tự 'x' tại vị trí 1 có thể mô tả như sau:** 

- Giả sử VG này bao gồm 3 Physical Volume: sdb, sdc, sdd.

- Rút nhầm ổ /dev/sdd ra, hoặc ổ đó bị hỏng, Nhưng hệ thống vẫn có metadata nói rằng VG này có 3 PV.

- Lúc này khi kiểm tra lại sẽ xuất hiện 'x'

**Kí tự 'm' tại vị trí 2 có thể mô tả như sau:**
- Giả sử có 1 ổ đĩa duy nhất (/dev/sdb) chứa dữ liệu quan trọng, nếu ổ này bị hỏng vật lý thì toàn bộ dữ liệu mất sạch.

- Để tránh mất dữ liệu, sẽ thêm một ổ đĩa giống hệt (/dev/sdc), nếu ghi dữ liệu LVM sẽ ghi đồng thời vào cả 2 ổ đó được gọi là **mirror**.

- Khi hiện 'm' lên tức là LV này đang được nhân bản sang ổ khác.  

>LVM mirror có tồn tại và hữu ích, nhưng trong thực tế hiện nay người ta ít dùng trực tiếp, vì có những lựa chọn khác tốt hơn, ổn định hơn hoặc dễ quản lý hơn. Nhân bản sang ổ khác tốn gấp đôi dung lượng.

#### Đưa các Physical volume (PV) thành Volume Group (VG)

- **Cú pháp**
```
sudo vgcreate <tên VG> <các phân vùng/ổ đĩa>
```

**Đưa /dev/sda4 và /dev/sda5 vào một volume group (VG)**

```
ngocduc@linux:~$ sudo vgcreate vgdata /dev/sda4 /dev/sda5
  Volume group "vgdata" successfully created
```

#### Xem Volume group (VG) đã tạo

- **Cú pháp** 
```
sudo vgs
```

```
ngocduc@linux:~$ sudo vgs
  VG        #PV #LV #SN Attr   VSize   VFree
  ubuntu-vg   1   1   0 wz--n- <28.00g 14.00g
  vgdata      2   0   0 wz--n-   4.99g  4.99g
```

**Giải thích các trường** 

| Trường    | Giải thích                        | Ghi chú                                                |
| --------- | --------------------------------- | ------------------------------------------------------ |
| **VG**    | Tên Volume Group                  | Tên bạn đặt (ví dụ: vgdata)                          |
| **#PV**   | Số lượng Physical Volume trong VG | VD: `2` nghĩa là có 2 PV (ổ đĩa vật lý hoặc phân vùng) |
| **#LV**   | Số lượng Logical Volume trong VG  | Mỗi LV là 1 “ổ đĩa ảo” mà OS có thể dùng (mount được)  |
| **#SN**   | Số lượng snapshot                 | Nếu bạn chưa dùng snapshot thì sẽ là `0`               |
| **Attr**  | Thuộc tính (Attribute) của VG     | Gồm 6 ký tự, ví dụ `wz--n-` (giải thích ở dưới)     |
| **VSize** | Tổng dung lượng VG                | Tổng cộng tất cả các PV trong VG                       |
| **VFree** | Dung lượng trống chưa cấp phát    | Dung lượng còn lại có thể dùng để tạo LV mới           |


- **Chi tiết trường Attr**

| **Vị trí** | **Ký tự**             | **Ý nghĩa**                                | **Giải thích chi tiết**                                                                               |
| ---------- | --------------------- | ------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| **1**      | `w` / `r`             | **Quyền truy cập (access permission)**     | `w` = VG cho phép **ghi & đọc**, `r` = chỉ **đọc** (read-only).                                       |
| **2**      | `z` / `n`             | **Khả năng mở rộng (resizable)**           | `z` = có thể thêm/bớt Physical Volume, `n` = không thể (not resizable).                               |
| **3**      | `p` / `-`             | **Trạng thái thiết bị (partial)**          | `p` = thiếu 1 hoặc nhiều Physical Volume trong VG (VD: ổ hỏng, mất kết nối).                          |
| **4**      | `c` / `l` / `s` / `-` | **Chính sách phân bổ (allocation policy)** | `c` = contiguous, `l` = cling, `s` = strict, `-` = normal. ( Không liên quan đến snapshot.)          |
| **5**      | `c` / `-` / `n`             | **Clustered**                              | `c` = VG thuộc **Clustered LVM**, `-` or `n` = không có cluster. (Hiếm gặp, chủ yếu trong môi trường HA hoặc SAN.) |
| **6**      | `x` / `-`             | **Exported flag**                          | `x` = VG đã được **export** (ẩn, chưa dùng), `-` = bình thường.                                       |


⚠️ vị trí 4,5 hiếm gặp, chuyên sâu về lĩnh vực nhỏ, tạm thời bỏ qua.

**Vị trí 1:**

Kí tự `r`:

- Nếu trong trường hợp là `r` thì có nghĩa là LVM không được phép ghi hoặc thay đổi metadata của Volume Group đó mà chỉ được đọc dữ liệu(read-only).

- Các thao tác sẽ không được thực hiện khi vị trí 1 là `r`: Thêm PV vào VG (vgextend), Mở rộng LV (lvextend), Xóa LV (lvremove), Tạo LV mới (lvcreate),... 

Kí tự `w`:

- Nếu trong trường hợp là `w` thì đây là chế độ mặc định cho phép thực hiện mọi thao tác không hạn chế như `r`.

**Tại vị trí 2:**

Kí tự `z`:

- Cho phép thay đổi kích thước của Volume group (thêm, bớt physical volume)

Kí tự `n`:

- Không cho phép thay đổi kích thước của Volume group (không thêm, bớt physical volume)

**Tại vị trí 3**

Kí tự `p`:  

- Thông báo rằng một phần dữ liệu hoặc metadata của VG bị mất khả năng truy cập do một trong các PV này bị mất, hỏng, hoặc không kết nối được (ví dụ ổ đĩa rút ra, lỗi I/O, hoặc hỏng partition). 

- LVM vẫn “nhớ” rằng VG này có tồn tại PV đó, nhưng khi kiểm tra lại, nó không thấy thiết bị vật lý đó nữa.

Kí tự `-`:

- Volume group hoàn toàn bình thường.


**Tại vị trí 6**

Kí tự `x`:

- exported có nghĩa là tạm thời ngưng sử dụng vg này trên máy này, không hề di chuyển hay xoá dữ liệu gì cả, nó sẽ: 
  - Đánh dấu VG là “exported” (đặt cờ x trong Attr).
  - Hệ thống Gỡ VG ra khỏi danh sách quản lý của LVM trên máy hiện tại.
  - Giữ nguyên metadata, PV, LV — không xóa dữ liệu.

- exported này thường được sử dụng trong trường hợp muốn toàn vẹn dữ liệu để chuyển ổ sang thiết bị mới, tránh xung đột (hiếm gặp trong môi trường thực tế).
#### Tạo Logical Volume (LV)

- Cú pháp

```
sudo lvcreate -L <dung lượng> -n <tên muốn đặt> <tên vg muốn tạo lv>
```

**Tạo logical volume có tên là LV từ Volume group vgdata đã tạo** 
```
ngocduc@linux:~$ sudo lvcreate -L 2.5G -n LV vgdata
  Logical volume "LV" created.
```

#### Xem Logical Volume (LV) đã tạo

- **Cú pháp** 

```
sudo lvs
```

```
ngocduc@linux:~$ sudo lvs
  LV        VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  ubuntu-lv ubuntu-vg -wi-ao---- <14.00g
  LV        vgdata    -wi-a-----   2.50g
```

### Ý nghĩa các trường

| Trường                                  | Ý nghĩa                                                   | Giải thích cụ thể              |
| --------------------------------------- | --------------------------------------------------------- | ------------------------------ |
| **LV**                                  | Tên Logical Volume                                        | `ubuntu-lv`, `vgdata`          |
| **VG**                                  | Volume Group chứa LV                                      | `ubuntu-vg`, `vgdata`          |
| **Attr**                                | 10 ký tự thể hiện trạng thái LV                           | Đây là phần quan trọng nhất |
| **LSize**                               | Dung lượng của LV                                         | `14.00g`, `2.50g`              |
| **Pool / Origin / Data% / Meta% / ...** | Các thông tin thêm (nếu là snapshot, thinpool, mirror...) | Khi bình thường thì trống      |


- **Chi tiết trường Attr**

>Trong Attr sẽ có 10 vị trí, các kí tự từ vị trí 7 trở đi hiếm gặp

| **Vị trí** | **Ký tự thường gặp nhất**    | **Ý nghĩa thực tế (chuẩn)**                               | **Ghi chú**                                    |
| ---------- | ---------------------------- | --------------------------------------------------------- | ---------------------------------------------- |
| **1**      | `-`, `m`, `s`, `t`, `T`, `r` | Kiểu LV (linear, mirror, snapshot, thin, thin pool, RAID) | Quan trọng nhất để nhận dạng loại LV           |
| **2**      | `w`, `r`                     | Quyền ghi/đọc (write/read-only)                           | `w` mặc định, `r` hiếm                         |
| **3**      | `i`, `s`                | Allocation/permission inheritance hoặc trạng thái         | `i` = inherited, `s` = suspended |
| **4**      | `a`, `I`                     | Trạng thái kích hoạt (active/inactive)                    | Active = dùng được, mount được                 |
| **5**      | `o`, `t`, `T`, `-`           | Loại quan hệ (origin, thin, thin pool)                    | Phân biệt snapshot gốc hoặc thin LV            |
| **6**      | `s`, `e`, `-`, `o`                | Snapshot hoặc external origin                             | Thường chỉ thấy khi snapshot có liên quan      |

**Vị trí 1:**

ký tự `m`:

- Cho biết logical volume đó là mirrored volume, tức là dữ liệu được nhân bản (mirror) lên nhiều ổ đĩa vật lý/phân vùng khác nhau.

- Khi ghi dữ liệu vào logical volume, LVM sẽ đồng thời ghi dữ liệu đó lên các ổ đĩa/phân vùng mirror để đảm bảo tính sẵn sàng và an toàn dữ liệu.

- Tổng dung lượng thực tế sử dụng sẽ bằng:
(kích thước LV) × (số mirror + 1).

Kí tự `-`

- Là logical volume thông thường không có tính năng đặc biệt, không phải thin, mirrored, snap shot,...

- Cách dữ liệu được lưu trữ trong LV bình thường:

  - Linear: mặc định, dữ liệu được nối liên tục trên các PV (thường gặp nhất).

  - Striped: có thể sử dụng, tức dữ liệu xen kẽ trên nhiều PV để tăng tốc độ, nhưng ít gặp hơn.

  - linear và striped có doc riêng.

Kí tự `s` - snapshot LV

- Kí tự này cho biết rằng LV hiện tại đang là snapshot - bản sao tạm thời của một LV khác, thường được dùng làm backup, rollback, testing không làm LV chính.

- Các thay đổi trên LV gốc không làm mất dữ liệu snapshot, vì snapshot lưu trạng thái LV tại thời điểm nó được tạo.

**Vị trí 2:**

Kí tự `r`:
- chỉ cho phép đọc không được ghi, sửa, xoá dữ liệu trên LV đó.

Kí tự `w`;

- Gồm tất cả các thao tác đọc, ghi, sửa, xoá.

**Vị trí 3:**

Kí tự `i` - inherited 

- LV thừa kế các đặc điểm từ Volume Group (VG) hoặc từ LV cha (ví dụ snapshot hoặc mirror).

- Những thứ mà được thừa kế:

1. Allocation policy (cách phân bổ PE):

    - Linear hay Striped, hay RAID type.

    - Khi bạn tạo LV mới mà không chỉ định rõ, nó sẽ thừa hưởng kiểu allocation mặc định từ VG.

2. Permission / access policy

    - LV có thể ghi (w) hay chỉ đọc (r) → nếu không override, LV thừa hưởng quyền truy cập từ VG hoặc LV cha.

3. Các thiết lập khác liên quan đến LV

    - Ví dụ: metadata flags, snapshot hoặc thin LV có thể inherit từ LV gốc.

    - Không phải trạng thái active/suspended trực tiếp (đó là a hoặc s), mà là cách LV được thiết lập khi khởi tạo.


Kí tự `s` - suspended

- LV đang tạm dừng, không active.
- Khi ở trạng thái này LV không thể đọc, ghi, sửa, xoá dữ liệu.
- Thường do sysadmin thao tác thủ công khi thay đổi kích thước, di chuyển, hoặc tạo snapshot.

### Resize volume

#### Demo mở rộng vg và lv

##### **1. Mở rộng VG**

- Ban đầu có vg_test gồm /dev/sda3 và /dev/sda4

```
ducnn@linuxfilesystem:~$ sudo pvs
  PV         VG      Fmt  Attr PSize  PFree
  /dev/sda3  vg_test lvm2 a--  48.00m 16.00m
  /dev/sda4  vg_test lvm2 a--  56.00m 56.00m

root@linuxfilesystem:~# vgs
  VG      #PV #LV #SN Attr   VSize   VFree
  vg_test   2   1   0 wz--n- 104.00m 72.00m
```

- Tạo thêm phân vùng mới /dev/sda5 và add thêm vào vg_test

- Cú pháp thêm phân vùng mới vào vg
```
vgextend <tên vg> <tên phân vùng>
```

```bash
root@linuxfilesystem:~# vgextend vg_test /dev/sda5
  Volume group "vg_test" successfully extended

root@linuxfilesystem:~# vgs
  VG      #PV #LV #SN Attr   VSize   VFree
  vg_test   3   1   0 wz--n- 172.00m 140.00m
```

Lúc này thì vg_test đã lên thành 3 PV, tuy nhiên phải đưa /dev/sda5 thành physical volume trước khi add vào VG.

##### **2. Mở rộng LV**
- Hiện đã tạo sẵn một lv_test

```
root@linuxfilesystem:~# lvs
  LV      VG      Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_test vg_test -wi-a----- 32.00m
```
- Cú pháp extend:

- Mở rộng LV bằng dung lượng cố định

```
lvextend -L +<kích thước vd: 1G> <đường dẫn tới LV>
```

- Mở rộng lv_test thêm 40M

```
root@linuxfilesystem:~# lvextend -L +40M /dev/vg_test/lv_test
  Size of logical volume vg_test/lv_test changed from 32.00 MiB (8 extents) to 72.00 MiB (18 extents).
  Logical volume vg_test/lv_test successfully resized.

root@linuxfilesystem:~# lvs
  LV      VG      Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_test vg_test -wi-a----- 72.00m
```
##### **3. Mở rộng LV format ext4**

- Lưu ý: ext4 và xfs có cơ chế extend online tức là mở rộng phân vùng mà fileSystem đó quản lý, đối với các loại khác không hỗ trợ cơ chế này phải umount.

- format lv_test sang ext4 rồi mount vào /mnt/mount_point

```
root@linuxfilesystem:~# df -h /dev/vg_test/lv_test
Filesystem                   Size  Used Avail Use% Mounted on
/dev/mapper/vg_test-lv_test   64M   24K   59M   1% /mnt/mount_point
```
**Mở rộng ext4 khi nào ?**

- Trong trường hợp phân vùng ext4 quản lí sắp đầy, muốn mở rộng thêm thì phải mở rộng LV mà format trước sau đó mới mở rộng phân vùng ext4 quản lí.

- Trong trường hợp chỉ mở rộng LV mà không mở rộng ext4, ext4 lúc này chỉ biết tới phân vùng hiện tại nó có chứ không biết tới LV đã được mở rộng, bởi vậy cần mở rộng thêm cả phân vùng mà ext4 quản lí.

**Bước 1: Extend LV thêm 30M**

```
root@linuxfilesystem:~# lvextend -L +30M /dev/vg_test/lv_test
  Rounding size to boundary between physical extents: 32.00 MiB.
  Size of logical volume vg_test/lv_test changed from 72.00 MiB (18 extents) to 104.00 MiB (26 extents).
  Logical volume vg_test/lv_test successfully resized.
```

**Bước 2: Resize filesystem ext4 online**

- **Cú pháp** 

```
resize2fs <đường dẫn tới lv> <kích thước mở rộng>
```
Trong trường hợp muốn mở rộng hết dung lượng còn lại thì không thêm trường <kích thước mở rộng>

```bash
root@linuxfilesystem:~# resize2fs /dev/vg_test/lv_test
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/vg_test/lv_test is mounted on /mnt/mount_point; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 1
The filesystem on /dev/vg_test/lv_test is now 26624 (4k) blocks long.
```

- Trước khi mở rộng fs ext4

```bash
root@linuxfilesystem:~# df -h /dev/vg_test/lv_test
Filesystem                   Size  Used Avail Use% Mounted on
/dev/mapper/vg_test-lv_test   64M   24K   59M   1% /mnt/mount_point
```

- Sau khi mở rộng 
```bash
root@linuxfilesystem:~# df -h /dev/vg_test/lv_test
Filesystem                   Size  Used Avail Use% Mounted on
/dev/mapper/vg_test-lv_test   96M   24K   91M   1% /mnt/mount_point
```

### Di chuyển dữ liệu từ PV này sang PV khác và xoá PV mà không bị downtime

- Demo chuyển dữ liệu từ PV này sang PV khác trong cùng VG và xoá PV cũ đi.

- Ban đầu đang có 4 phân vùng nằm trong cùng một VG như sau:

```bash
root@linuxfilesystem:~# pvs -o+pv_used
  PV         VG      Fmt  Attr PSize    PFree    Used
  /dev/sda3  vg_test lvm2 a--    48.00m   48.00m     0
  /dev/sda4  vg_test lvm2 a--    56.00m       0  56.00m
  /dev/sda5  vg_test lvm2 a--    68.00m   20.00m 48.00m
  /dev/sda6  vg_test lvm2 a--  1020.00m 1020.00m     0
```
- Chuyển data từ một PV sang PV khác có 2 cách dùng:

1. Chuyển toàn bộ PV A sang PV B, trường hợp này PV B phải có dung lượng lớn hơn hoặc bằng PV A.

- **Cú pháp** 
```
pvmove <nguồn> <đích>
pvmove /dev/sda1 /dev/sdb1
```

2. Chuyển data của PV A sang các PV trống còn lại 

```
pvmove /dev/sdX
```

**Chuyển dung lượng từ /dev/sda4 sang /dev/sda6**

- Ban đầu trước khi chuyển

```
root@linuxfilesystem:~# pvs -o+pv_used
  PV         VG      Fmt  Attr PSize    PFree    Used
  /dev/sda3  vg_test lvm2 a--    48.00m   48.00m     0
  /dev/sda4  vg_test lvm2 a--    56.00m       0  56.00m
  /dev/sda5  vg_test lvm2 a--    68.00m   20.00m 48.00m
  /dev/sda6  vg_test lvm2 a--  1020.00m 1020.00m     0
```

- Chuyển data từ sda4 sang sda6
```
root@linuxfilesystem:~# pvmove /dev/sda4 /dev/sda6
  /dev/sda4: Moved: 85.71%
```

- số 85.71% ám chỉ tiến trình đã đạt được 85.71% rồi.

- Sau khi chuyển 

```
root@linuxfilesystem:~# pvs -o+pv_used
  PV         VG      Fmt  Attr PSize    PFree   Used
  /dev/sda3  vg_test lvm2 a--    48.00m  48.00m     0
  /dev/sda4  vg_test lvm2 a--    56.00m  56.00m     0
  /dev/sda5  vg_test lv m2 a--    68.00m  20.00m 48.00m
  /dev/sda6  vg_test lvm2 a--  1020.00m 964.00m 56.00m
```

**Xoá /dev/sda4 ra khỏi VG**

- **Cú pháp**

```
vgreduce <tên VG> <tên LV>
```

```
root@linuxfilesystem:~# vgreduce vg_test /dev/sda4
  Removed "/dev/sda4" from volume group "vg_test"
```

- Lúc này vg_test chỉ còn 3 vg thay vì 4 như trước

```
root@linuxfilesystem:~# vgs
  VG      #PV #LV #SN Attr   VSize  VFree
  vg_test   3   1   0 wz--n- <1.11g <1.01g
```
  
### Giảm kích thước của LV 

**resize offline** có nghĩa là umount, sau đó dùng lệnh để thay đổi kích thước chứ không xoá phân vùng đó để tạo lại.

VD: resize phân vùng format ext4 thì sẽ phải umount, sau đó giảm kích thước filesystem đi tức là vùng mà filesystem đó quản lý, sau đó mới giảm kích thước của phân vùng đó đi.

Trong trường hợp của xfs, không hỗ trợ resize kể cả online hay offline, muốn giảm thì sẽ phải umount, backup, xoá phân vùng, tạo phân vùng mới có kích thước nhỏ hơn, format lại sau đó khôi phục.

Mấu chốt của resize offline là không xoá phân vùng.

**resize online** có nghĩa là thay đổi trực tiếp kích thước của filesystem và phân vùng mà không phải umount.


**Đối với LVM:**

- Có thể giảm kích thước của LV 

- Cú pháp giảm bao nhiêu dung lượng:

```
lvreduce --resizefs -L -<kích thước> tênvg/tênlv
lvreduce --resizefs -L -64M vg_test/lv_test
```

- Cú pháp giảm tới bao nhiêu (bỏ dấu '-' trước kích thước):

```
lvreduce --resizefs -L <kích thước> tênvg/tênlv
lvreduce --resizefs -L 80M vg_test/lv_test
```
>option --resizef: option này sẽ thay đổi kích thước của filesystem được format vào LV. 


**Trường hợp 1:** Giảm kích thước của lv khi chưa format fs và chưa có dữ liệu   
- Trường hợp này không cần option ``--resizefs`` vì nó chưa format filesystem

```bash
root@linuxfilesystem:~# lvs
  LV      VG      Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_test vg_test -wi-a----- 84.00m
```

```bash
root@linuxfilesystem:~# lvreduce -L -4M /dev/vg_test/lv_test
  WARNING: Reducing active logical volume to 80.00 MiB.
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce vg_test/lv_test? [y/n]: y
  Size of logical volume vg_test/lv_test changed from 84.00 MiB (21 extents) to 80.00 MiB (20 extents).
  Logical volume vg_test/lv_test successfully resized.
```

**Trường hợp 2:** giảm kích thước của lv khi format nhưng chưa mount 

- Đã format ext4.

```bash
root@linuxfilesystem:~# lvs
  LV      VG      Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_test vg_test -wi-a----- 84.00m
```

```bash
root@linuxfilesystem:~# lvreduce --resizefs -L -4M vg_test/lv_test
fsck from util-linux 2.37.2
/dev/mapper/vg_test-lv_test contains a file system with errors, check forced.
/dev/mapper/vg_test-lv_test: 11/18944 files (0.0% non-contiguous), 5308/21504 blocks
resize2fs 1.46.5 (30-Dec-2021)
Resizing the filesystem on /dev/mapper/vg_test-lv_test to 20480 (4k) blocks.
The filesystem on /dev/mapper/vg_test-lv_test is now 20480 (4k) blocks long.

  Size of logical volume vg_test/lv_test changed from 84.00 MiB (21 extents) to 80.00 MiB (20 extents).
  Logical volume vg_test/lv_test successfully resized.
```

- Trường hợp format với xfs

```bash
root@linuxfilesystem:~# lvreduce --resizefs -L -4M vg_test/lv_test
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
        - scan filesystem freespace and inode maps...
        - found root inode chunk
Phase 3 - for each AG...
        - scan (but don't clear) agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - agno = 2
        - agno = 3
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 1
        - agno = 2
        - agno = 3
No modify flag set, skipping phase 5
Phase 6 - check inode connectivity...
        - traversing filesystem ...
        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify link counts...
No modify flag set, skipping filesystem flush and exiting.
fsadm: Xfs filesystem shrinking is unsupported.
  /sbin/fsadm failed: 1
  Filesystem resize failed.     <---failed
```

- Đối với xfs thì fail bởi vì nó không hỗ  trợ giảm, nếu muốn thì phải làm thủ công backup, umount, xoá, tạo mới xfs sau đó khôi phục.

⇒ Việc giảm kích thước có thành công hay không sau khi đã format còn phụ thuộc vào loại filesystem.


**Trường hợp 3:** giảm kích thước của lv khi format và  mount 

- Đã format lv_test filesystem ext4 và mount vào /mnt/mount_point, ban đầu có size là 80M

- Giảm đi 20M

```bash
root@linuxfilesystem:~# lvreduce --resizefs -L -20M vg_test/lv_test
Do you want to unmount "/mnt/mount_point" ? [Y|n] y   ⬅️ yêu cầu umount
fsck from util-linux 2.37.2
/dev/mapper/vg_test-lv_test: 11/20480 files (9.1% non-contiguous), 2323/20480 blocks
resize2fs 1.46.5 (30-Dec-2021)
Resizing the filesystem on /dev/mapper/vg_test-lv_test to 15360 (4k) blocks.
The filesystem on /dev/mapper/vg_test-lv_test is now 15360 (4k) blocks long.

  Size of logical volume vg_test/lv_test changed from 80.00 MiB (20 extents) to 60.00 MiB (15 extents).
  Logical volume vg_test/lv_test successfully resized.
```
- Trường hợp này sẽ yêu cầu umount.
- Sau khi giảm kích thước xong:

```
root@linuxfilesystem:~# lvs
  LV      VG      Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_test vg_test -wi-ao---- 60.00m
```

Có thể dùng lệnh này để xem đang ở chế độ linear hay striped

```
lvs -o lv_name,vg_name,seg_type 
```

-	Có thể cân bằng dữ liệu trên nhiều ổ vật lý được không
-	Tại sao khi tạo lv mặc dù chưa có dữ liệu nào mà khi chạy pvs mà nó vẫn bị chiếm dung lượng

```
dd if=/dev/zero of=/mnt/mount_point/file40M bs=1M count=40
```
