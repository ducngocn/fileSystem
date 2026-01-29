LVM striped = raid0  
LVM mirrored = raid 1    
LVM raid = raid 5,6  

### Lý thuyết Striped LVM

Khi ghi dữ liệu vào một logical volume của LVM, filesystem sẽ phân bố dữ liệu đó trên các physical volume bên dưới. Có thể kiểm soát cách dữ liệu được ghi lên các physical volume bằng cách tạo một striped logical volume. Với các thao tác đọc/ghi tuần tự lớn, điều này có thể cải thiện hiệu suất I/O của dữ liệu.

Striping tăng hiệu suất bằng cách ghi dữ liệu lên một số lượng physical volume xác định theo cách vòng tròn. Khi dùng striping, các thao tác I/O có thể được thực hiện song song. Trong một số trường hợp, điều này có thể mang lại tăng hiệu suất gần như tuyến tính cho mỗi physical volume bổ sung vào stripe.

Minh họa sau đây cho thấy dữ liệu được phân chia thành stripe trên ba physical volume:

- Stripe dữ liệu đầu tiên được ghi lên PV1

- Stripe dữ liệu thứ hai được ghi lên PV2

- Stripe dữ liệu thứ ba được ghi lên PV3

- Stripe dữ liệu thứ tư được ghi trở lại PV1

![alt text](image.png)


### Thao tác 

- Tạo LV kiểu striped
```
lvcreate -i 3 -I 4 -L 2G -n striped_logical_volume volgroup01
```

```
-i: là số phân vùng tham gia

-I  là kích tước của mỗi stripe (đơn vị Kb)

-L kích thước của LV đó 

-n là tên của LV

Lưu ý: 
-Trường -I không được lớn hơn PE size.
-Thông thường trường PE size đơn vị là MB, trường -I đơn vị là KB
```
> LV có kiểu là striped nếu trong câu lệnh tạo có option -i nếu không thì sẽ là linear

- **Stripe**: một phần dữ liệu nhỏ được tách ra từ toàn bộ luồng dữ liệu để phân phối lên nhiều ổ đĩa. Nếu mặc định là 64KiB và có thể chỉnh kích thước thủ công thông qua option -I.

- Không thể đổi stripesize sau khi đã tạo LV

- Khi tạo striped LV, nó sẽ lấy đều không gian của các phân vùng vật lý nó đã khai báo trong lệnh

**ví dụ:** 

- Có 5 phân vùng vật lý trong vg_test với kích thước là 500m

- Sau đó tạo một lv_test_1 với kích thước 1G với 4 phân vùng.

```
root@linuxfilesystem:~# lvcreate -i 4 -I 4 -L 1G -n lv_test_1 vg_test
  Logical volume "lv_test_1" created.
```

```
root@linuxfilesystem:~# pvs
  PV         VG      Fmt  Attr PSize   PFree
  /dev/sda3  vg_test lvm2 a--  496.00m 240.00m
  /dev/sda4  vg_test lvm2 a--  496.00m 240.00m
  /dev/sda5  vg_test lvm2 a--  496.00m 240.00m
  /dev/sda6  vg_test lvm2 a--  496.00m 240.00m
  /dev/sda7  vg_test lvm2 a--  496.00m 496.00m
```

- Lúc này nó sẽ chia 1G thành các PE và lấy tương ứng kích thước PE tương ứng đó với 4 phân vùng lần lượt từ sd3 tới sd6.

- LV sẽ lấy những phân vùng mà có đủ không gian, không nhất thiết là từ sd3 nếu sd3 không đủ khả năng.

- Trong trường hợp nếu tạo thêm các LV mới thì nó vẫn sẽ lấy lần lượt  phân vùng có khả năng cung cấp.

Ví dụ: 

- Tạo thêm một lv_test_2 với kích thước 500M với 3 phân vùng vật lý

```
root@linuxfilesystem:~# lvcreate -i 3 -I 4 -L 500M -n lv_test_2 vg_test
  Rounding size 500.00 MiB (125 extents) up to stripe boundary size 504.00 MiB (126 extents).
  Logical volume "lv_test_2" created.
```

- Lúc này thì nó thấy 3 phân vùng đầu vẫn còn khả năng cung cấp nên sẽ lấy đều 3 phân vùng từ phân vùng đầu là sda3

```
root@linuxfilesystem:~# pvs
  PV         VG      Fmt  Attr PSize   PFree
  /dev/sda3  vg_test lvm2 a--  496.00m  72.00m
  /dev/sda4  vg_test lvm2 a--  496.00m  72.00m
  /dev/sda5  vg_test lvm2 a--  496.00m  72.00m
  /dev/sda6  vg_test lvm2 a--  496.00m 240.00m
  /dev/sda7  vg_test lvm2 a--  496.00m 496.00m
```
- Kiểm tra loại PV là striped bằng command 

```
lvs -o +segtype
```
```
root@linuxfilesystem:~# lvs -o +segtype
  LV        VG      Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert Type
  lv_test   vg_test -wi-ao----  <1.01g                                                     striped
  lv_test_2 vg_test -wi-a----- 500.00m                                                     striped
  lv_test_3 vg_test -wi-a----- 240.00m                                                     striped
  lv_test_4 vg_test -wi-a----- 228.00m                                                     striped

```
  
- Hiển thị thông tin dung lượng của file system

```
hf -h
```

- PE size = 4, 8,... MB

### Cách LVM cấp không gian từ các PV thuộc VG cho striped LV


![alt text](image-1.png)

Ví dụ: tạo LV kích thước 225M dựa vào 3 PV, PE để mặc định là 4MB

```
root@linuxfilesystem:~# lvcreate -i 3 -L 225M -n lv_test_4 vg_test
```

- LVM sẽ tạo LV dựa trên PE (đơn vị tối thiểu mà phân vùng quản lí), nó sẽ quy đổi 225MB thành số lượng PE trước bằng cách lấy 225 chia 4 ~ 56,25 làm tròn lên 57 PE.

- Sau đó, LVM sẽ tính toán số lượng PE cần lấy trong mỗi PV, nó sẽ dựa vào số lượng PV để tạo LV (dựa vào option -i ở đây là 3). Lấy 57 chia 3 = 19 tức là nó cần lấy 19 PE từ mỗi PV, trong trường hợp chia ra số thập phân vd 19,2 thì làm tròn lên là 20.

- Từ đây sẽ lấy 19PE tương ứng từ các PV, rồi đổi lại thành 19 nhân 4 bằng 76MB, tức là LV sẽ lấy 76MB từ mỗi PV.


Ưu điểm:

- Tăng tốc độ I/O vì dữ liệu được ghi đồng thời trên nhiều PV.

- Khi đọc/ghi, LVM có thể truy xuất nhiều PV cùng lúc

Nhược điểm:

- Không có dự phòng, nếu một ổ hỏng thì mất dữ liệu trong LV.


>Nếu chỉ tạo LV từ 1 PV thì nó là linear

- Lệnh xem phân vùng nào mà cấu thành nên LV

```
root@linuxfilesystem:~# lvs -a -o+devices

 LV        VG      Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert Devices
  lv_test   vg_test -wi-a-----  <2.91g                                   /dev/sda5(0),/dev/sda6(0)
  lv_test   vg_test -wi-a-----  <2.91g                                   /dev/sda8(0),/dev/sda11(0)
  lv_test   vg_test -wi-a-----  <2.91g                                   /dev/sda7(0),/dev/sda12(0)
  lv_test_2 vg_test -wi-a----- 300.00m                                   /dev/sda7(99)
```
- Lệnh xem loại LV

```
root@linuxfilesystem:~# lvs -o+segtype

LV        VG      Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert Type
  lv_test   vg_test -wi-a-----  <2.91g                                                     striped
  lv_test   vg_test -wi-a-----  <2.91g                                                     striped
  lv_test   vg_test -wi-a-----  <2.91g                                                     striped
  lv_test_2 vg_test -wi-a----- 300.00m                                                     linear

```

### Mở rộng striped LV

- Khi mở rộng LV, phải mở rộng thêm đúng số PV (số lượng PV ở đây chính là trường -i trong câu lệnh tạo Striped LV)

- Cú pháp:

```
lvextend <địa chỉ lv> -L +<Kích thước mở rộng>
```

- Trong trường hợp tạo thêm 2 PV (giả sử trường hợp này i = 2), thì thứ tự ghi dữ liệu của nó sẽ lần lượt ghi hết trên 2 PV đầu đã rồi mới chuyển sang 2 PV mới tiếp theo.

- Có thể xem dựa vào Logical Extents như sau:

```
root@linuxfilesystem:~# lvdisplay -m /dev/vg_test/lv_test

 --- Segments ---
  Logical extents 0 to 247:
    Type                striped
    Stripes             2
    Stripe size         64.00 KiB
    Stripe 0:
      Physical volume   /dev/sda3
      Physical extents  0 to 123
    Stripe 1:
      Physical volume   /dev/sda5
      Physical extents  0 to 123

  Logical extents 248 to 545:
    Type                striped
    Stripes             2
    Stripe size         64.00 KiB
    Stripe 0:
      Physical volume   /dev/sda8
      Physical extents  0 to 148
    Stripe 1:
      Physical volume   /dev/sda11
      Physical extents  0 to 148
```

- Trong trường hợp này có thể thấy được được 2 PV đầu (sda3 và sda5) nằm trong segment đầu, và 2 PV sau (sda8 và sda11) nằm trong segment sau. Segment đầu sẽ chứa LE từ 0 tới 247, segment sau chứa LE từ 248  tới 545. Điều này có nghĩa là bao giờ ghi hết trong segment đầu rồi mới đến segment sau.

- Dó đó bao giờ ghi hết  2 PV đầu (sda3 và sda5) thì mới ghi vào 2 PV sau (sda8 và sda11).

- Trong trường hợp mà tạo 3 phân vùng kích thước lần lượt là sda10 = 496,sda11 = 596, sda12 = 396.

- Nếu tạo LV với i = 2 và L = 992 (496 x 2), lúc này là dùng hết sda10 rồi, chỉ còn sda11 và sda12. Đã có 1 segment thứ nhất là sda10 và sda11.

- Sau đó mở rộng LV thêm 40M. Lúc này nó sẽ tự động tạo ra một segment thứ 2 với 2 phân vùng là sda11 và sda12.

```
root@linuxfilesystem:~# pvs
  PV         VG      Fmt  Attr PSize   PFree
  /dev/sda10 vg_test lvm2 a--  496.00m 496.00m
  /dev/sda11 vg_test lvm2 a--  596.00m 596.00m
  /dev/sda12 vg_test lvm2 a--  396.00m 396.00m
  /dev/sda3          lvm2 ---  500.00m 500.00m
  /dev/sda4  vg_test lvm2 a--  696.00m 696.00m
  /dev/sda5          lvm2 ---  500.00m 500.00m

 --- Segments ---
  Logical extents 0 to 247:
    Type                striped
    Stripes             2
    Stripe size         64.00 KiB
    Stripe 0:
      Physical volume   /dev/sda10
      Physical extents  0 to 123
    Stripe 1:
      Physical volume   /dev/sda11
      Physical extents  0 to 123

  Logical extents 248 to 257:
    Type                striped
    Stripes             2
    Stripe size          64.00 KiB
    Stripe 0:
      Physical volume   /dev/sda11
      Physical extents  124 to 128
    Stripe 1:
      Physical volume   /dev/sda12
      Physical extents  0 to 4
```

- Có thể chỉ định PV nào dùng cho LV:

```
root@linuxfilesystem:~# lvcreate -i 2 -L 40M -n lv_test_2 vg_test_2 /dev/sda10 /dev/sda11
```

- Trong trường hợp không chỉ định PV thì LV sẽ tự động chọn các PV theo quy tắc:

  - Chọn những PV đang còn đủ PE liên tục.
  - Chọn theo thứ tự tăng dần ID PV trong VG. (ID có thể hiểu là thứ tự PV được add vào vg, cái nào vào trước là 1, rồi vào tiếp là 2,...).
### Trong thực tế sử dụng như thế nào





----

1. không biết nó chọn phân vùng nào, bởi vì khi mở rộng thêm lv thì thấy nó chỉ lấy của pv đầu và pv cuối là 2 pv ban đầu đã cấp không gian cho nó

2. ``trong trường hợp mở rộng lv, ví dụ ban đầu đã dùng hết 2 pv, sau đó add thêm 2 pv nữa vào vg và up size của lv lên thì không biết dữ liệu lúc này nó được ghi như nào``

3. ``khi mà tạo thêm pv mới thì có cần 2 con có cùng kích thước không ``
- Không cần cùng kích thước

4. ``thêm một cái vào thì sao``
- Trong trường hợp này vì LV đang khởi tạo 2 PV do vậy nó sẽ tìm tới 2 PV có khả năng, nếu thêm PV vào với số lượng nhỏ hơn -i thì nó sẽ không thể chạy được.

5. Nếu trong trường hợp thêm 2 ổ khác kích thước và để -i là 2 thì nếu ổ bé hết dung lượng trước thì còn ổ lớn thì lại phí không gian trống còn lại của ổ lớn hả.

6. Nếu trong vg có 4 PV nhưng khi tạo LV thì chỉ lấy i = 2 thôi, trong trường hợp mà dùng hết 2 PV rồi thì nó có thể mở rộng lấy luôn của 2 PV còn lại không

7. trong thực tế người ta dùng lvm/striped LVM trong trường hợp nào

8. nếu 2 pv ban đầu còn một ít và add thêm 2 pv mới thì khi lv mở rộng với kích thước lớn hơn kích thước còn trống của 2pv đầu thì nó sẽ xử lí như thế nào

9.``thử trường hợp tạo lv từ 2 pv nhưng chưa lấy hết dung lượng rồi show -m sau đó mở rộng nốt dung lượng cồn lại sau đó show -m``