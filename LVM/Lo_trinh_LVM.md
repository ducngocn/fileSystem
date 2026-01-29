🧭 LỘ TRÌNH 4 NGÀY THÀNH THẠO LVM (cho sinh viên sysadmin)
🎯 Mục tiêu cuối cùng:

Hiểu rõ cơ chế PV → VG → LV.

Nắm 6 tính năng chính của LVM mà thực tế hay dùng.

Làm được các thao tác tạo, mở rộng, di chuyển, backup và phục hồi dữ liệu bằng LVM.

🔹 NGÀY 1 – NỀN TẢNG & CẤU TRÚC LVM

Mục tiêu: Hiểu và thao tác cơ bản với PV, VG, LV.

🧠 Lý thuyết cần nắm:

Cấu trúc 3 tầng: PV → VG → LV.

Cách LVM tạo ra lớp trừu tượng giữa hardware và filesystem.

Ưu điểm chính: linh hoạt, gộp ổ, không cần chia lại partition.

💻 Thực hành:

Kiểm tra các ổ hiện có:
```
lsblk
fdisk -l
```

Tạo PV từ partition hoặc disk mới:
```
pvcreate /dev/sdb /dev/sdc
pvdisplay
```

Gộp PV thành VG:
```
vgcreate vgdata /dev/sdb /dev/sdc
vgdisplay
```

Tạo LV từ VG:
```
lvcreate -n lvdata -L 5G vgdata
lvdisplay
```

Định dạng & mount:

mkfs.ext4 /dev/vgdata/lvdata
mount /dev/vgdata/lvdata /mnt

✅ Kết quả ngày 1:

Hiểu mô hình LVM.

Tự tay tạo được PV, VG, LV và gắn vào hệ thống.

🔹 NGÀY 2 – MỞ RỘNG & THU NHỎ (RESIZE)

Mục tiêu: Nắm vững việc thay đổi dung lượng mà không mất dữ liệu.

🧠 Lý thuyết:

Mở rộng (extend): khi ổ sắp đầy.

Thu nhỏ (reduce): khi muốn tiết kiệm không gian.

Cần chú ý filesystem khác nhau (ext4, xfs).

💻 Thực hành:

Mở rộng LV:

lvextend -L +2G /dev/vgdata/lvdata
resize2fs /dev/vgdata/lvdata


Thu nhỏ LV (ext4):

Umount trước:

umount /mnt
e2fsck -f /dev/vgdata/lvdata
resize2fs /dev/vgdata/lvdata 4G
lvreduce -L 4G /dev/vgdata/lvdata
mount /mnt


Kiểm tra dung lượng sau thay đổi:

df -h /mnt

✅ Kết quả ngày 2:

Biết cách mở rộng và thu nhỏ logical volume an toàn.

Hiểu sự khác biệt giữa filesystem ext4 và xfs khi resize.

🔹 NGÀY 3 – MOVE, STRIPING, RAID

Mục tiêu: Di chuyển dữ liệu khi thay ổ, tăng hiệu năng và độ an toàn.

🧠 Lý thuyết:

pvmove: di chuyển dữ liệu sang ổ khác mà không downtime.

Striping: chia dữ liệu song song → tăng tốc đọc/ghi.

RAID LVM: nhân bản dữ liệu → chống lỗi đĩa.

💻 Thực hành:

Di chuyển dữ liệu giữa PV:

pvmove /dev/sdb /dev/sdd


Tạo striped LV (2 ổ):

lvcreate -i2 -I64 -L 4G -n lvstripe vgdata


Tạo RAID 1 LV:

lvcreate --type raid1 -m1 -L 2G -n lvraid vgdata

✅ Kết quả ngày 3:

Biết di chuyển dữ liệu khi thay ổ.

Biết tạo volume tăng hiệu năng và an toàn dữ liệu.

🔹 NGÀY 4 – SNAPSHOT & KIỂM TRA TỔNG HỢP

Mục tiêu: Biết tạo snapshot để sao lưu, rollback, test update.

🧠 Lý thuyết:

Snapshot là bản sao tại thời điểm T của LV.

Dùng để backup, test nâng cấp, rollback nhanh.

💻 Thực hành:

Tạo snapshot:

lvcreate -s -n lvsnap -L 1G /dev/vgdata/lvdata


Mount snapshot & test:

mount /dev/vgdata/lvsnap /mnt/snap


Rollback snapshot:

lvconvert --merge /dev/vgdata/lvsnap
reboot


Xóa snapshot khi không cần:

lvremove /dev/vgdata/lvsnap

✅ Kết quả ngày 4:

Biết tạo, dùng và phục hồi snapshot.

Tổng hợp lại toàn bộ chu trình quản lý ổ bằng LVM.

🎯 Sau 4 ngày bạn sẽ:

Làm chủ 6 tính năng chính: Flexible, Resize, Move, Striping, RAID, Snapshot.

Biết thao tác quản lý ổ linh hoạt, mở rộng, backup mà không downtime.

Đạt mức thực hành tốt LVM trong môi trường thật (lab, server ảo, hoặc production nhỏ).