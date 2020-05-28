# Cấu hình Multi Site

## Mô hình Lab

|Server|Hostname|IP|
|------|--------|--|
|Master|ceph01|192.168.1.1|
|Secondary|ceph02|192.168.1.2|

## 1. Khởi tạo RGW trên 2 site

- **Thực hiện trên Site 1 - Master Zone**
```sh
ceph-deploy --overwrite-conf config push ceph01
ceph-deploy rgw create ceph01
systemctl restart ceph-radosgw@rgw.ceph01.service
systemctl enable ceph-radosgw@rgw.ceph01.service
```
- **Thực hiện trên Site 2 - Secondary Zone**
```sh
ceph-deploy --overwrite-conf config push ceph02
ceph-deploy rgw create ceph02
systemctl restart ceph-radosgw@rgw.ceph02.service
systemctl enable ceph-radosgw@rgw.ceph02.service
```
## 2. Cấu hình Master Zone

### 2.2 Tạo RGW multi-site 

Realm lưu trữ zone groups, zones, và 1 time `period` với nhiều `epochs` cho việc tracking thay đổi tới configuration.

Chạy lệnh sau trên Node `ceph01` để tạo `realm`:
```sh
radosgw-admin realm create --rgw-realm=movies --default
```
### 1.3 Tạo Master zonegroup. 

1 RGW realm phải có ít nhật 1 RGW zone group, nó sẽ phục vụ như Master zone group cho realm. Chạy command sau trên Node `ceph01` để tạo Master zone group
```sh
radosgw-admin zonegroup create \
--rgw-zonegroup=vn \
--endpoints=http://192.168.1.1:8080 \
--rgw-realm=movies \
--master --default
```
### 1.4. Tạo Master zone

1 RGW zone group phải có ít nhất 1 RGW zone. Chạy command sau trên Node `ceph01` để tạo Master zone:
```sh
radosgw-admin zone create --rgw-zonegroup=vn \
--rgw-zone=ceph01 \
--endpoints=http://192.168.1.1:8080 \
--master --default
```

### 1.5. Remove thông tin default zonegroup và zone từ Cluster 1:
```sh
radosgw-admin zonegroup remove --rgw-zonegroup=default --rgw-zone=default
radosgw-admin zone delete --rgw-zone=default
radosgw-admin zonegroup delete --rgw-zonegroup=default
```
Update period với zonegroup `vn` mới tạo và zone `ceph01`:
```sh
radosgw-admin period update --commit
```
### 1.6 Remove RGW default pools:
```sh
for i in `ceph osd pool ls | grep default.rgw`; do ceph osd pool delete $i $i --yes-i-really-really-mean-it; done
```
### 1.7 Tạo RGW multi-site system user. 

Trong Master zone, tạo 1 system user để thực hiện authentication giữa multi-site radosgw daemons:
```sh
radosgw-admin user create --uid="replication-user" \
--display-name="Multisite v2 replication user" \
--system
```
### 1.8 Cuối cùng, update period với thông tin system user này:
```sh
radosgw-admin zone modify --rgw-zone=ceph01 \
--access-key=HOD20LFF1UWD56Y240XW \
--secret=SJJQC4RS0IczsgQIVRUJoWbBOMsEpNPGFSeElJCR

radosgw-admin period update --commit
```
### 1.9 Update section `[client.rgw.ceph01]` trong file `ceph.conf` với `rgw_zone=ceph01`:
```sh
[client.rgw.ceph01]
rgw_zone=ceph01
```
### 1.10 Restart `ceph01` RGW daemon:
```sh
systemctl restart ceph-radosgw.target
```
## 2. Cấu hình Secondary zone

### 2.2 Đầu tiên, pull RGW realm.

Phải sử dụng RGW endpoint URL và access key và secret key của master zone trong master zonegroup để pull realm tới Node secondary zone RGW:
```sh
radosgw-admin realm pull \
--url=http://192.168.1.1:8080 \
--access-key=HOD20LFF1UWD56Y240XW \
--secret=SJJQC4RS0IczsgQIVRUJoWbBOMsEpNPGFSeElJCR
```
### 2.3 Setup default realm cho RGW multi-site:
```sh
radosgw-admin realm default --rgw-realm=movies
```

```sh
radosgw-admin period pull \
--url=http://192.168.1.1:8080 \
--access-key=HOD20LFF1UWD56Y240XW \
--secret=SJJQC4RS0IczsgQIVRUJoWbBOMsEpNPGFSeElJCR
```
### 2.3 Tạo zone
```sh
radosgw-admin zone create --rgw-zonegroup=vn --rgw-zone=ceph02 \
--access-key=HOD20LFF1UWD56Y240XW \
--secret=SJJQC4RS0IczsgQIVRUJoWbBOMsEpNPGFSeElJCR \
--endpoints=http://192.168.1.2:8080
```
### 2.4 Xóa default zone
```sh
radosgw-admin zone rm --rgw-zone=default
```
```sh
for i in `ceph osd pool ls | grep default.rgw`; do ceph osd pool delete $i $i --yes-i-really-really-mean-it ; done
```
### 2.5 Sửa file cấu hình ceph và thêm vào dòng sau:
```
[client.rgw.ceph02]
...
rgw_zone=ceph02
```
### 2.6 Cập nhật period
```sh
radosgw-admin period update --commit
```
### 2.7 Restart service RGW
```sh
systemctl restart ceph-radosgw@rgw.`hostname -s`
```
### 2.8 Kiểm tra đồng bộ
```sh
radosgw-admin sync status
```
## Tài liệu tham khảo
- https://docs.ceph.com/docs/master/radosgw/multisite/
- https://access.redhat.com/solutions/3366681
