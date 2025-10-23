## Nội dung
- [So sánh tổng quan ext4 và xfs](#tổng-quan-ext4-và-xfs)
- [Tổng quan 4 đặc điểm chính:](#4-đặc-điểm-chính-của-xfs)
    - [Allocation group](#allocation-group)
    - [Extent-based allocation](#extent-based-allocation)
    - [Delayed allocation](#delayed-allocation)
    - [Journaling](#journaling)

## Tổng quan ext4 và xfs

- **Ext4** phù hợp với điều kiện limited ram hơn so với xfs vì xfs có nhiều overhead/tác vụ hơn để xử lí nên tiêu tốn ram hơn  

- **Ext4** cũng phù hợp để xử lí file nhỏ hơn vì nó cũng có ít overhead hơn so với xfs dẫn đến xử lí nhanh hơn.

Overhead = các công việc “bổ sung” mà hệ thống phải làm ngoài công việc chính, để đảm bảo mọi thứ hoạt động đúng.

VD: Máy tính: ghi file 1KB lên disk:

Công việc chính = ghi 1KB dữ liệu.

Overhead = cập nhật inode, bitmap, journaling metadata, tra cứu block → không phải dữ liệu, nhưng bắt buộc phải làm.

- **XFS** được tối ưu cho khối lượng dữ liệu lớn, tốc độ truy xuất cao, và tác vụ I/O song song. Nhờ khả năng xử lý nhiều luồng đọc/ghi cùng lúc và hỗ trợ mở rộng quy mô dễ dàng, XFS thường được sử dụng trong các máy chủ lưu trữ lớn, môi trường doanh nghiệp, hoặc các hệ thống có file dung lượng rất lớn.

- **XFS** không mạnh trong việc xử lý nhiều file nhỏ hoặc các tác vụ yêu cầu thao tác nhanh trên số lượng file nhỏ vì chúng xử lí dữ liệu phức tạp với nhiều overhead hơn.

## 4 đặc điểm chính của XFS

| Feature | Mô tả ngắn | Lý do cần hiểu |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| **1. Allocation Group (AG)** | XFS chia filesystem thành nhiều vùng độc lập, mỗi vùng quản lý inode + free space riêng → cho phép nhiều luồng đọc/ghi song song. | Là **lý do chính khiến XFS scale tốt** và hiệu năng cao trên hệ thống nhiều CPU/disk. |
| **2. Extent-based allocation** | Thay vì lưu từng block, XFS dùng extent (chuỗi block liền nhau) → ít phân mảnh, nhanh hơn khi đọc file lớn. | Hiểu rõ điều này giúp bạn **giải thích hiệu năng và chống phân mảnh của XFS**. |
| **3. Delayed allocation** | XFS trì hoãn việc cấp phát block đến khi ghi dữ liệu thật → dễ gom block liền nhau, tăng hiệu năng ghi. | Cực kỳ quan trọng khi phân tích hiệu năng I/O. |
| **4. Journaling (Log)** | Chỉ journal metadata (không phải data), giúp phục hồi nhanh sau crash. . | Là nền tảng cho **tính ổn định và phục hồi nhanh của XFS**. |



## Allocation Group

- Sau khi một phân vùng được format với XFS, bên trong phân vùng đó, XFS sẽ tự động chia nhỏ không gian thành các vùng logic có kích thước bằng nhau gọi là Allocation Group (AG).

- Mỗi AG hoạt động gần như một đơn vị quản lý độc lập, có bảng inode, bảng không gian trống (free space) và metadata riêng, giúp XFS có thể xử lý song song nhiều thao tác đọc/ghi trên cùng một filesystem.

- Nhờ cơ chế này, nhiều tiến trình hoặc luồng có thể truy cập đồng thời mà không gây ***tranh chấp tài nguyên***, từ đó tăng ***khả năng mở rộng*** (scalability) và ***hiệu năng I/O song song***.

***Tranh chấp tài nguyên có thể được hiểu như sau:***
```
Giả sử có hai tiến trình cùng ghi file trên một filesystem ext4.

Tiến trình A ghi file1.

Tiến trình B ghi file2.

Cả hai đều phải xin quyền truy cập vào cùng một bảng free block và bảng inode chung (vì ext4 có một vùng quản lý tập trung).
Khi tiến trình A đang cập nhật, hệ thống sẽ lock (khóa) vùng đó lại tạm thời để đảm bảo không bị ghi chồng.
Tiến trình B phải chờ A xong mới được truy cập.

➡️ Đó chính là tranh chấp tài nguyên — nhiều tiến trình muốn truy cập cùng một cấu trúc quản lý, nên phải chờ nhau, làm giảm tốc độ và khả năng mở rộng.
```

***Tăng khả năng mở rộng (scalability) và hiệu năng I/O song song:***
```
Khi số lượng tiến trình truy cập filesystem tăng lên, thay vì phải xử lý tuần tự từng tiến trình như ext4 (gây giảm tốc độ), XFS có thể xử lý nhiều tiến trình cùng lúc nhờ cơ chế Allocation Group (AG) hoạt động độc lập.

Hiệu năng I/O song song nghĩa là hệ thống có thể thực hiện nhiều thao tác đọc/ghi đồng thời mà không bị chậm lại, vì mỗi AG có thể quản lý và xử lý I/O riêng biệt, giúp tận dụng tối đa tài nguyên CPU và ổ đĩa.
```

- Các file và thư mục trong XFS có thể trải dài qua nhiều AG khác nhau, nghĩa là dữ liệu của một file lớn có thể được phân bổ trên nhiều vùng để tận dụng khả năng song song hóa.

**VD:**

- File 10GB

- Filesystem XFS với 4 AG (AG0 → AG3)

- 4 CPU core

XFS có thể chia file như sau:

| Phần dữ liệu | Nằm ở AG | CPU xử lý |
| ------------ | -------- | --------- |
| 0–2.5GB      | AG0      | CPU 0     |
| 2.5–5GB      | AG1      | CPU 1     |
| 5–7.5GB      | AG2      | CPU 2     |
| 7.5–10GB     | AG3      | CPU 3     |

→ Cả 4 CPU cùng đọc/ghi 4 phần khác nhau đồng thời, không chờ nhau.  
→ Tổng thời gian xử lý giảm mạnh.

**Đọc dữ liệu (Read) song song:**

1. File lớn được truy vấn, XFS xác định các phần nằm trên từng AG.

2. Mỗi AG trả dữ liệu của phần tương ứng đồng thời, sử dụng nhiều luồng đọc.

3. Dữ liệu được load vào RAM, sau đó được tổng hợp lại theo thứ tự logic của file.

4. Ứng dụng nhận được file hoàn chỉnh, mặc dù vật lý dữ liệu nằm rải rác trên nhiều AG.

**Ghi dữ liệu (Write) song song:**

1. File lớn được chia thành nhiều phần (extent) phân bổ trên các AG khác nhau.

2. Mỗi AG nhận phần dữ liệu tương ứng và ghi độc lập, đồng thời trên nhiều CPU/core.

3. Trong RAM, XFS theo dõi các phần này riêng biệt.

4. Sau khi tất cả AG ghi xong, dữ liệu trên disk đã được lưu liên tục về mặt logic, file hoàn chỉnh được thể hiện trong filesystem.

❓ **Khi nào thì nó ghi song song:**

(chat gpt) khi kích thước file lớn hơn agsize-là số block của một vùng logic, hoặc khi trên các vùng không còn đủ extent chứ nguyên một file.

❓(chatgpt) Nếu trong trường hợp có phân vùng chia ra thành 4 ag chẳng hạn 1,2,3,4 nếu file A này lớn hơn kích thước trống của extent ag1 nhưng lại có thể nằm trong ag2 thì nó sẽ ưu tiên theo extent liên tục tức là ag2.

**Cách xem số lượng AG sau khi format**

```
xfs_info /mount point
```

```
ngocduc@linux:~$ xfs_info /mnt/par3
meta-data=/dev/sda5              isize=512    agcount=4, agsize=25600 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=102400, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

- agcount chính là số lượng ag trong phân vùng đó.

## Extent-based allocation

**Extent-based allocation** là cơ chế quản lý không gian lưu trữ trong filesystem bằng các vùng liên tục gọi là “extent”, thay vì quản lý từng block riêng lẻ. extent là các block liên tiếp nhau.

- Trong XFS, các block của file được quản lý thông qua các extent, các extent sẽ được cấp sau khi người dùng ghi file.

- Nhờ đó, thay vì phải lưu danh sách hàng nghìn block riêng lẻ, XFS chỉ cần vài extent để mô tả toàn bộ nội dung của file.

=> Giảm phân mảnh file và tăng tốc độ truy xuất dữ liệu do dữ liệu được lưu trên các block liền kề.

- Trong feature này xfs còn sử dụng 2 cây B+:
    - Cây 1 - **cây theo độ dài**: Lưu các extent theo độ dài vùng trống tức là các block liên tục chưa được cấp cho file. Dùng để tìm vùng trống đủ lớn để ghi file mới.
    - Cây 2 - **cây theo vị trí bắt đầu**: Lưu các extent theo vị trí bắt đầu của block đầu tiên trên các extent đó. Dùng để tìm vùng trống gần file hiện tại, giảm phân mảnh khi mở rộng file.


**Khi ghi file:**

- **Tra cây theo độ dài** để tìm vùng trống tạo extent vừa đủ (≥ kích thước file, nhưng nhỏ nhất có thể) → tối ưu hóa không gian.

- Chọn extent phù hợp, ghi dữ liệu và cập nhật cả hai cây (không lưu phần extent đã cấp nữa và không lưu địa chỉ bắt đầu của extent đã cáp).

**Ví dụ:**
```
Free space B+ tree trước:
Length tree: [50KB]
Start tree:  [start=1000, length=50KB]

Cấp 12KB cho file:
- Xóa 50KB khỏi cả 2 cây
- extent còn lại 38KB → start = 1003 (block), length = 38KB
```

- Trong trường hợp mà không còn block liên tục đủ lớn để cấp cho file ví dụ file 10KB nhưng không còn extent nào ≥ 10KB, thì xfs sẽ cấp vùng trống lớn nhất hiện có, cập nhật B+ tree và tìm tiếp vùng trống cho phần dữ liệu còn lại.

**Khi mở rộng file hiện có:**

- **Tra cây theo vị trí bắt đầu** để tìm extent liền kề file hiện tại.

- Nếu vùng trống này đủ lớn → chọn luôn.

- Nếu không đủ → sang cây theo độ dài để tìm vùng trống phù hợp.

**Ví dụ:**  

---
- XFS xem vị trí ngay sau file A, chẳng hạn là block 11. Nó tra cây (theo vị trí bắt đầu) để kiểm tra xem có extent trống liền kề block 11 không. Nếu có extent liền kề, nó kiểm tra kích thước của extent:   

    - Nếu extent đủ lớn (≥ số lượng blocks cần mở rộng) → chọn luôn extent này để mở rộng file.  
    - Nếu extent nhỏ hơn yêu cầu → không đủ → chuyển sang cây theo độ dài.
    - Trong trường hợp có nhiều extent đủ lớn, XFS chọn extent nhỏ nhất ≥ số block cần mở rộng → tối ưu hóa sử dụng không gian.

---

- Sau khi ghi xong → cập nhật cả hai cây để giữ dữ liệu đồng bộ.

**Khi xóa file hoặc giải phóng block:**

- Gộp (merge) các extent liền kề nhờ tra cây theo start block.

- Thêm lại extent mới vào cả hai cây để duy trì thông tin free space.

**So sánh với extent EXT4**

- Nhìn chung về khái niệm extent đều là tập hợp của các block liện nhau.

-Cơ chế cấp phát extent khác nhau:

**ext4:** 

- Cơ chế cấp phát:  
EXT4 quản lý vùng trống bằng bitmap.
Khi cần cấp phát, hệ thống duyệt bitmap tuần tự, gặp đủ số block trống liên tục thì cấp phát ngay.
Cơ chế này đơn giản, ít overhead, hiệu năng tốt với file vừa và nhỏ. Bởi vì nếu so sánh với xfs, thì ext4 chỉ cần duyệt lần lượt và sau đó là cập nhật inode và bitmap tương ứng, trong khi đó xfs sẽ cập nhật 2 cây B+, metadata của ag, những cập nhật này sẽ tốn ram và cpu hơn(overhead).

- Hạn chế với file lớn:  
Khi filesystem đã bị chia nhỏ, các vùng trống thường rải rác.
Nếu cần ghi một file lớn, EXT4 phải duyệt tuần tự qua nhiều block group để tìm vùng trống đủ dài.
Trong trường hợp không có vùng nào đủ lớn, nó sẽ duyệt gần như hết ổ đĩa thì mới nhận ra không đủ vùng trống trước khi quay lại ghép nhiều extent nhỏ để đủ dung lượng.  
→ Tác động: tăng thời gian cấp phát, giảm hiệu năng, và dễ gây phân mảnh file.

**xfs:** 

Tra B+ tree theo size để chọn extent tối ưu vừa đủ, liên tục, giảm phân mảnh. Cập nhật cả B+ tree theo block  sau khi cấp phát. Tốt cho file lớn & I/O nhiều luồng.

- XFS quản lý vùng trống bằng hai cây B+ tree thay vì bitmap như EXT4.
Khi cấp phát, XFS có thể:

    - Tra nhanh theo kích thước để tìm một vùng trống liền đủ lớn.

    - Trong trường hợp mà nó không có extent đủ lớn thì sẽ nhanh chóng phát hiện ra, lúc này xfs sẽ chia file đó vào các ag khác nhau để xử lí.

    - Tuy nhiên, với file nhỏ, do XFS phải thực hiện nhiều thao tác phụ (cập nhật 2 B+ tree, metadata AG, tính toán extent...) nên có nhiều overhead hơn → xử lý chậm hơn EXT4.


## Delayed allocation 

**Khái niệm**


- Delayed Allocation là cơ chế trì hoãn việc cấp phát block trong quá trình ghi file.

- Thay vì cấp block ngay khi ứng dụng đẩy data vào cache, xfs chờ đến khi dữ liệu thực sự được ghi xuống đĩa (flush) mới cấp phát block vật lý.


**Nguyên nhân ra đời**


Các filesystem cũ như ext2 / ext3 cấp phát block ngay khi ghi vào cache, dù dữ liệu chưa được flush xuống ổ đĩa.

Do đó, nếu một file được ghi nhiều lần liên tiếp, hệ thống có thể cấp phát nhiều block rời rạc → phân mảnh file.


>Phân mảnh file tức là dữ liệu trong cùng một file nhưng lại nằm dải rác ở nhiều vị trí không liền kề. Điều này gây ra:  
a. giảm hiệu năng truy suất đĩa do khi file nằm rải rác thì dầu đọc phải nhảy qua nhiều vị trí.  
b. gây độ trễ khi truy cập dữ liệu.

- ***Ví dụ:***  echo 3 lần liên tục vào cùng một file  
 echo a >> BangChuCai  
 echo b >> BangChuCai  
 echo c >> BangChuCai  

- Đối với **ext2** chưa hỗ trợ delayed allocation   
Lúc này sau mỗi lần echo dữ liệu mới được đẩy vào cache thôi chưa được đẩy vào disk, nhưng nó đã yêu cầu cấp block rồi. Lúc này sau 3 lần thì nó yêu cầu 3 block khác nhau và có thể không liên tục dẫn đến cùng một file nhưng bị phân mảnh.

- Đối với **xfs**  
filesystem này sẽ đợi cả 3 lần echo xong mới xin cấp phát block, nó sẽ tính toán xem cần bao nhiêu và cấp phát bấy nhiêu block liên tục.

**Nguyên lí hoạt động** 

```
Ứng dụng gọi write() → dữ liệu ghi vào page cache (RAM), chưa cấp block.

Hệ thống theo dõi kích thước dữ liệu đang chờ flush.

Khi flush xảy ra (do cache đầy hoặc hết thời gian chờ):

Filesystem mới tính toán kích thước thực cần ghi.

Cấp phát vùng block vật lý lớn, liền kề trên đĩa.

Ghi dữ liệu một lần xuống vùng đó.

➡️ Kết quả:

Dữ liệu của file nằm liên tục trên đĩa → giảm phân mảnh.

Giảm số lần ghi nhỏ lẻ → tăng hiệu năng ghi đĩa.
```

>Nếu không còn đủ không gian trống liên tục để cấp phát một extent duy nhất cho toàn bộ dữ liệu,
XFS sẽ chia file thành nhiều extent nhỏ hơn, trong đó mỗi extent vẫn là một dải block liên tiếp trên đĩa.

## Journaling 

**Khái niệm: journaling**


- Journaling File System là cơ chế trong hệ thống quản lý tệp, cho phép ghi lại nhật ký (journal) của các thay đổi metadata (và đôi khi cả dữ liệu) trước khi áp dụng trực tiếp lên vùng dữ liệu chính.

- Cơ chế này giúp:

    - Đảm bảo tính nhất quán (consistency) và đồng bộ trạng thái của hệ thống tệp,

    - Giảm thiểu nguy cơ hỏng cấu trúc filesystem khi mất điện, treo máy hoặc lỗi ghi,

    - Cho phép khôi phục nhanh về trạng thái ổn định gần nhất bằng cách đọc lại hoặc bỏ qua các thay đổi đã lưu trong journal.

    - Mục tiêu của journaling không phải là ngăn mất dữ liệu người dùng hoàn toàn, mà là đảm bảo hệ thống tệp luôn ở trạng thái nhất quán (consistent), tránh hư hại cấu trúc khi có sự cố.


**Lý do ra đời**

- Do tính không nhất quán có thể xảy ra trong khi ghi hoặc thay đổi dữ liệu mà gặp tình trạng mất điện, crash giữa chừng.

**Ví dụ:**
Khi xóa một file, cần 3 bước:

1. Xóa entry của file khỏi thư mục.

2. Trả inode của file về danh sách inode trống.

3. Trả các block dữ liệu của file về danh sách block trống.

Nếu crash xảy ra:

- Sau bước 1 nhưng trước bước 2 → file bị mất entry trong thư mục, nhưng inode vẫn “tồn tại” → inode mồ côi, gây tốn dung lượng (file không tồn tại nữa nhưng inode vẫn còn và vẫn trỏ tới block trên đĩa dẫn tới đĩa vẫn trong tình trạng chứa dữ liệu không thể tái sử dụng)

- Sau bước 2 nhưng trước bước 3 → các block dữ liệu chưa được thu hồi → giảm dung lượng khả dụng của ổ đĩa.

Nếu đảo thứ tự:

- Làm bước 3 trước bước 1, crash giữa chừng → block của file có thể bị cấp cho file khác → 2 file chia sẻ cùng dữ liệu → mất toàn vẹn dữ liệu.

- Làm bước 2 trước bước 1 → file sẽ không truy cập được dù vẫn “tồn tại” → lỗi logic.

→ **journaling** ra đời để đảm bảo tính nhất quán của filesystem trong trường hợp sự cố xảy ra

**Nguyên lý hoạt động:**

```
Journaling không lưu trữ toàn bộ file, mà chỉ ghi lại những thay đổi sắp diễn ra. Nguyên tắc cơ bản:

Khi người dùng thực hiện thay đổi (ví dụ thêm, sửa hoặc xóa file), thông tin thay đổi được ghi trước vào vùng journal.
Nếu quá trình hoàn tất thành công, hệ thống đánh dấu “commit” và thay đổi chính thức áp dụng.
Nếu máy tính tắt đột ngột, hệ thống có thể đọc lại journal để khôi phục trạng thái gần nhất, tránh hỏng file.
```

❓**Khi nào thì xfs quyết định ghi vào journal**

- việc xfs quyết định đẩy dữ liệu từ cache vào journal phụ thuộc vào nhiều tham số như dirty_ratio, dirty_background_ratio, dirty_writeback_centisecs, dirty_expire_centisecs, vm.dirtytime_expire_seconds,.. Nhưng chung quy lại sẽ quy về 2 điều dưới đây:  
- **Khi đủ lớn:**

-Khi ghi file liên tục:
  Gom các thay đổi (metadata + data) vào RAM.
  Khi lượng thay đổi đạt đến ngưỡng hoặc không còn chỗ trong transaction hiện tại
➜ đẩy toàn bộ transaction đó vào journal và bắt đầu transaction mới.


- **Khi hết thời gian chờ:**  

Linux có bộ đếm thời gian đồng bộ (commit interval) — mặc định khoảng 5 giây.
Dù transaction chưa đầy, nếu hết 5 giây mà vẫn còn dữ liệu chưa ghi, xfs sẽ:
➜ ép ghi transaction hiện tại vào journal để đảm bảo không giữ dữ liệu quá lâu trong RAM.
Cơ chế này giúp tránh mất dữ liệu quá nhiều nếu mất điện: bạn chỉ mất tối đa khoảng 5 giây dữ liệu mới nhất.


**Transaction:** Là một nhóm thay đổi mà filesystem muốn thực hiện: Ghi nội dung file (data block),tạo inode cho file,


**Cơ chế  mô tả như sau:**


1. Ứng dụng ghi dữ liệu:

Dữ liệu (data) được ghi vào cache trong RAM.

Metadata (inode, block bitmap…) được cập nhật trong cache.

2. Flush data trước:

Kernel ép data trong page cache phải được ghi xuống block dữ liệu thật trên disk.

Đảm bảo block đã chứa data hợp lệ trước khi metadata trỏ đến nó.

3. Ghi metadata vào journal:

Các thay đổi metadata trong cache được ghi xuống journal area (log).

Khi toàn bộ metadata log đã được ghi, transaction được coi là commit.

4. Ghi metadata thật ra disk chính:

Sau commit, metadata trong cache (inode, bảng block…) mới được flush ra disk chính (filesystem area).

Nếu chưa kịp flush mà crash → khi reboot, kernel sẽ dùng journal để replay và ghi lại metadata.

> journal sẽ nằm trong cùng phân vùng dữ liệu chính của filesystem → gọi là internal log, hoặc trên thiết bị riêng biệt (ổ SSD riêng hoặc một phân vùng khác)  
> Trong khi đó journal trong ext4 sẽ nằm chung trong  cùng phân vùng chứ không tách riêng.

### Tình huống xảy ra sau khi đã có order journaling


Ghi "abc" vào file test.txt (file ban đầu trống).

Trình tự xảy ra:

1. User write() → dữ liệu “abc” được ghi vào page cache (RAM).
→ Chưa có gì trên đĩa, chỉ trong RAM.

2. Kernel (xfs) sẽ flush dữ liệu xuống đĩa thật
→ Ghi "abc" vào data block thật (vùng lưu dữ liệu file).
→ Nhưng inode (metadata) vẫn còn trong bộ nhớ, chưa cập nhật xuống đĩa.

3. Chuẩn bị transaction metadata (inode, block mapping, timestamp, v.v.)
→ Ghi metadata đó vào journal.

4. Ghi commit để đánh dấu toàn bộ transaction đã được đẩy vào journal.

⚡ Giờ nếu tắt máy (crash) ở các giai đoạn khác nhau:  
**Trường hợp 1:** Crash trước khi commit

```
Dữ liệu "abc" đã được ghi xuống đĩa thật 

Nhưng metadata (inode) chưa được commit vào journal 

Khi hệ thống khởi động lại:

Journal chưa có transaction hoàn chỉnh, chưa commit → toàn bộ metadata chưa hoàn chỉnh này trong journal sẽ bị bỏ qua.

Inode cũ vẫn nói file size = 0, không trỏ tới block chứa "abc".

Block "abc" đó tuy tồn tại trên đĩa, nhưng không có metadata nào trỏ tới nó → filesystem coi như không thuộc về file nào.

=> File trông “trống”, vì theo metadata, nó vẫn có kích thước 0 và ban đầu chưa trỏ vào block nào cả.
Nhưng dữ liệu "abc" vẫn nằm đâu đó trên đĩa — chỉ là không được “tham chiếu”.

=>File không mất tính nhất quán (inode khớp với dữ liệu mà nó biết), nhưng dữ liệu mới chưa được chính thức ghi nhận.
```
**Trường hợp 2:** Crash sau khi commit
```
Journal đã có metadata mới (size, mapping block,...) và đã commit thành công.

Khi khôi phục, journal sẽ replay metadata.

Inode trỏ đúng tới block có "abc".

File xuất hiện đầy đủ "abc".

File chính xác và an toàn.
```

 