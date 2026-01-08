## Nội Dung
- [Khái niệm](#khái-niệm)
- [Lý do ra đời](#lý-do-ra-đời)
- [Các thành phần chính của LVM](#các-thành-phần-chính-của-lvm)

- [4 Các đặc điểm chính:](#6-đặc-điểm-của-lvm)

    - [Flexible capacity](#flexible-capacity)

    - [Resize volume](#resize-volume)

    - [Snapshot](#snapshot-logical-volume)

    - [Move](#di-chuyển-dữ-liệu-từ-pv-này-sang-pv-khác-không-bị-downtime)

### LVM Logical volume manager 

#### Khái niệm

- Là một công cụ quản lý ổ đĩa logic cho phép tạo ra một lớp trừu tượng giữa phần cứng (ổ đĩa, phân vùng) và hệ thống tệp.  

- Thay vì hệ điều hành làm việc trực tiếp với ổ đĩa/phân vùng vật lý, nó làm việc với logical volume (LV).

- Nhờ có lớp trung gian này, LVM mang lại tính linh hoạt: có thể mở rộng, thu nhỏ, di chuyển hoặc gộp dung lượng mà không cần định dạng lại phân vùng hay dừng ứng dụng.

#### Lý do ra đời

- LVM (Logical Volume Manager) được sinh ra để khắc phục hạn chế của phân vùng truyền thống trên Linux.

- Trước khi có LVM, Linux sử dụng các partition cơ bản trên ổ cứng. Những hạn chế chính:

1. Cứng nhắc về dung lượng

    - Khi tạo partition, phải dự đoán dung lượng cần dùng.

    - Nếu hết dung lượng, không thể mở rộng partition hiện có mà không xóa hoặc tạo lại.

2. Khó mở rộng partition

   - Muốn tăng dung lượng partition thường phải: sao lưu dữ liệu → xóa partition → tạo partition mới → khôi phục dữ liệu.

    - Quá trình này rất mất thời gian và có rủi ro mất dữ liệu nếu thao tác sai.

3. Khó quản lý nhiều ổ cứng

    - Mỗi ổ cứng phải quản lý riêng.

    - Không thể gộp nhiều ổ cứng thành một không gian lưu trữ linh hoạt.

=> LVM ra đời để giải quyết tất cả các vấn đề này:

- Cho phép mở rộng logical volume mà không ảnh hưởng dữ liệu.

- Gộp nhiều ổ cứng thành Volume Group duy nhất.

- Quản lý linh hoạt dung lượng mà không cần dừng dịch vụ hay mất dữ liệu.


### Các thành phần chính:

- Physical Volume (PV): là phân vùng hoặc toàn bộ ổ đĩa vật lý được đưa vào quản lý bởi LVM. Lúc này thì os sẽ không thao tác trực tiếp với phân vùng này nữa.

- Volume Group (VG): là tập hợp các PV, ổ đĩa ảo, tạo thành một bể dung lượng chung. Kích thước của VG = tổng dung lượng các PV. Từ VG, có thể phân bổ dung lượng cho các LV.

- Logical Volume (LV): là phân vùng ảo được tạo ra từ VG. OS coi LV giống như một phân vùng bình thường, nhưng có thể mở rộng, thu nhỏ hoặc di chuyển mà không cần chia lại phân vùng vật lý.

<img src="images/mô hình lvm.png" alt="Mô hình LVM" width="400"/>

### 4 Đặc điểm của LVM

| Đặc điểm chính            | Mục đích / Chức năng chính                                                           |
| -------------------- | ------------------------------------------------------------------------------------ |
| **Flexible capacity** | Gộp nhiều ổ đĩa vật lý (PV) thành Volume Group (VG) để tạo Logical Volume linh hoạt. |
| **Resize volume**    | Mở rộng hoặc thu nhỏ LV/VG để tận dụng dung lượng trống mà không ảnh hưởng dữ liệu.  |
| **Snapshot**         | Tạo bản sao tạm thời của LV tại một thời điểm, dùng để backup, rollback hoặc test.   |
| **Move**             | Di chuyển dữ liệu LV giữa các PV trong VG thay thế ổ.               |


#### Flexible capacity  

- LVM cho phép gộp nhiều thiết bị vật lý (Physical Volume – PV) lại thành một không gian lưu trữ chung (Volume Group – VG).

- Từ đó, có thể tạo Logical Volume (LV) có dung lượng tùy ý từ không gian VG này.

#### Resize volume

- Tính năng resize volume cho phép tăng (extend) dung lượng của VG và LV mà không cần xóa hoặc chia lại partition.

- Đây là một trong những điểm mạnh nổi bật của LVM so với cách chia partition truyền thống.

- Ưu điểm chính:

  - Thay đổi dung lượng LV động mà không ảnh hưởng đến dữ liệu.

  - Cho phép tận dụng dung lượng trống trong VG.

  - Có thể mở rộng xuyên qua nhiều ổ đĩa vật lý.

  - Giảm rủi ro so với việc chia lại partition.

#### Snapshot Logical volume

##### Khái niệm

- Snapshot là cơ chế tạo bản sao ảo (virtual copy) của LV tại một thời điểm xác định. Không làm gián đoạn LV gốc, tức là hệ thống vẫn chạy bình thường.

- Snapshot không phải copy toàn bộ LV ngay lập tức, chỉ ghi metadata để biết trạng thái LV tại thời điểm snaps.

- Snapshot dùng để backup tạm thời, rollback, testing, không dùng làm LV chính.

- Snapshot là một Logical Volume riêng trong cùng Volume Group (VG) với LV gốc.


##### Cơ chế Copy-On-Write (COW)


- Khi LV gốc thay đổi một vùng dữ liệu:

    - Snapshot copy vùng dữ liệu cũ trước khi bị ghi đè.

    - LV gốc tiếp tục hoạt động bình thường.

    - Nhờ đó, snapshot luôn giữ nguyên trạng thái LV tại thời điểm snapshot.

##### Snapshot tiết kiệm dung lượng

- Nếu LV gốc ít thay đổi → snapshot cần rất ít dung lượng.

- Ví dụ: nếu chỉ 3–5% LV gốc thay đổi → snapshot chỉ chiếm 3–5% dung lượng LV gốc.

- Lưu ý: snapshot không thay thế backup, chỉ là bản sao ảo để test, rollback, hoặc backup tạm thời.

##### Kích thước snapshot

- Snapshot cần dung lượng để lưu các thay đổi của LV gốc.

- Nếu tạo snapshot và ghi đè toàn bộ LV gốc → snapshot cần ít nhất bằng dung lượng LV gốc.

- Chọn kích thước snapshot dựa vào mức độ thay đổi dự kiến:

    - LV read-mostly (ít ghi) → snapshot nhỏ.

    - LV nhiều ghi → snapshot lớn.

- Nếu snapshot đầy → snapshot bị invalid, không thể theo dõi thêm thay đổi nữa. 

- Snapshot có thể resize: tăng để tránh bị đầy, giảm nếu quá lớn để giải phóng dung lượng.

##### Quyền truy cập và hoạt động

- LV gốc (origin) vẫn đọc/ghi bình thường — snapshot không ảnh hưởng đến hoạt động của LV gốc.

- Snapshot có thể là read-only (mặc định) hoặc read-write nếu chỉ định khi tạo.
→ Cho phép thử nghiệm, test, hoặc thay đổi dữ liệu mà không ảnh hưởng đến LV gốc.

- Khi một vùng dữ liệu trên snapshot bị ghi (write):  
→ LVM sẽ copy block dữ liệu gốc vào vùng riêng của snapshot (nếu chưa có)  
→ sau đó snapshot ghi đè lên bản copy đó.  
→ Từ thời điểm này, block đó không còn nhận thay đổi từ LV gốc nữa (tức là “độc lập hoàn toàn”).

#### Di chuyển dữ liệu từ PV này sang PV khác không bị downtime

- Là cơ chế khác biệt lớn giữa LVM và phân vùng truyền thống.

**Mục đích:**
- Di chuyển dữ liệu của các Logical Volume (LV) từ một Physical Volume (PV) sang một PV khác trong cùng Volume Group (VG) mà không cần ngắt dịch vụ hoặc downtime.
Việc này giúp bảo trì, thay thế, hoặc tái phân bố dữ liệu một cách an toàn và linh hoạt.

**Thường áp dụng khi:**

- Ổ đĩa cũ sắp hỏng hoặc cần thay thế.

- Muốn loại bỏ một PV khỏi VG mà không mất dữ liệu.

**Nguyên nhân ra đời của cơ chế này:**

- Trước khi có LVM hoặc cơ chế pvmove, việc thay thế ổ cứng, mở rộng, hay bảo trì hệ thống phải thực hiện thủ công và offline (backup → thay ổ → phục hồi).  
➜ Quá trình này mất thời gian, dễ lỗi và có rủi ro mất dữ liệu nếu thao tác sai.

**=> Cơ chế này ra đời để khắc phục các nhược điểm đó:**

- Cho phép di chuyển dữ liệu động giữa các PV trong cùng VG.

- Không cần downtime, không ảnh hưởng đến LV logic hoặc ứng dụng đang chạy.

- Giúp bảo trì, thay thế ổ nhanh chóng và an toàn.

**Cơ chế hoạt động**

1. Mirror tạm thời (Temporary Mirror):

- Khi pvmove bắt đầu, LVM tạo mirror tạm thời giữa:

    - PV nguồn (chứa dữ liệu gốc)

    - PV đích (nơi dữ liệu sẽ chuyển sang)

- Nếu di chuyển sang nhiều PV đích, dữ liệu sẽ được chia thành từng Physical Extent (PE) và mỗi phần sẽ tạo mirror tạm sang PV đích tương ứng.

2. Đồng bộ dữ liệu:

- LVM copy từng PE từ PV nguồn sang PV đích.

- Trong lúc này, mọi ghi/đọc vào LV vẫn được đồng bộ song song trên cả mirror → đảm bảo dữ liệu nhất quán.

3. Xóa mirror gốc:

- Khi tất cả PE đã được di chuyển thành công (100%), LVM gỡ bỏ mirror gốc.

- Dữ liệu LV giờ chỉ nằm trên PV đích, PV nguồn trống hoàn toàn.