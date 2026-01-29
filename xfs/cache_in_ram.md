**Vấn đề:** Liệu có kiểm soát được size của page cache trong ram không, trong trường hợp mà cơ chế delayed allocation của xfs lưu tạm vào page cache thì nó sẽ chiếm bao nhiêu trong ram bởi còn phải dành ram cho các tác vụ khác. 

Câu hỏi: là liệu có thay đổi size được không? vì nếu trong trường hợp page cache nó chiếm nhiều quá thì còn ram đâu để xử lí tác vụ khác

Page cache là gì? tại sao khi ram trống nó lại cấp phát cho cache? 

Cơ chế này như nào, khi nào thì cache trả ram, khi nào thì nó chiếm, có thể thay đổi kích thước nó chiếm được không?

## Page cache

Page cache là vùng RAM mà hệ điều hành dùng để lưu tạm dữ liệu được đọc/ghi từ ổ đĩa.

không biết mặc định ban đầu thì kích thước page cache là bao nhiêu

``Key:``    

"Thông thường, toàn bộ vùng RAM vật lý chưa được cấp phát trực tiếp cho ứng dụng sẽ được hệ điều hành sử dụng để làm page cache.
Vì phần bộ nhớ này sẽ rảnh rỗi nếu không dùng và có thể thu hồi dễ dàng khi ứng dụng cần thêm RAM, nên nó không gây giảm hiệu năng. Thậm chí, hệ điều hành có thể vẫn báo vùng bộ nhớ đó là “free” hoặc “available”, dù thực ra nó đang được dùng cho page cache."

So với RAM, đọc/ghi ổ cứng rất chậm, đặc biệt là truy cập ngẫu nhiên tốn nhiều thời gian tìm kiếm (seek).
Do đó, RAM càng nhiều thì hệ thống càng nhanh, vì có thể lưu nhiều dữ liệu hơn trong bộ nhớ đệm (cache).

Nên nếu bạn có 128 GB RAM, và XFS đang hoạt động mạnh, kernel có thể dùng 50–100 GB cho cache nếu không có tiến trình khác cần RAM.

➡️ Không thể “đặt cấu hình: XFS cache = 32 GB” được trực tiếp.

Cache luôn chiếm RAM, nhưng có thể nhường lại bất cứ lúc nào.

=> Bởi vậy mà không cần quá quan tâm đến vấn đề liệu page cache có chiếm quá nhiều ram không.

**Trong trường hợp mà ứng dụng cần thêm ram ví dụ:**
```
Giả sử RAM tổng: 12 GB

Ứng dụng đang dùng: 4 GB

Page cache (XFS, filesystem) đang dùng: 8 GB

Bây giờ có ứng dụng mới cần thêm 2 GB RAM.
```

- Clean pages trong cache

Các trang chưa bị sửa (clean page) trong page cache có thể bị loại bỏ ngay.

Kernel chỉ cần drop page cache đó → giải phóng RAM cho ứng dụng.

Khi ứng dụng truy cập file đó sau này → Linux sẽ load lại từ disk.

Không ảnh hưởng dữ liệu của ứng dụng, và ứng dụng không biết page cache đã bị drop.

- Dirty pages trong cache

Các trang đã bị sửa (dirty pages) → chưa ghi xuống đĩa.

Kernel phải flush dữ liệu ra đĩa trước khi reclaim.

| Giai đoạn hệ thống          | Hành vi của kernel                   |
| --------------------------- | ------------------------------------ |
| RAM dồi dào                 | Cấp phát trực tiếp từ free pages     |
| RAM giảm dưới low watermark | Gọi `kswapd` → reclaim bất đồng bộ   |
| RAM giảm dưới min watermark | Tiến trình bị block → direct reclaim |

## Ram nó sẽ cấp và thu hồi page cache theo cơ chế nào ?

Trong Linux, RAM trống sẽ được kernel tận dụng để lưu trữ dữ liệu thường xuyên được truy cập trong Page Cache.
Nhờ đó, các lần đọc/ghi sau này có thể được xử lý nhanh hơn nhiều so với truy xuất trực tiếp từ ổ đĩa.
Ngoài ra, khi ghi dữ liệu ra đĩa, các cơ chế như delayed allocation cũng lưu tạm dữ liệu trong Page Cache trước khi ghi thực sự.

Tuy nhiên, lượng RAM dùng cho Page Cache không cố định, mà được điều chỉnh tùy vào mức độ rảnh của bộ nhớ hệ thống.
Linux sử dụng các ngưỡng watermarks để đánh giá tình trạng bộ nhớ và quyết định khi nào cần thu hồi (reclaim) dung lượng Page Cache.

Các ngưỡng watermarks được xác định dựa trên số lượng trang (page) trống trong từng vùng nhớ (memory zone).
Ba mức watermarks chính gồm:

Min Watermark (mức tối thiểu)
Là số lượng page trống tối thiểu mà vùng nhớ phải duy trì để đảm bảo cấp phát không làm giảm hiệu năng hệ thống đáng kể.

Low Watermark (mức thấp)
Cao hơn một chút so với mức min, tạo vùng đệm an toàn để hệ thống có thời gian kích hoạt cơ chế reclaim trong nền.

High Watermark (mức cao)
Là mức “thoải mái”, phản ánh trạng thái bộ nhớ khỏe mạnh với đủ free pages để xử lý các đợt cấp phát đột biến.

Khi lượng free pages < low watermark, kernel sẽ bắt đầu reclaim dần Page Cache và các vùng nhớ có thể thu hồi khác.
Nếu free pages giảm xuống dưới min watermark và Page Cache đã được thu hồi gần hết, hệ thống sẽ ngừng cấp phát mới, thậm chí có thể kích hoạt OOM Killer để bảo vệ sự ổn định.

**Cú pháp xem thông tin chi tiết về cách kernel quản lý bộ nhớ vật lý (RAM), được chia theo Node và Zone.**

```
cat /proc/zoneinfo
```

- Lện này hiển thị toàn bộ thông tin chi tiết về việc kernel chia nhỏ và quản lý bộ nhớ vật lý (RAM).

- RAM trong Linux được chia làm nhiều node và zone:

    - Node = đại diện cho một vùng bộ nhớ vật lý thuộc một CPU (nếu hệ thống có nhiều CPU socket hoặc NUMA).

    - Zone = chia nhỏ bộ nhớ trong mỗi node theo mục đích sử dụng:

        - DMA – vùng rất thấp (dành cho thiết bị cũ, cần truy cập vùng <16MB)

        - DMA32 – vùng dưới 4GB, dùng cho thiết bị 32-bit

        - Normal – vùng RAM chính (nơi hầu hết tiến trình, page cache, kernel hoạt động)

        - Movable – (nếu có) vùng chứa page có thể di chuyển (chống phân mảnh)


Lệnh trên kia có nghĩa là gì?  
Hệ thống tính các watermarks thế nào ?  
Demo xem thử là có đúng là nó dựa vào watermarks để cấp phát page cache không?  
Có thể thay đổi được chúng không?  