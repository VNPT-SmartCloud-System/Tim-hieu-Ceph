# Tìm hiểu Output của `ceph df`

## Xem xét ví dụ sau:

Giả sử có 1 cụm Ceph với 2 pool `cephfs_data` và `cephfs_meta` như sau:

Ban đầu pool `cephfs_data` chưa có dữ liệu

```sh
$ ceph df

RAW STORAGE:
    CLASS     SIZE       AVAIL      USED        RAW USED     %RAW USED
    hdd       60 GiB     58 GiB     130 MiB      2.1 GiB          3.55
    TOTAL     60 GiB     58 GiB     130 MiB      2.1 GiB          3.55

POOLS:
    POOL            ID     STORED      OBJECTS     USED        %USED     MAX AVAIL
    cephfs_data      1         0 B           0         0 B         0        27 GiB
    cephfs_meta      2     125 MiB          54     126 MiB      0.22        55 GiB
```

Trong section `GGLOBAL`:
- `total_used_bytes:136249344`
- `total_used_raw_bytes:2283732992`

Trong section `POOL`:
- cephfs_meta:
  - `bytes_used:0`
- cephfs_meta:
  - `bytes_used:131923968`

- Ghi dữ liệu vào pool `cephfs_data`
```
seq 1 4000 |xargs -I {} -P 10 dd if=/dev/zero of={} bs=1K seek=4095 count=1
```
```sh
# ceph df
RAW STORAGE:
    CLASS     SIZE       AVAIL      USED        RAW USED     %RAW USED
    hdd       60 GiB     58 GiB     399 MiB      2.4 GiB          3.98
    TOTAL     60 GiB     58 GiB     399 MiB      2.4 GiB          3.98

POOLS:
    POOL            ID     STORED      OBJECTS     USED        %USED     MAX AVAIL
    cephfs_data      1      16 MiB          4k     250 MiB      0.45        55 GiB
    cephfs_meta      2     144 MiB          58     144 MiB      0.26        55 GiB
```
Trong section `GLOBAL`:
- `total_used_bytes:424214528`
- `total_used_raw_bytes:2571698176`

Trong section `POOL`:
- cephfs_meta:
  - `bytes_used:157745152`
- cephfs_meta:
  - `bytes_used:262144000

=> Tổng duong lượng tăng thêm của cả 2 pool là: (1)
```sh
(157745152 - 0) + (262144000 - 131923968) = 287965184
```
=> Lượng USER ở GLOBAL tăng thêm là: (2)
```sh
424214528 - 136249344 = 287965184

=> Từ (1) và (2) => Kết quả khớp
```

## Thông số USED ở pool

4000 object sử dụng 250MiB=262144000 Byte 

=> 1 object là 65536 Byte

## Một số thông số của `ceph df`
- `ceph df` trong section GLOBAL:
  - `USED` show không gian (của toàn bộ OSD) được phân bổ cho data objects trên block(slow) device.
  - `RAW USED` gồm 2 phần là dung lương của `USED` 1 phần khác.
- `ceph df` trong section POOLS:
  - `BYTES USED` đổi tên thành `STORED`. Thể hiện lượng dữ liệu được lưu trữ bởi người dùng
  - `USED` biểu thị lượng không gian được phân bổ hoàn toàn cho dữ liệu bởi tất cả các Node OSD tính bằng KB.
  - `QUOTA BYTES`, `QUOTA OBJECTS` không được hiển thị trong mode `non-detailed`.

## Tài liệu tham khảo
- https://docs.ceph.com/docs/master/releases/nautilus/
- https://forum.proxmox.com/threads/ceph-raw-usage-grows-by-itself.38395/
- https://stackoverflow.com/questions/51022467/ceph-raw-storage-usage-versus-pool-storage-usage
