# RUNBOOK – Cấu hình HA 2 OPNsense trên VMware Cloud Director

## 1. Mục tiêu

Tài liệu này hướng dẫn triển khai mô hình **High Availability Active/Passive** cho hai firewall OPNsense chạy trên VMware Cloud Director.

Giải pháp sử dụng:

- **CARP** để cung cấp Virtual IP trên LAN và WAN.
- **pfsync** để đồng bộ firewall states.
- **XMLRPC Sync** để đồng bộ cấu hình từ Primary sang Backup.
- **Unicast CARP** để phù hợp với môi trường VMware Cloud Director / NSX.
- **MAC Learning / MAC Discovery** ở lớp VMware networking để CARP virtual MAC hoạt động đúng.
- Một network riêng cho pfsync.

Mục tiêu sau triển khai:

- Client LAN sử dụng một gateway duy nhất.
- Không cần thay đổi default gateway khi firewall failover.
- LAN và WAN VIP được chuyển từ Primary sang Backup khi Primary mất hoàn toàn.
- Firewall state được đồng bộ giữa hai node.
- Có thể sử dụng WAN CARP VIP làm Source NAT IP chung sau khi WAN VIP đã được provider network hỗ trợ đầy đủ.

---

# 2. Thiết kế IP

## 2.1. OPNsense-01 – Primary

| Interface | IP Address | Chức năng |
|---|---|---|
| WAN | `61.14.236.213/26` | WAN node Primary |
| LAN | `10.0.0.1/24` | LAN node Primary |
| PFSYNC | `172.16.255.1/24` | HA state/config synchronization |

## 2.2. OPNsense-02 – Backup

| Interface | IP Address | Chức năng |
|---|---|---|
| WAN | `61.14.236.217/26` | WAN node Backup |
| LAN | `10.0.0.10/24` | LAN node Backup |
| PFSYNC | `172.16.255.2/24` | HA state/config synchronization |

## 2.3. CARP Virtual IP

| Network | CARP VIP | VHID | CARP virtual MAC |
|---|---|---:|---|
| WAN | `61.14.236.218/26` | `1` | `00:00:5e:00:01:01` |
| LAN | `10.0.0.253/24` | `2` | `00:00:5e:00:01:02` |

## 2.4. Default gateway

WAN upstream gateway:

```text
61.14.236.193
```

LAN client gateway:

```text
10.0.0.253
```

---

# 3. Kiến trúc

```text
                              Internet
                                 |
                       Upstream Gateway
                        61.14.236.193
                                 |
             +-------------------+-------------------+
             |                                       |
        OPNsense-01                            OPNsense-02
        PRIMARY                               BACKUP
             |                                       |
      WAN 61.14.236.213                  WAN 61.14.236.217
             \                                       /
              +------ WAN CARP VIP ----------------+
                     61.14.236.218/26
                     VHID 1
                     vMAC 00:00:5e:00:01:01

      LAN 10.0.0.1                       LAN 10.0.0.10
             \                                       /
              +------ LAN CARP VIP ----------------+
                     10.0.0.253/24
                     VHID 2
                     vMAC 00:00:5e:00:01:02
                              |
                           Clients
                     GW = 10.0.0.253

      PFSYNC 172.16.255.1  <------->  PFSYNC 172.16.255.2
```

Mỗi OPNsense sử dụng **3 network adapter**:

```text
NIC 1 = WAN
NIC 2 = LAN
NIC 3 = PFSYNC
```

Không cần tạo NIC thứ tư cho WAN CARP VIP.

---

# 4. Phần A – Chuẩn bị trên vCenter

> Phần này áp dụng cho các network/port group được quản lý trực tiếp bởi vSphere.
>
> Nếu network của VCD là **NSX-backed Segment**, không cấu hình MAC Learning trực tiếp trên vCenter port group để thay thế cho NSX Segment Profile. Với NSX-backed network, thực hiện thêm Phần B.

## 4.1. Placement của hai firewall

Khuyến nghị đặt hai OPNsense trên hai ESXi host khác nhau.

Ví dụ:

```text
OPNsense-01 → ESXi-Host-01
OPNsense-02 → ESXi-Host-02
```

Nếu cluster có DRS, nên tạo **VM-VM Anti-Affinity Rule**:

```text
VMs:
- OPNsense-01
- OPNsense-02

Rule:
Separate Virtual Machines
```

Mục tiêu là tránh cả hai firewall cùng mất khi một ESXi host down.

---

## 4.2. Kiểm tra VM network adapters

Mỗi VM phải có ba vNIC:

```text
Network Adapter 1 → WAN Network
Network Adapter 2 → LAN Network
Network Adapter 3 → PFSYNC Network
```

Khuyến nghị sử dụng cùng loại adapter trên cả hai node, ví dụ:

```text
VMXNET3
```

Interface order phải giống nhau trên cả hai VM để OPNsense interface assignments đồng nhất.

---

## 4.3. Port-group security nếu network được quản lý trực tiếp bởi vCenter

Nếu WAN/LAN là Distributed Port Group hoặc Standard Port Group **không do NSX quản lý**, CARP có thể cần cho phép guest sử dụng virtual MAC khác với MAC được gán cho vNIC.

Trong vSphere Client:

```text
Networking
→ chọn Distributed Port Group
→ Configure / Edit Settings
→ Security
```

Áp dụng trên port group chuyên dụng cho OPNsense HA:

```text
MAC Address Changes = Accept
Forged Transmits    = Accept
```

Không bật `Promiscuous Mode` nếu không có yêu cầu cụ thể.

Nếu đang sử dụng MAC Learning ở Distributed Port Group:

```text
MAC Learning = Enabled
Forged Transmits = Accept
```

Không nên bật đồng thời:

```text
MAC Learning = Enabled
Promiscuous Mode = Accept
```

trên các phiên bản vSphere hiện đại vì đây không phải tổ hợp cấu hình được khuyến nghị.

### Khuyến nghị

Tạo dedicated Port Group chỉ dành cho HA firewall thay vì thay security policy của một port group dùng chung.

---

# 5. Phần B – Chuẩn bị trên VMware Cloud Director / NSX

## 5.1. Tạo ba network

Cần ba network logic.

### WAN Network

WAN phải cung cấp L2 connectivity cho cả hai OPNsense.

Các IP sử dụng:

```text
OPNsense-01 = 61.14.236.213/26
OPNsense-02 = 61.14.236.217/26
CARP VIP    = 61.14.236.218/26

Gateway     = 61.14.236.193
```

WAN có thể là:

```text
Direct Org VDC Network
```

hoặc network tương đương được provider expose từ External Network.

WAN VIP `.218` phải:

- Thuộc subnet `/26`.
- Không được cấp cho VM khác.
- Không nằm trong conflict với IP pool đang sử dụng.
- Được upstream/provider network cho phép sử dụng.

---

## 5.2. LAN Network

Tạo một Isolated Org VDC Network:

```text
Name: OPN-LAN-HA
Subnet: 10.0.0.0/24
```

Gắn vào:

```text
OPNsense-01 LAN
OPNsense-02 LAN
Client / workload cần sử dụng firewall
```

Không cấu hình gateway VCD trên network này nếu OPNsense đóng vai trò gateway.

Gateway của workload sẽ là:

```text
10.0.0.253
```

---

## 5.3. PFSYNC Network

Tạo network riêng:

```text
Name: OPN-PFSYNC
Subnet: 172.16.255.0/24
Type: Isolated
```

Network này chỉ nên gắn vào:

```text
OPNsense-01 PFSYNC NIC
OPNsense-02 PFSYNC NIC
```

Không dùng network này cho workload khác.

---

## 5.4. Gắn network cho hai VM

### OPNsense-01

```text
NIC 1 → WAN Network
NIC 2 → OPN-LAN-HA
NIC 3 → OPN-PFSYNC
```

### OPNsense-02

```text
NIC 1 → WAN Network
NIC 2 → OPN-LAN-HA
NIC 3 → OPN-PFSYNC
```

NIC order phải giống nhau.

---

# 6. Phần C – NSX Segment Profile cho CARP

> Áp dụng khi Org VDC Network của VCD được backed bởi NSX Segment.

CARP sử dụng virtual MAC khác với MAC của vNIC.

Ví dụ:

```text
WAN VHID 1
CARP vMAC = 00:00:5e:00:01:01

LAN VHID 2
CARP vMAC = 00:00:5e:00:01:02
```

NSX phải cho phép học và forward các MAC này.

---

## 6.1. Tạo MAC Discovery Profile

Trong NSX Manager:

```text
Networking
→ Segments
→ Segment Profiles
→ MAC Discovery
→ Add MAC Discovery Profile
```

Ví dụ:

```text
Name: OPNsense-CARP-MAC-Discovery
```

Thiết lập:

```text
MAC Change:               Yes
MAC Learning:             Yes
MAC Learning Aging Time:  600
Unknown Unicast Flooding: Yes
MAC Limit:                4096
MAC Limit Policy:         Allow
```

Trong môi trường đã triển khai, `Unknown Unicast Flooding = Yes` là cần thiết để traffic tới CARP vMAC trên LAN được forward đúng.

---

## 6.2. Gán profile vào LAN Segment

Trong NSX Manager:

```text
Networking
→ Segments
→ chọn Segment backing cho OPN-LAN-HA
→ Edit
→ Segment Profiles
```

Chọn:

```text
MAC Discovery Profile:
OPNsense-CARP-MAC-Discovery
```

Save.

---

## 6.3. Gán profile vào WAN Segment nếu WAN là NSX-backed Segment

Nếu WAN Direct Network/External Network sử dụng NSX Segment và CARP vMAC phải đi qua segment đó, áp dụng cùng nguyên tắc:

```text
MAC Change:               Yes
MAC Learning:             Yes
Unknown Unicast Flooding: theo thiết kế/provider requirement
MAC Limit Policy:         Allow
```

Việc sử dụng `Unknown Unicast Flooding` trên WAN phải phù hợp với provider network design.

Nếu WAN backing là vSphere Distributed Port Group trực tiếp thay vì NSX Segment, sử dụng policy ở phần vCenter thay vì NSX Segment Profile.

---

# 7. Phần D – Cài đặt interface trên OPNsense-01

Trên OPNsense-01:

```text
Interfaces
→ Assignments
```

Assign:

```text
WAN
LAN
PFSYNC
```

Interface name có thể tương ứng:

```text
vmx0 = LAN
vmx1 = WAN
vmx2 = PFSYNC
```

Tên thực tế phụ thuộc thứ tự NIC.

---

## 7.1. WAN

```text
Interfaces
→ WAN
```

Cấu hình:

```text
IPv4 Configuration Type: Static IPv4
IPv4 Address: 61.14.236.213/26
Gateway: 61.14.236.193
```

Enable interface và Save.

---

## 7.2. LAN

```text
Interfaces
→ LAN
```

Cấu hình:

```text
IPv4 Configuration Type: Static IPv4
IPv4 Address: 10.0.0.1/24
```

---

## 7.3. PFSYNC

```text
Interfaces
→ PFSYNC
```

Cấu hình:

```text
Enable Interface: Yes
IPv4 Configuration Type: Static IPv4
IPv4 Address: 172.16.255.1/24
```

---

# 8. Phần E – Cài đặt interface trên OPNsense-02

Thực hiện cùng interface assignment với Primary.

---

## 8.1. WAN

```text
IPv4 Address: 61.14.236.217/26
Gateway: 61.14.236.193
```

---

## 8.2. LAN

```text
IPv4 Address: 10.0.0.10/24
```

---

## 8.3. PFSYNC

```text
IPv4 Address: 172.16.255.2/24
```

---

# 9. Phần F – Firewall rules phục vụ HA

OPNsense yêu cầu CARP advertisement được phép trên interface tương ứng.

---

## 9.1. LAN CARP rule

Trên LAN của cả hai node:

```text
Firewall
→ Rules
→ LAN
```

Tạo rule:

```text
Action: Pass
Protocol: CARP
Source: LAN net
Destination: any
```

Do thiết kế đang dùng **Unicast CARP**, nên có thể giới hạn source/destination theo node IP.

Ví dụ Primary:

```text
Source: 10.0.0.10
Destination: 10.0.0.1
Protocol: CARP
```

Backup tương ứng ngược lại.

---

## 9.2. WAN CARP rule

Trên WAN của cả hai node:

```text
Protocol: CARP
```

Có thể giới hạn peer:

```text
Primary:
Source      = 61.14.236.217
Destination = 61.14.236.213

Backup:
Source      = 61.14.236.213
Destination = 61.14.236.217
```

---

## 9.3. PFSYNC network rules

Trên interface PFSYNC của cả hai node, cho phép traffic giữa hai địa chỉ:

```text
172.16.255.1
172.16.255.2
```

Có thể sử dụng rule đơn giản:

```text
Action: Pass
Interface: PFSYNC
Protocol: any
Source: PFSYNC net
Destination: PFSYNC net
```

Vì đây là dedicated HA network chỉ nối hai firewall.

Nếu cần hardening, giới hạn các protocol cần thiết:

```text
pfsync
HTTPS/XMLRPC
ICMP quản trị
```

---

# 10. Phần G – Cấu hình LAN CARP VIP

## 10.1. Primary

Trên OPNsense-01:

```text
Interfaces
→ Virtual IPs
→ Settings
→ Add
```

Cấu hình:

```text
Mode / Type: CARP
Interface: LAN
Address: 10.0.0.253/24
VHID Group: 2
Password: <CARP_SHARED_SECRET>
Advertising Base: 1
Advertising Skew: 0
Peer IPv4: 10.0.0.10
```

Save và Apply.

---

## 10.2. Backup

Trên OPNsense-02:

```text
Mode / Type: CARP
Interface: LAN
Address: 10.0.0.253/24
VHID Group: 2
Password: <same CARP_SHARED_SECRET>
Advertising Base: 1
Advertising Skew: 100
Peer IPv4: 10.0.0.1
```

Save và Apply.

---

# 11. Phần H – Cấu hình WAN CARP VIP

## 11.1. Primary

Trên OPNsense-01:

```text
Interfaces
→ Virtual IPs
→ Settings
→ Add
```

Cấu hình:

```text
Mode / Type: CARP
Interface: WAN
Address: 61.14.236.218/26
VHID Group: 1
Password: <CARP_SHARED_SECRET>
Advertising Base: 1
Advertising Skew: 0
Peer IPv4: 61.14.236.217
```

---

## 11.2. Backup

Trên OPNsense-02:

```text
Mode / Type: CARP
Interface: WAN
Address: 61.14.236.218/26
VHID Group: 1
Password: <same CARP_SHARED_SECRET>
Advertising Base: 1
Advertising Skew: 100
Peer IPv4: 61.14.236.213
```

---

# 12. Phần I – Cấu hình pfsync

## 12.1. Primary

Trên OPNsense-01:

```text
System
→ High Availability
→ Settings
```

Phần State Synchronization:

```text
Synchronize all states via: PFSYNC
Sync compatibility: OPNsense 24.7 or above
Synchronize Peer IP: 172.16.255.2
Defer pfsync: Disable
```

`Disable preempt`:

```text
Unchecked
```

---

## 12.2. Backup

Trên OPNsense-02:

```text
System
→ High Availability
→ Settings
```

Cấu hình:

```text
Synchronize all states via: PFSYNC
Sync compatibility: OPNsense 24.7 or above
Synchronize Peer IP: 172.16.255.1
```

Không cấu hình XMLRPC synchronization theo chiều Backup → Primary.

---

# 13. Phần J – Cấu hình XMLRPC Sync

XMLRPC chỉ cấu hình trên Primary.

Trên OPNsense-01:

```text
System
→ High Availability
→ Settings
```

Phần:

```text
Configuration Synchronization Settings (XMLRPC Sync)
```

Cấu hình:

```text
Perform synchronization: Enabled
Synchronize Config: 172.16.255.2
Remote System Username: <HA_SYNC_USER>
Remote System Password: <PASSWORD>
```

Có thể dùng `root`, nhưng trong production nên sử dụng account riêng có quyền phù hợp nếu policy tổ chức cho phép.

Chọn các mục cần synchronize.

Khuyến nghị:

```text
Firewall Rules
NAT
Virtual IPs
Aliases
DHCP
IPsec
OpenVPN
Services cần thiết khác
```

Không cấu hình XMLRPC từ OPNsense-02 về OPNsense-01.

---

# 14. Phần K – Cấu hình Preempt

Trên cả hai node:

```text
System
→ High Availability
→ Settings
```

Để:

```text
Disable preempt = Unchecked
```

Mục tiêu là node có priority cao hơn được phép trở lại MASTER khi hệ thống ổn định.

Primary sử dụng:

```text
AdvSkew = 0
```

Backup sử dụng:

```text
AdvSkew = 100
```

---

# 15. Phần L – Cấu hình client LAN

Client phía LAN không sử dụng IP vật lý của từng firewall làm gateway.

Không sử dụng:

```text
10.0.0.1
10.0.0.10
```

Sử dụng:

```text
Default Gateway = 10.0.0.253
```

Ví dụ:

```text
IP Address:      10.0.0.100
Subnet Mask:     255.255.255.0
Default Gateway: 10.0.0.253
DNS:             theo thiết kế
```

---

# 16. Phần M – Outbound NAT

Có hai giai đoạn triển khai.

## 16.1. Giai đoạn ban đầu

Có thể giữ Automatic Outbound NAT để xác nhận HA LAN/WAN và failover node.

Trong trường hợp này:

```text
Traffic qua Primary → public source 61.14.236.213
Traffic qua Backup  → public source 61.14.236.217
```

---

## 16.2. Giai đoạn production với shared WAN public IP

Sau khi WAN CARP VIP `61.14.236.218` đã được upstream/VCD/NSX hỗ trợ đầy đủ, chuyển sang:

```text
Firewall
→ NAT
→ Outbound
```

Chọn:

```text
Hybrid Outbound NAT
```

Tạo rule:

```text
Interface: WAN
TCP/IP Version: IPv4
Protocol: any
Source: 10.0.0.0/24
Destination: any
Translation / target: 61.14.236.218
```

Mục tiêu:

```text
Client
  ↓
10.0.0.253
  ↓
Active OPNsense
  ↓
SNAT
  ↓
61.14.236.218
  ↓
Internet
```

Khi failover:

```text
Primary → Backup
```

public source IP vẫn giữ:

```text
61.14.236.218
```

---

# 17. Phần N – Inbound NAT / Published Services

Nếu có publish service từ Internet, sử dụng WAN CARP VIP hoặc IP Alias gắn vào CARP VHID.

Ví dụ:

```text
61.14.236.218:443
→ Internal Server
```

Tạo Port Forward trên Primary:

```text
Firewall
→ NAT
→ Port Forward
```

XMLRPC Sync sẽ replicate NAT/rules sang Backup nếu mục tương ứng đã được chọn.

Nếu có nhiều public IP bổ sung, nên sử dụng:

```text
IP Alias
```

gắn với WAN CARP VHID thay vì tạo một VHID riêng cho từng public IP.

---

# 18. Phần O – HA trạng thái mong muốn

Trong trạng thái bình thường:

```text
OPNsense-01

LAN VIP 10.0.0.253
MASTER

WAN VIP 61.14.236.218
MASTER
```

```text
OPNsense-02

LAN VIP 10.0.0.253
BACKUP

WAN VIP 61.14.236.218
BACKUP
```

Khi OPNsense-01 down hoàn toàn:

```text
OPNsense-02

LAN VIP 10.0.0.253
MASTER

WAN VIP 61.14.236.218
MASTER
```

Client tiếp tục sử dụng:

```text
Default Gateway = 10.0.0.253
```

---

# 19. Phần P – Trình tự triển khai khuyến nghị

Thực hiện theo đúng thứ tự:

```text
1. Chuẩn bị 2 VM OPNsense.

2. Tạo / chuẩn bị:
   - WAN Network
   - LAN Isolated Network
   - PFSYNC Isolated Network

3. Cấu hình vCenter placement / Anti-Affinity.

4. Cấu hình vSphere port security nếu network không phải NSX-backed.

5. Cấu hình NSX MAC Discovery Profile nếu network là NSX-backed.

6. Gắn 3 NIC giống thứ tự trên hai OPNsense.

7. Cấu hình physical IP:
   - WAN
   - LAN
   - PFSYNC

8. Cấu hình firewall rules phục vụ CARP/pfsync/XMLRPC.

9. Cấu hình LAN CARP VIP.

10. Cấu hình WAN CARP VIP.

11. Cấu hình pfsync trên cả hai node.

12. Cấu hình XMLRPC trên Primary.

13. Cấu hình client sử dụng LAN CARP VIP làm gateway.

14. Đồng bộ firewall/NAT/VIP/services.

15. Cấu hình Outbound NAT dùng WAN CARP VIP sau khi provider network đã support VIP.

16. Đưa hệ thống vào production.
```

---

# 20. Checklist cấu hình cuối cùng

## vCenter

- [ ] OPNsense-01 và OPNsense-02 chạy trên hai ESXi host khác nhau.
- [ ] Có DRS Anti-Affinity nếu cluster hỗ trợ.
- [ ] Hai VM có cùng số lượng và thứ tự vNIC.
- [ ] vNIC type đồng nhất.
- [ ] Dedicated Port Group sử dụng đúng security policy nếu network do vSphere trực tiếp quản lý.

## VCD

- [ ] WAN network gắn vào cả hai firewall.
- [ ] LAN Isolated Network gắn vào cả hai firewall và workload.
- [ ] PFSYNC Isolated Network chỉ gắn vào hai firewall.
- [ ] WAN VIP `.218` không conflict.
- [ ] WAN VIP thuộc subnet/public range hợp lệ.

## NSX

- [ ] MAC Discovery Profile đã tạo.
- [ ] MAC Change = Yes.
- [ ] MAC Learning = Yes.
- [ ] MAC Limit Policy = Allow.
- [ ] LAN Segment áp dụng đúng profile.
- [ ] Unknown Unicast Flooding áp dụng theo network design.
- [ ] WAN Segment áp dụng profile phù hợp nếu là NSX-backed.

## OPNsense Primary

- [ ] WAN `61.14.236.213/26`.
- [ ] LAN `10.0.0.1/24`.
- [ ] PFSYNC `172.16.255.1/24`.
- [ ] LAN VIP `10.0.0.253`, VHID 2, AdvSkew 0.
- [ ] WAN VIP `61.14.236.218`, VHID 1, AdvSkew 0.
- [ ] pfsync peer `172.16.255.2`.
- [ ] XMLRPC target `172.16.255.2`.
- [ ] Disable preempt unchecked.

## OPNsense Backup

- [ ] WAN `61.14.236.217/26`.
- [ ] LAN `10.0.0.10/24`.
- [ ] PFSYNC `172.16.255.2/24`.
- [ ] LAN VIP `10.0.0.253`, VHID 2, AdvSkew 100.
- [ ] WAN VIP `61.14.236.218`, VHID 1, AdvSkew 100.
- [ ] pfsync peer `172.16.255.1`.
- [ ] Không cấu hình XMLRPC sync ngược.
- [ ] Disable preempt unchecked.

## Client

- [ ] Default gateway `10.0.0.253`.

## NAT

- [ ] Production Outbound NAT sử dụng WAN CARP VIP `61.14.236.218` khi upstream network đã support.
- [ ] NAT/rules được XMLRPC synchronize.

---

# 21. Ghi chú thiết kế

## Interface assignment

OPNsense yêu cầu interface assignments giữa hai node phải đồng nhất.

Ví dụ:

```text
LAN    = vmx0
WAN    = vmx1
PFSYNC = vmx2
```

trên cả hai firewall.

---

## PFSYNC security

PFSYNC có khả năng đồng bộ firewall state, do đó network này nên:

- Dedicated.
- Isolated.
- Không expose tới tenant workload.
- Không route tới Internet.

---

## CARP VHID

Mỗi CARP group trên cùng broadcast domain phải có VHID riêng.

Thiết kế hiện tại:

```text
WAN = VHID 1
LAN = VHID 2
```

---

## CARP virtual MAC

IPv4 CARP sử dụng format:

```text
00:00:5e:00:01:<VHID>
```

Do đó:

```text
VHID 1 → 00:00:5e:00:01:01
VHID 2 → 00:00:5e:00:01:02
```

VMware network phải cho phép các MAC này được học và forward đúng.

---

# 22. Tài liệu tham khảo chính thức

OPNsense – Configure CARP:

https://docs.opnsense.org/manual/how-tos/carp.html

Broadcom – Forged Transmits and MAC Address Changes:

https://knowledge.broadcom.com/external/article/427110

Broadcom – Configuring L2 Port Security on NSX-backed port groups:

https://knowledge.broadcom.com/external/article/394260

Broadcom – MAC Learning / port-level settings:

https://knowledge.broadcom.com/external/article/419625

Broadcom – External / Direct Network in VMware Cloud Director:

https://knowledge.broadcom.com/external/article/429841

---

**End of Runbook**
