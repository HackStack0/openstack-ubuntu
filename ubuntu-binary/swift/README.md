# Swift 安裝與設定
OpenStack Swift 提供多租戶物件儲存服務，其具備高可擴展性，並利用 RESTful HTTP API 來對大量非結構化資料進行存取與管理。

- [部署前系統環境準備](#部署前系統環境準備)
- [Controller Node](#controller-node)
    - [Controller 安裝前準備](#controller-安裝前準備)
    - [Controller 套件安裝與設定](#controller-套件安裝與設定)
- [Storage Node](#storage-node)
    - [Storage 安裝前準備](#storage-安裝前準備)
    - [Storage 套件安裝與設定](#storage-套件安裝與設定)
- [建立與分散初始 Rings](#建立與分散初始-rings)
    - [建立 Account Ring](#建立-account-ring)
    - [建立 Container Ring](#建立-container-ring)
    - [建立 Object Ring](#建立-object-ring)
    - [將 Rings 分散到儲存節點](#將-rings-分散到儲存節點)
- [完成安裝](#完成安裝)
- [驗證服務](#驗證服務)

# 部署前系統環境準備
當要加入 Swift 來提供物件儲存給雲端租戶時，必須額外新增節點來提供實際資料儲存。首先如教學最開始的步驟，要先設定基本主機環境與安裝基本套件。

這邊會加入兩台 Storage 節點，規格如下所示：
* **Storage Node**: 雙核處理器, 8 GB 記憶體, 250 GB 硬碟（/dev/sda）與 500 GB 硬碟（/dev/sdb）。

在節點上需要提供對映的多張網卡（NIC）來設定給不同網路使用：
* **Management（管理網路）**：10.0.0.0/24，需要一個 Gateway 並設定為 10.0.0.1。
> 這邊需要提供一個 Gateway 來提供給所有節點使用內部的私有網路，該網路必須能夠連接網際網路來讓主機進行套件安裝與更新等。

* **Storage（儲存網路）**：10.0.2.0/24，不需要 Gateway。
> P.S. 該網路並非必要，若想使用如 NAS 或 SAN 的儲存來給 Swift 充當後端儲存的話，可以考慮加入一個獨立的網路給這些儲存系統使用。

這邊將第一張網卡介面設定為 `Management（管理網路）`：
* IP address：10.0.0.51
* Network mask：255.255.255.0 (or /24)
* Default gateway：10.0.0.1

> 另一台以尾數 IP 類推。

<font color=red>P.S.</font> 基本環境安裝與設定請參考[基本套件環境安裝](../basic-binary/README.md)。

# Controller Node
在 Controller 節點我們需要安裝 Swift 中的 Proxy 服務。

### Controller 安裝前準備
Swift 與其他服務不同，Controller 節點不使用任何資料庫，取而代之是在每個 Storage 節點上安裝 SQLite 資料庫。

首先要建立 Service 與 API Endpoint，首先導入 `admin` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Swift 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Swift user
$ openstack user create --domain default --password SWIFT_PASS --email swift@example.com swift

# 新增 Swift 到 Admin Role
$ openstack role add --project service --user swift admin

# 建立 Swift service
$ openstack service create --name swift  --description "OpenStack Object Storage" object-store

# 建立 Swift v1 public endpoints
$ openstack endpoint create --region RegionOne \
object-store public http://10.0.0.11:8080/v1/AUTH_%\(tenant_id\)s

# 建立 Swift v1 internal endpoints
$ openstack endpoint create --region RegionOne \
object-store internal http://10.0.0.11:8080/v1/AUTH_%\(tenant_id\)s

# 建立 Swift v1 admin endpoints
$ openstack endpoint create --region RegionOne \
object-store admin http://10.0.0.11:8080/v1
```

### Controller 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install -y swift swift-proxy python-swiftclient \
python-keystoneclient python-keystonemiddleware memcached
```

安裝完成後，建立用於存放設定檔的目錄：
```sh
$ sudo mkdir -p /etc/swift
```

接著透過網路來下載```proxy-server.conf```範例檔案板模：
```sh
$ sudo curl -o /etc/swift/proxy-server.conf \
https://git.openstack.org/cgit/openstack/swift/plain/etc/proxy-server.conf-sample?h=stable/newton
```

然後編輯`/etc/swift/proxy-server.conf`設定檔，在`[DEFAULT]`部分加入以下內容：
```
[DEFAULT]
bind_port = 8080
user = swift
swift_dir = /etc/swift
```

在`[pipeline:main]`部分加入以下內容：
```
[pipeline:main]
pipeline = catch_errors gatekeeper healthcheck proxy-logging cache container_sync bulk ratelimit authtoken keystoneauth container-quotas account-quotas slo dlo versioned_writes proxy-logging proxy-server
```
> 其他的模組資訊，可以參考 [Deployment Guide](http://docs.openstack.org/developer/swift/deployment_guide.html)。

在`[app:proxy-server]`部分加入以下內容：
```
[app:proxy-server]
...
use = egg:swift#proxy
account_autocreate = true
```

在`[filter:keystoneauth]`部分加入以下內容：
```
[filter:keystoneauth]
...
use = egg:swift#keystoneauth
operator_roles = admin,user
```

在`[filter:authtoken]`部分加入以下內容：
```
[filter:authtoken]
...
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = swift
password = SWIFT_PASS
delay_auth_decision = True
```
> 這邊`SWIFT_PASS`可以隨需求修改。

在`[filter:cache]`部分加入以下內容：
```
[filter:cache]
...
use = egg:swift#memcache
memcache_servers = 10.0.0.11:11211
```

# Storage Node
安裝與設定完成 Controller 上的 Swift 所有服務後，接著要來設定實際儲存資料的 Storage 節點，其中會提供 Account、Container 以及Object 服務。

### Storage 安裝前準備
在開始設定之前，首先要安裝 Swift 相依的相關軟體：
```sh
$ sudo apt-get install -y xfsprogs rsync
```

然後將非系統使用的硬碟進行格式化來當作 Swift 資料儲存用：
```sh
$ sudo mkfs.xfs -f /dev/sdb
```
> 若有多顆則以此方式重複進行。

接著建立一個目錄來提供給該硬碟進行掛載使用：
```sh
$ sudo mkdir -p /srv/node/sdb
```
> 若有多顆則以此方式重複進行。如以下方式：
```sh
$ sudo mkdir -p /srv/node/sdc
```

完成後，編輯`/etc/fstab`設定檔，來提供開機時自動掛載：
```
/dev/sdb /srv/node/sdb xfs noatime,nodiratime,nobarrier,logbufs=8 0 2
```
> 若有多顆則以此方式重複進行。

當上述都沒問題後，就可以將剛建立的所有硬碟進行掛載，使用以下指令：
```sh
$ sudo mount /srv/node/sdb
```
> 若有多顆則以此方式重複進行。

接著編輯`/etc/rsyncd.conf`設定檔加入以下內容：
```
uid = swift
gid = swift
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
address = MANAGEMENT_IP

[account]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/account.lock

[container]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/container.lock

[object]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/object.lock
```
> P.S. `MANAGEMENT_IP`取代為 Storage 節點的管理網路 IP。

編輯`/etc/default/rsync`設定檔，來進行 rsync 異地備援設定：
```
RSYNC_ENABLE=true
```

完成上述後就可以重新啟動 rsync 服務：
```sh
$ sudo service rsync start
```

### Storage 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install -y swift swift-account swift-container swift-object
```

安裝完成後，要從網際網路上下載 Swift 的 Account、Container 與 Object Server 的設定檔：
```sh
# Account server
$ sudo curl -o /etc/swift/account-server.conf \
https://git.openstack.org/cgit/openstack/swift/plain/etc/account-server.conf-sample?h=stable/newton

# Container server
$ sudo curl -o /etc/swift/container-server.conf \
https://git.openstack.org/cgit/openstack/swift/plain/etc/container-server.conf-sample?h=stable/newton

# Object server
$ sudo curl -o /etc/swift/object-server.conf \
https://git.openstack.org/cgit/openstack/swift/plain/etc/object-server.conf-sample?h=stable/newton
```

首先編輯`/etc/swift/account-server.conf`設定檔，在`[DEFAULT]`部分設定以下內容：
```
[DEFAULT]
...
bind_ip = MANAGEMENT_IP
bind_port = 6202
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = True
```
> P.S. `MANAGEMENT_IP`取代為 Storage 節點的管理網路 IP。

在`[pipeline:main]`部分加入以下內容：
```
[pipeline:main]
pipeline = healthcheck recon account-server
```
> 其他的模組資訊，可以參考 [Deployment Guide](http://docs.openstack.org/developer/swift/deployment_guide.html)。

在`[filter:recon]`部分加入以下內容：
```
[filter:recon]
...
use = egg:swift#recon
recon_cache_path = /var/cache/swift
```

編輯`/etc/swift/container-server.conf`設定檔，在`[DEFAULT]`部分設定以下內容：
```
[DEFAULT]
...
bind_ip = MANAGEMENT_IP
bind_port = 6201
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = true
```
> P.S. `MANAGEMENT_IP`取代為 Storage 節點的管理網路 IP。

在`[pipeline:main]`部分加入以下內容：
```
[pipeline:main]
pipeline = healthcheck recon container-server
```
> 其他的模組資訊，可以參考 [Deployment Guide](http://docs.openstack.org/developer/swift/deployment_guide.html)。

在`[filter:recon]`部分加入以下內容：
```
[filter:recon]
...
use = egg:swift#recon
recon_cache_path = /var/cache/swift
```

編輯`/etc/swift/object-server.conf`設定檔，在`[DEFAULT]`部分設定以下內容：
```
[DEFAULT]
...
bind_ip = MANAGEMENT_IP
bind_port = 6200
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = true
```
> P.S. `MANAGEMENT_IP`取代為 Storage 節點的管理網路 IP。

在`[pipeline:main]`部分加入以下內容：
```
[pipeline:main]
pipeline = healthcheck recon object-server
```
> 其他的模組資訊，可以參考 [Deployment Guide](http://docs.openstack.org/developer/swift/deployment_guide.html)。

在`[filter:recon]`部分加入以下內容：
```
[filter:recon]
use = egg:swift#recon
...
recon_cache_path = /var/cache/swift
recon_lock_path = /var/lock
```

完成上述所有設定後，要確保 Swift 能夠存取掛載目錄，這邊需要更改權限：
```
$ sudo chown -R swift:swift /srv/node
```

接著建立一個目錄讓 Swift 作為快取是使用：
```
sudo mkdir -p /var/cache/swift
sudo chown -R root:swift /var/cache/swift
sudo chmod -R 775 /var/cache/swift
```

# 建立與分散初始 Rings
在這個階段進行之前，要確保 Controller 與兩台 Storage 節點都確定設定與安裝完成 Swift。若沒問題的話，回到`Controller`節點，並進入`/etc/swift` 目錄來完成以下步驟。
```sh
$ cd /etc/swift
```

### 建立 Account Ring
Account Server 使用 Account Ring 來維護容器的列表。首先透過以下指令建立一個 `account.builder` 檔案：
```sh
$ sudo swift-ring-builder account.builder create 10 2 1
```
> 這邊使用單一 Region 及兩個 Zone，其中最大分區為`2 ^ 10 (1024)`、每個物件有`2`副本，並且物件多次移動分區之間的最小時間為`1`小時

然後新增每一個`Storage`節點的 Account Server 資訊到 Ring 中：
```sh
# Object1 sdb
$ sudo swift-ring-builder account.builder \
add --region 1 --zone 1 --ip 10.0.0.51 \
--port 6202 --device sdb --weight 100

# Object2 sdb
$ sudo swift-ring-builder account.builder \
add --region 1 --zone 2 --ip 10.0.0.52 \
--port 6202 --device sdb --weight 100
```

當完成後，即可透過以下指令來驗證是否正確：
```sh
$ sudo swift-ring-builder account.builder
account.builder, build version 2
1024 partitions, 2.000000 replicas, 1 regions, 2 zones, 2 devices, 100.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:00:00 remaining)
The overload factor is 0.00% (0.000000)
Devices:   id region zone ip address:port replication ip:port  name weight partitions balance flags meta
            0      1    1  10.0.0.51:6202      10.0.0.51:6202   sdb 100.00          0 -100.00
            1      1    2  10.0.0.52:6202      10.0.0.52:6202   sdb 100.00          0 -100.00
```

若沒有任何問題，即可將 Ring 進行重新平衡調整：
```sh
$ sudo swift-ring-builder account.builder rebalance
Reassigned 2048 (200.00%) partitions. Balance is now 0.00.  Dispersion is now 0.00
```

### 建立 Container Ring
Container Server 使用 Container Ring 來維護物件的列表。透過以下指令建立一個`container.builder`檔案：
```sh
$ sudo swift-ring-builder container.builder create 10 2 1
```

然後新增每一個`Storage`節點的 Container Server 資訊到 Ring 中：
```sh
# Object1 sdb
$ sudo swift-ring-builder container.builder \
add --region 1 --zone 1 --ip 10.0.0.51 \
--port 6201 --device sdb --weight 100

# Object2 sdb
$ sudo swift-ring-builder container.builder \
add --region 1 --zone 2 --ip 10.0.0.52 \
--port 6201 --device sdb --weight 100
```

當完成後，即可透過以下指令來驗證是否正確：
```sh
$ sudo swift-ring-builder container.builder
container.builder, build version 2
1024 partitions, 2.000000 replicas, 1 regions, 2 zones, 2 devices, 100.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:00:00 remaining)
The overload factor is 0.00% (0.000000)
Devices:   id region zone ip address:port replication ip:port  name weight partitions balance flags meta
            0      1    1  10.0.0.51:6201      10.0.0.51:6201   sdb 100.00          0 -100.00
            1      1    2  10.0.0.52:6201      10.0.0.52:6201   sdb 100.00          0 -100.00
```

若沒有任何問題，即可將 Ring 進行重新平衡調整：
```sh
$ sudo swift-ring-builder container.builder rebalance
Reassigned 2048 (200.00%) partitions. Balance is now 0.00.  Dispersion is now 0.00
```

### 建立 Object Ring
Object Server 使用 Object Ring 來維護本地裝置上物件位置的列表。。透過以下指令建立一個`object.builder`檔案：
```sh
$ sudo swift-ring-builder object.builder create 10 2 1
```

然後新增每一個`Storage`節點的 Object Server 資訊到 Ring 中：
```sh
# Object1 sdb
$ sudo swift-ring-builder object.builder \
add --region 1 --zone 1 --ip 10.0.0.51 \
--port 6200 --device sdb --weight 100

# Object2 sdb
$ sudo swift-ring-builder object.builder \
add --region 1 --zone 2 --ip 10.0.0.52 \
--port 6200 --device sdb --weight 100
```

當完成後，即可透過以下指令來驗證是否正確：
```sh
$ sudo swift-ring-builder object.builder
object.builder, build version 2
1024 partitions, 2.000000 replicas, 1 regions, 2 zones, 2 devices, 100.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:00:00 remaining)
The overload factor is 0.00% (0.000000)
Devices:   id region zone ip address:port replication ip:port  name weight partitions balance flags meta
            0      1    1  10.0.0.51:6200      10.0.0.51:6200   sdb 100.00          0 -100.00
            1      1    2  10.0.0.52:6200      10.0.0.52:6200   sdb 100.00          0 -100.00
```

若沒有任何問題，即可將 Ring 進行重新平衡調整：
```sh
$ sudo swift-ring-builder object.builder rebalance
Reassigned 2048 (200.00%) partitions. Balance is now 0.00.  Dispersion is now 0.00
```

### 將 Rings 分散到儲存節點
接著我們要將上述建立的所有 Ring 檔案傳送到所有`Storage`節點上的`/etc/swift`目錄，這邊透過以下方式：
```sh
scp account.ring.gz container.ring.gz object.ring.gz object1:~/
scp account.ring.gz container.ring.gz object.ring.gz object2:~/

ssh object1 "sudo mv ~/*.ring.gz /etc/swift/"
ssh object2 "sudo mv ~/*.ring.gz /etc/swift/"
```

# 完成安裝
若上面步驟都進行順利的話，接下來要進入最後階段，首先回到`Controller`節點，透過網路下載`/etc/swift/swift.conf`設定檔：
```sh
$ sudo curl -o /etc/swift/swift.conf \
https://git.openstack.org/cgit/openstack/swift/plain/etc/swift.conf-sample?h=stable/newton
```

接著編輯`/etc/swift/swift.conf`設定檔，在`[swift-hash]`部分設定 Path 的 Hash 字首字尾：
```sh
[swift-hash]
...
swift_hash_path_suffix = 1505cb4249801981da86
swift_hash_path_prefix = 42da359c6af55b2e3f7d
```
> 可透過`openssl rand -hex 10`產生。

在`[storage-policy:0]`部分加入以下內容：
```
[storage-policy:0]
...
name = Policy-0
default = yes
```

完成後，複製`/etc/swift/swift.conf`設定檔到所有`Storage`節點上的`/etc/swift`目錄：
```sh
scp /etc/swift/swift.conf object1:~/
scp /etc/swift/swift.conf object2:~/

ssh object1 "sudo mv ~/swift.conf /etc/swift/"
ssh object2 "sudo mv ~/swift.conf /etc/swift/"
```

在所有安裝 Swift 的節點上更換`/etc/swift`目錄的擁有者與權限：
```sh
sudo chown -R root:swift /etc/swift
ssh object1 "sudo chown -R root:swift /etc/swift"
ssh object2 "sudo chown -R root:swift /etc/swift"
```

完成後在`Controller`節點重新 memcached 與 Swift 服務：
```sh
sudo service memcached restart
sudo swift-init all restart
```

在所有`Storage`節點重新啟動 Swift 服務：
```sh
ssh object1 "sudo swift-init all restart"
ssh object2 "sudo swift-init all restart"
```

# 驗證服務
首先回到`Controller`節點，接著導入`admin`帳號來驗證服務：
```sh
$ . admin-openrc
```

這邊可以透過 Swift client 來查看服務狀態，如以下方式：
```sh
$ swift stat
        Account: AUTH_ba57e6880afe403dbf78f946766df5e0
     Containers: 0
        Objects: 0
          Bytes: 0
X-Put-Timestamp: 1476255727.19986
    X-Timestamp: 1476255727.19986
     X-Trans-Id: txed49f33bf08a4f7da5302-0057fddfee
   Content-Type: text/plain; charset=utf-8
```

由於 OpenStack Client 已經整合 Swift 指令，故可以透過其建立 Container：
```sh
$ openstack container create container1
+---------------------------------------+------------+------------------------------------+
| account                               | container  | x-trans-id                         |
+---------------------------------------+------------+------------------------------------+
| AUTH_ba57e6880afe403dbf78f946766df5e0 | container1 | tx13ab9b24acc84d58b6ffc-0057fde064 |
+---------------------------------------+------------+------------------------------------+
```

接著上傳一個檔案進行測試，透過以下指令：
```sh
$ openstack object create container1 [FILE]
+------------------------------+------------+----------------------------------+
| object                       | container  | etag                             |
+------------------------------+------------+----------------------------------+
| cirros-0.3.4-x86_64-disk.img | container1 | ee1eca47dc88f4879d8a229cc70a07c6 |
+------------------------------+------------+----------------------------------+
```

而下載可以透過以下指令進行：
```sh
$ openstack object save container1 [FILE]

```
