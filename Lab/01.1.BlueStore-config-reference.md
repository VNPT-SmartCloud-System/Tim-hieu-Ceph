# Cầu hình BlueStore

## 1. Devices

BlueStore quản lý 1,2 hoặc 3 thiết bị lưu trữ:
- Primary
- WAL
- DB

Trường hợp đơn giản nhất, BlueStore quản lý 1 thiết bị lưu trữ duy nhất (Primary). Thiết bị này gồm 2 phần:

OSD metadata: Một phân vùng nhỏ định dạng XFS chứa metadata cho OSD, chứa thông tin về OSD như: its identifier, which cluster it belongs to, và its private keyring.
Data Phân vùng còn lại được quản lý bởi BlueStore chứa OSD data.

Có thể sử dụng thêm 2 thiết bị bổ sung:
- WAL (write-ahead-log): lưu trữ BlueStore internal journal hoặc write-ahead log. Nó được xác định bởi block.wal symbolic link trong thư mục data. Sử dụng WAL chỉ khi thiết bị nhanh hơn thiết bị primary, ví dụ: khi WAL sử dụng 1 SSD và primary sử dụng 1 HDD.
- DB: Lưu trữ BlueStore internal metadata thay vì trên thiết bị Primary để cài thiện hiệu năng. Nếu DB full, thì nó sẽ thêm metadata vào thiết bị primary. Chỉ sử dụng DB nếu nó nhanh hơn thiết bị Primary

Nếu kích thước SSD nhỏ (vd: nhỏ hơn 1 GB), nên sử dụng nó làm WAL device. Nếu có nhiều hơn thì nên triển khai DB device. BlueStore journal sẽ luôn nằm trên SSD, nên sử dụng DB device sẽ cho lợi ích tương tự với WAL đồng thời cho phép metadata lưu ở đó.

1 single-device BlueStore OSD có thể triển khai như sau:
```sh
ceph-volume lvm prepare --bluestore --data <device>
```
Để chỉ định WAL device và DB device,
```sh
ceph-volume lvm prepare --bluestore --data <device> --block.wal <wal-device> --block.db <db-device>
```
***Lưu ý:***`--data` có thể là 1 Logical Volume sử dụng vg/lv. Các device khác có thể là Logical Volume hoặc phân vùng GPT

## 2. Phương án triển khai

Có nhiều cách để trienr khai Bluestore OSD, đây là 2 use cases thường sử dụng nhất

### 2.1 Chỉ có block (data)

Nếu tất cả các thiết bị cụng loại, cùng là HDD và không có SSD, nó sẽ triển khai 1 block duy nhất và không tách `block.db` hoặc `block.wal`. 

lvm call cho 1 device `/dev/sda` như sau:
```sh
ceph-volume lvm create --bluestore --data /dev/sda
```
Nếu logical volumes đã được tạo cho mỗi thiết bị (1 LV sử dụng 100% device), sau đó lvm call cho 1 lv tên là `ceph-vg/block-lv` như sau:
```sh
ceph-volume lvm create --bluestore --data ceph-vg/block-lv
```

### 2.2 Gồm có block và block.db

Nếu kết hợp SSD và HDD, khuyến nghị nên để `block.db` trên SSD và `block` trên HDD. Sizing cho `block.db` càng lớn càng tốt. Tool `ceph-volume` không thể tự tạo nên Volume Groups và Logical Volumes cần tạo thủ công.

Ví du: Giả sử có 4 HDD (sda, sdb, sdc, sdd) và 1 SSD (sdx). Đầu tiên tạo Volume Groups trên HDD:
```sh
$ vgcreate ceph-block-0 /dev/sda
$ vgcreate ceph-block-1 /dev/sdb
$ vgcreate ceph-block-2 /dev/sdc
$ vgcreate ceph-block-3 /dev/sdd
```
Tạo Logical Volumes cho `block`:
```sh
$ lvcreate -l 100%FREE -n block-0 ceph-block-0
$ lvcreate -l 100%FREE -n block-1 ceph-block-1
$ lvcreate -l 100%FREE -n block-2 ceph-block-2
$ lvcreate -l 100%FREE -n block-3 ceph-block-3
```
Tạo 4 OSDs cho 4 HDD, giả sử 200GB SSD trong `/dev/sdx` tạo 4 Logical Volumes, mỗi cái 50GB:
```sh
$ vgcreate ceph-db-0 /dev/sdx
$ lvcreate -L 50GB -n db-0 ceph-db-0
$ lvcreate -L 50GB -n db-1 ceph-db-0
$ lvcreate -L 50GB -n db-2 ceph-db-0
$ lvcreate -L 50GB -n db-3 ceph-db-0
```
Cuối cùng tạo 4 OSDs với `ceph-volume`:
```sh
$ ceph-volume lvm create --bluestore --data ceph-block-0/block-0 --block.db ceph-db-0/db-0
$ ceph-volume lvm create --bluestore --data ceph-block-1/block-1 --block.db ceph-db-0/db-1
$ ceph-volume lvm create --bluestore --data ceph-block-2/block-2 --block.db ceph-db-0/db-2
$ ceph-volume lvm create --bluestore --data ceph-block-3/block-3 --block.db ceph-db-0/db-3
```

Hoạt động này tạo 4 OSDs với `block` trên HDD và 50GB Logical Volume từ SSD.

## 3. Sizing

Khi sử dụng HDD kết hợp SSD, điều quan trọng là tạo `block.db` đủ lớn cho BlueStore. Thông thường `block.db` thường lớn nhất có thể.

Khuyến nghị chung là kích thước `block.db` thường  từ 1% - 4% của kích thước `block `
The general recommendation is to have block.db size in between 1% to 4% of block size. Với RGW workloads, được khuyến nghĩ rằng kích thước `block.db` không nhỏ hơn 4% của kích thước `block` bởi vì RGW sử dụng nhiều để lưu metadata. Ví dụ: kích thước block là 1TB thì kích thước `block.db` không nhỏ hơn 40GB. Với RBD workloads, 1% - 2% của block size thường là đủ.

Nếu không kết hợp HDD và SSD thì không cần thiết tạo 2 logical volumes riêng biệt cho `block.db` (hoặc `block.wal`). Bluestore sẽ từ quản lý trong không gian của `block`.

## Tự động Sizing cache

Bluestore có thể tự động thay đổi kích thước cache khi tc_malloc được cấu hình như là bộ cấp phát bộ nhớ và `bluestore_cache_autotune setting is enabled`, đây là tùy chọn mặc định. Bluestore sẽ duy trì việc sử dụng OSD heap memory theo kích thước được chỉ định qua tùy chọn `osd_memory_target`. Đây là thuật thoán tốt nhất và cache sẽ không nhỏ hơn giá trị chỉ định trong `osd_memory_cache_min`. Tỉ lệ cache sẽ đươch chọn dựa trên hệ thống phân cấp ưu tiên. Nếu thông tin ưu tiên không có sẵn,`bluestore_cache_meta_ratio` và `bluestore_cache_kv_ratio` được sử dụng dự phòng.

## Sizing thủ công

Lương bộ nhớ sử dụng của mỗi OSD cho BlueStore’s cache được xác đinh trong cấu hình `bluestore_cache_size`. Nếu không được set thì mặc định là 0, đây là 1 giá trị mặc định khác được sử dụng dựa vào việc sử dụng HDD hay SSD cho thiết bị Primary ( `bluestore_cache_size_ssd` `bluestore_cache_size_hdd`).

cache memory budget được cấu hình có thể sử dụng trong 1 số cách khác nhau:
- Key/Value metadata (i.e., RocksDB’s internal cache)
- BlueStore metadata
- BlueStore data (i.e., recently read or written object data)


Việc sử dụng bộ nhớ cache bị chi phối bởi các tùy chọn sau: `bluestore_cache_meta_ratio` và `bluestore_cache_kv_ratio`. Fraction của cache cho data bị chi phối bởi kích thước bluestore cache (dựa vào `bluestore_cache_size[_ssd|_hdd]` và device class của Primary device) cũng như tỷ lệ meta và kv. The data fraction can be calculated by <effective_cache_size> * (1 - bluestore_cache_meta_ratio - bluestore_cache_kv_ratio)

data fraction có thể được tính bằng:
```sh
<effective_cache_size> * (1 - bluestore_cache_meta_ratio - bluestore_cache_kv_ratio)
```

## Checksums

## Tài liệu tham khảo
- https://docs.ceph.com/docs/octopus/rados/configuration/bluestore-config-ref/
- https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/3/html/administration_guide/osd-bluestore
- https://ceph.io/community/bluestore-default-vs-tuned-performance-comparison/
