# Cisco IOS-XE - Implementation & Operations Guide

## Daftar Isi

- Modul 1: Arsitektur Service-Provider, Platform & Licensing
- Modul 2: Underlay SP, OSPF + BFD + MPLS LDP + BGP Route Reflector
- Modul 3: MPLS L3VPN, VRF, RD / RT, PE-CE BGP / OSPF / Static
- Modul 4: MPLS L2VPN & EVPN untuk SP, VPWS, VPLS, EVPN DCI
- Modul 5: Internet Edge, ASBR, BGP Filtering, RPKI, Bogon, Anti Spoofing
- Modul 6: QoS Service-Provider, Gold / Silver / Bronze, MPLS EXP
- Modul 7: High Availability SP, Redundancy, ISSU, Fast Reroute
- Modul 8: Operasi SP, SLA, Monitoring, Telemetry, Automation, NOC Playbook
- Modul 9: STUDI KASUS ISP, Implementasi Full P / PE / RR / ASBR / CE


## Modul 1: Arsitektur Service-Provider, Platform & Licensing

### 1.1 Prinsip SP yang Berbeda dari Enterprise

Pada lingkungan Enterprise, semua router mengetahui VRF. Pada SP, hanya PE yang mengetahui VRF. P tidak diperkenankan mengetahui VRF. Hal tersebut memungkinkan SP mencapai skala ribuan customer.

| Prinsip SP | Artinya | Dampak Apabila Dilanggar |
|---|---|---|
| P hanya mengetahui Loopback dan Label | P hanya menjalankan OSPF + LDP | P overload, core tidak scalable |
| PE yang mengetahui VRF | PE yang memiliki RD, RT, VRF | Isolasi customer terjaga |
| RR yang mengetahui semua VPNv4 | RR melakukan reflect terhadap route, tidak melakukan forwarding traffic | PE tidak memerlukan full mesh |
| Customer tidak melihat Core | CE hanya melihat PE sebagai next-hop | Security dan simplicity |
| Internet di VRF terpisah | VRF INTERNET terpisah dari L3VPN | Internet flapping tidak mengganggu L3VPN |

### 1.2 Platform SP

| Platform | Role SP | Interface Khas | Cocok Untuk |
|---|---|---|---|
| Catalyst 9600 | P Core, RR, PE Core Besar | HundredGigE1/0/1 | Core 100G, RR, PE besar |
| Catalyst 9500 | P, PE Lite, PE Agregasi | TwentyFiveGigE1/0/1 | P 25G / 100G, PE agregasi |
| ASR 1000-X | PE Utama, ASBR, Border | TenGigabitEthernet0/0/0, GigabitEthernet0/0/0 | PE MPLS, Internet Edge, Agregasi |
| ISR 1100 / 4000 | CE, CPE Customer | GigabitEthernet0/0/0 | CE customer kecil |
| Catalyst 8200 | CE, CPE Customer Menengah | GigabitEthernet0/0/0 | CE customer menengah, branch |

Keputusan Principal: P gunakan 9500 / 9600. PE utama gunakan ASR 1000-X karena QFP dan skala VRF besar. CE gunakan ISR / 8200 karena ekonomis dan memadai untuk PE-CE.

### 1.3 Licensing SP

| Lisensi | Yang Diaktifkan |
|---|---|
| Network-Advantage | OSPF full, BGP full, MPLS LDP, MPLS L3VPN, MPLS L2VPN, EVPN, VRF, Multicast |

Periksa pada seluruh P, PE, RR, ASBR sebelum go-live. Satu device tanpa network-advantage dapat menyebabkan failure pada seluruh service customer di POP tersebut.

Aturan wajib SP: `license boot level network-advantage` tidak diperkenankan dilewatkan. Tanpa itu BGP, MPLS, EVPN tidak beroperasi.

```
license boot level network-advantage
write memory
reload
show license summary
```

### 1.4 Golden Image SP

Satu Golden Image per platform family. Contoh: seluruh ASR di 17.12.4, semua 9500 di 17.12.4. Tidak diperkenankan mencampur versi di satu POP. Perbedaan versi dapat menyebabkan perbedaan perilaku LDP dan BGP.

```
install add file flash:cat9k_iosxe.17.12.04.SPA.bin activate commit
install remove inactive
```

### 1.5 Safe Change SP

SP tidak diperkenankan down karena typo. Safety net wajib diterapkan.

```
archive
 path flash:archive-$h-$t
 maximum 14
 write-memory
```

Workflow:

```
copy running-config flash:pre-change-SP.cfg
reload in 10
configure terminal
! ... config ...
reload cancel
write memory
```

Rollback:

```
configure replace flash:pre-change-SP.cfg force
```

Larangan: Tidak diperkenankan `copy startup-config running-config` untuk rollback. Itu merge, bukan replace.


## Modul 2: Underlay SP, OSPF + BFD + MPLS LDP + BGP Route Reflector

### 2.1 Day-0 SP

Loopback0 adalah identitas. Semua BGP, OSPF Router-ID, LDP mengacu ke Loopback0. Management di Mgmt-vrf terpisah. MTU 9216 end to end di core untuk MPLS + EVPN.

```
hostname P-JKT-01
!
license boot level network-advantage
!
no ip domain-lookup
ip domain-name isp.local
ip name-server 10.0.0.53
!
service password-encryption
service timestamps log datetime msec localtime show-timezone
logging buffered 100000
logging host 10.0.0.100
!
ntp server 10.0.0.10 prefer
clock timezone WIB 7 0
!
aaa new-model
aaa authentication login default local
!
username netadmin privilege 15 secret STRONG-PASSWORD
!
ip ssh version 2
!
line vty 0 15
 transport input ssh
!
vrf definition Mgmt-vrf
 !
 address-family ipv4
 exit-address-family
!
interface GigabitEthernet0/0
 description MGMT OOB
 vrf forwarding Mgmt-vrf
 ip address 192.168.99.11 255.255.255.0
 no shutdown
!
interface Loopback0
 description ROUTER-ID / BGP / LDP
 ip address 10.0.0.21 255.255.255.255
!
archive
 path flash:archive-$h-$t
 maximum 14
 write-memory
```

### 2.2 OSPF Underlay SP

OSPF di SP hanya untuk Loopback dan link core. Tidak terdapat route customer di OSPF core.

| Decision | Pilihan SP Ini | Alasan |
|---|---|---|
| Area | Single Area 0 untuk Core | Simpel, troubleshooting cepat |
| Network Type | Point-to-Point semua link core | No DR/BDR, konvergensi cepat |
| Router-ID | Loopback0 manual | Stabil |
| Auth | MD5 di semua link | Mencegah rogue router |
| BFD | Aktif semua link core | Deteksi failure di bawah 1 detik |
| Timer | Hello 1 detik, Dead 3 detik | Konvergensi cepat |

Template P / PE Core Link:

```
router ospf 100
 router-id 10.0.0.21
 auto-cost reference-bandwidth 100000
 passive-interface default
 no passive-interface HundredGigE1/0/1
 no passive-interface Loopback0
 timers throttle spf 10 100 5000
!
interface Loopback0
 ip ospf 100 area 0
 ip ospf passive
!
interface HundredGigE1/0/1
 description TO P-SBY HundredGigE1/0/1 - 100G
 no switchport
 mtu 9216
 ip address 10.1.0.0 255.255.255.254
 ip ospf 100 area 0
 ip ospf network point-to-point
 ip ospf hello-interval 1
 ip ospf dead-interval 3
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 OSPF-KEY-SP
 ip ospf bfd
 bfd interval 300 min_rx 300 multiplier 3
 no shutdown
```

### 2.3 MPLS LDP

Semua P dan PE aktifkan MPLS LDP. P tidak perlu BGP.

```
mpls label protocol ldp
mpls ldp router-id Loopback0 force
mpls ldp discovery targeted-hello accept
!
interface HundredGigE1/0/1
 description CORE LINK
 no switchport
 mtu 9216
 ip address 10.1.0.0 255.255.255.254
 ip ospf 100 area 0
 mpls ip
 mpls label protocol ldp
 no shutdown
```

Verifikasi underlay:

```
show ip ospf neighbor
show ip route 10.0.0.11
show mpls ldp neighbor
show mpls forwarding-table
show bfd neighbors
```

P wajib dapat ping semua Loopback PE dan RR. P tidak diperkenankan mengetahui route customer.

### 2.4 BGP Route Reflector SP

Semua PE peer hanya ke RR. PE tidak peer ke PE langsung. Dua RR per region untuk redundansi.

Template RR, Catalyst 9600:

```
router bgp 64500
 bgp router-id 10.0.0.1
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 bgp cluster-id 10.0.0.1
 neighbor 10.0.0.11 remote-as 64500
 neighbor 10.0.0.11 description PE-JKT-01
 neighbor 10.0.0.11 update-source Loopback0
 neighbor 10.0.0.12 remote-as 64500
 neighbor 10.0.0.12 description PE-SBY-01
 neighbor 10.0.0.12 update-source Loopback0
 neighbor 10.0.0.2 remote-as 64500
 neighbor 10.0.0.2 description RR-SBY-01
 neighbor 10.0.0.2 update-source Loopback0
 !
 address-family vpnv4
 neighbor 10.0.0.11 activate
 neighbor 10.0.0.11 send-community extended
 neighbor 10.0.0.11 route-reflector-client
 neighbor 10.0.0.12 activate
 neighbor 10.0.0.12 send-community extended
 neighbor 10.0.0.12 route-reflector-client
 neighbor 10.0.0.2 activate
 neighbor 10.0.0.2 send-community extended
 exit-address-family
 !
 address-family l2vpn evpn
 neighbor 10.0.0.11 activate
 neighbor 10.0.0.11 send-community extended
 neighbor 10.0.0.11 route-reflector-client
 neighbor 10.0.0.12 activate
 neighbor 10.0.0.12 send-community extended
 neighbor 10.0.0.12 route-reflector-client
 neighbor 10.0.0.2 activate
 neighbor 10.0.0.2 send-community extended
 exit-address-family
```

Verifikasi RR:

```
show bgp vpnv4 unicast summary
show bgp l2vpn evpn summary
```

## Modul 3: MPLS L3VPN, VRF, RD / RT, PE-CE

### 3.1 Konsep L3VPN SP

Setiap customer memiliki VRF sendiri di PE. RD membuat route unik meski customer gunakan prefix yang identik, contoh dua customer identik-identik gunakan 192.168.1.0/24. RT mengontrol VRF mana diperkenankan berkomunikasi ke VRF mana.

### 3.2 VRF Definition Wajib IOS-XE

```
vrf definition BANK
 rd 64500:1001
 route-target export 64500:1001
 route-target import 64500:1001
 !
 address-family ipv4
 route-target export 64500:1001
 route-target import 64500:1001
 exit-address-family
!
vrf definition RETAIL
 rd 64500:1002
 route-target export 64500:1002
 route-target import 64500:1002
 !
 address-family ipv4
 route-target export 64500:1002
 route-target import 64500:1002
 exit-address-family
!
vrf definition INTERNET
 rd 64500:9000
 route-target export 64500:9000
 route-target import 64500:9000
 !
 address-family ipv4
 route-target export 64500:9000
 route-target import 64500:9000
 exit-address-family
!
vrf definition DC-EVPN
 rd 64500:1003
 route-target export 64500:1003
 route-target import 64500:1003
 !
 address-family ipv4
 route-target export 64500:1003
 route-target import 64500:1003
 exit-address-family
```

Selalu isi RT di level vrf definition dan di address-family ipv4. RD unik per customer per VRF.

### 3.3 Pola RT SP

| Pola | Contoh | Kapan Digunakan |
|---|---|---|
| Simetris | Export = Import = 64500:1001 | Customer biasa, site to site full mesh |
| Hub-Spoke | Hub export 64500:1001, import 64500:1002. Spoke sebaliknya | Retail, branch hanya ke pusat, tidak antar branch |
| Extranet | Bank export 64500:1001, Vendor import 64500:1001 | Bank mau berkomunikasi ke vendor tertentu |
| Internet | INTERNET RT 64500:9000 terpisah | Internet tidak diperkenankan leak ke L3VPN |

### 3.4 BGP PE L3VPN

Template PE, ASR 1000-X:

```
router bgp 64500
 bgp router-id 10.0.0.11
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 10.0.0.1 remote-as 64500
 neighbor 10.0.0.1 description RR-JKT-01
 neighbor 10.0.0.1 update-source Loopback0
 neighbor 10.0.0.2 remote-as 64500
 neighbor 10.0.0.2 description RR-SBY-01
 neighbor 10.0.0.2 update-source Loopback0
 !
 address-family vpnv4
 neighbor 10.0.0.1 activate
 neighbor 10.0.0.1 send-community extended
 neighbor 10.0.0.2 activate
 neighbor 10.0.0.2 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf BANK
 neighbor 192.168.255.1 remote-as 65001
 neighbor 192.168.255.1 description CE-BANK-JKT
 neighbor 192.168.255.1 activate
 neighbor 192.168.255.1 as-override
 neighbor 192.168.255.1 send-community both
 redistribute connected
 exit-address-family
 !
 address-family ipv4 vrf RETAIL
 redistribute connected
 redistribute static
 exit-address-family
 !
 address-family ipv4 vrf INTERNET
 redistribute connected
 redistribute static
 exit-address-family
```

Catatan: `as-override` wajib jika customer gunakan ASN yang identik di dua site, contoh Bank 65001 di JKT dan SBY. Tanpa as-override, BGP drop karena loop AS.

### 3.5 PE-CE, Tiga Opsi SP

**Opsi A: eBGP, untuk customer dengan ASN**

```
interface GigabitEthernet0/0/0
 description TO CE-BANK-JKT
 vrf forwarding BANK
 ip address 192.168.255.2 255.255.255.252
 no shutdown
!
router bgp 64500
 address-family ipv4 vrf BANK
 neighbor 192.168.255.1 remote-as 65001
 neighbor 192.168.255.1 activate
 neighbor 192.168.255.1 as-override
 exit-address-family
```

**Opsi B: OSPF, untuk customer dengan OSPF internal**

```
router ospf 200 vrf BANK
 router-id 192.168.255.2
 redistribute bgp 64500 subnets
!
interface GigabitEthernet0/0/1
 description TO CE-BANK-OSPF
 vrf forwarding BANK
 ip address 192.168.255.6 255.255.255.252
 ip ospf 200 area 0
 no shutdown
```

**Opsi C: Static, untuk customer kecil**

```
interface GigabitEthernet0/0/2
 description TO CE-RETAIL-JKT
 vrf forwarding RETAIL
 ip address 192.168.255.6 255.255.255.252
 no shutdown
!
ip route vrf RETAIL 172.17.10.0 255.255.255.0 192.168.255.5
```

Penting: Setelah `vrf forwarding`, IP lama hilang. Wajib diketik ulang `ip address`.

### 3.6 Verifikasi L3VPN

```
show ip ospf neighbor
show mpls ldp neighbor
show bgp vpnv4 unicast summary
show bgp vpnv4 unicast vrf BANK summary
show ip route vrf BANK
show ip vrf detail BANK
ping vrf BANK 172.16.20.1 source 172.16.10.1
ping vrf RETAIL 172.17.20.1 source 172.17.10.1 size 9000 df-bit
ping vrf BANK 172.17.20.1
```

Ping terakhir wajib gagal, itu bukti isolasi. Jika ping antar VRF sukses, RT leak.

## Modul 4: MPLS L2VPN & EVPN untuk SP, VPWS, VPLS, EVPN DCI

### 4.1 Kapan Menggunakan L2VPN, Kapan L3VPN

| Service | Gunakan | Contoh |
|---|---|---|
| L3VPN | Customer butuh routing antar site | Bank, Retail, kantor branch |
| L2VPN VPWS | Customer butuh satu link point to point L2 | Cluster database, link leased line virtual |
| VPLS | Customer butuh multipoint L2 | Customer dengan 3 site satu VLAN |
| EVPN | Customer butuh L2 + L3 modern, DCI | Data Center Interconnect, Cloud |

### 4.2 VPWS, Point to Point L2

VPWS adalah pseudowire MPLS point to point. Customer merasa seperti kabel langsung.

Pada PE JKT-01, ASR 1000-X:

```
pseudowire-class BANK-PW
 encapsulation mpls
!
interface GigabitEthernet0/0/3
 description TO CE-BANK-L2-JKT
 no switchport
 xconnect 10.0.0.12 1001 encapsulation mpls pw-class BANK-PW
 no shutdown
```

Pada PE SBY-01:

```
pseudowire-class BANK-PW
 encapsulation mpls
!
interface GigabitEthernet0/0/3
 description TO CE-BANK-L2-SBY
 no switchport
 xconnect 10.0.0.11 1001 encapsulation mpls pw-class BANK-PW
 no shutdown
```

Verifikasi VPWS:

```
show xconnect all
show mpls l2transport vc 1001 detail
show mpls l2transport vc summary
```

Jika VPWS down, periksa: LDP neighbor up, MTU 9216, VC ID identik di kedua PE, peer IP adalah Loopback0 PE lawan.

### 4.3 VPLS, Multipoint L2

VPLS untuk customer dengan lebih dari 2 site satu broadcast domain. Pada IOS-XE, VPLS menggunakan VFI.

```
l2vpn vfi context RETAIL-VPLS
 vpn id 2002
 member 10.0.0.12 encapsulation mpls
 member 10.0.0.13 encapsulation mpls
!
bridge-domain 200
 member vfi RETAIL-VPLS
 member GigabitEthernet0/0/4 service-instance 200
!
interface GigabitEthernet0/0/4
 description TO CE-RETAIL-VPLS
 no switchport
 service instance 200 ethernet
 encapsulation dot1q 200
 rewrite ingress tag pop 1 symmetric
 bridge-domain 200
 no shutdown
```

Catatan SP: VPLS tidak efisien karena flood. Jika customer baru, direkomendasikan EVPN dulu, bukan VPLS. VPLS hanya untuk customer legacy yang belum dapat migrasi.

### 4.4 EVPN untuk SP, DCI dan L2 Modern

EVPN adalah pengganti VPLS yang modern. Control plane BGP EVPN, data plane VXLAN atau MPLS.

Arsitektur EVPN SP:

```
[PE JKT - VTEP] --- [P Core MPLS] --- [PE SBY - VTEP]
 VNI 10300, VRF DC-EVPN BGP EVPN VNI 10300, VRF DC-EVPN
```

Mapping wajib konsisten:

| VLAN | VNI L2 | VRF | VNI L3 | Subnet |
|---|---|---|---|---|
| 300 | 10300 | DC-EVPN | 50300 | 192.168.30.0/24 |
| 100 | 10100 | BANK | 50100 | 192.168.100.0/24 |

Template PE EVPN, ASR 1000-X / 9500:

```
interface Loopback1
 description VTEP EVPN SOURCE
 ip address 10.0.1.11 255.255.255.255
 ip ospf 100 area 0
!
vrf definition DC-EVPN
 rd 64500:1003
 route-target export 64500:1003
 route-target import 64500:1003
 !
 address-family ipv4
 route-target export 64500:1003
 route-target import 64500:1003
 exit-address-family
!
router bgp 64500
 bgp router-id 10.0.0.11
 no bgp default ipv4-unicast
 neighbor 10.0.0.1 remote-as 64500
 neighbor 10.0.0.1 update-source Loopback0
 neighbor 10.0.0.2 remote-as 64500
 neighbor 10.0.0.2 update-source Loopback0
 !
 address-family l2vpn evpn
 neighbor 10.0.0.1 activate
 neighbor 10.0.0.1 send-community extended
 neighbor 10.0.0.2 activate
 neighbor 10.0.0.2 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf DC-EVPN
 advertise l2vpn evpn
 redistribute connected
 exit-address-family
!
l2vpn evpn
 replication-type ingress
 router-id Loopback1
!
l2vpn evpn instance 300 vlan-based
 encapsulation vxlan
 replication-type ingress
!
interface nve1
 no ip address
 source-interface Loopback1
 host-reachability protocol bgp
 member vni 10300 vrf DC-EVPN
 member vni 50300 vrf DC-EVPN
 no shutdown
!
interface Vlan300
 description DC-EVPN SVI - ANYCAST GATEWAY
 vrf forwarding DC-EVPN
 ip address 192.168.30.1 255.255.255.0
 ip virtual-router address 192.168.30.1
 no shutdown
```

`ip virtual-router address` adalah Anycast Gateway. Semua PE gunakan IP gateway identik. VM bermigrasi POP tanpa ganti gateway.

Route Type EVPN:

| Route Type | Nama | Fungsi |
|---|---|---|
| RT-2 | MAC / IP Advertisement | MAC endpoint dan IP-nya |
| RT-3 | Inclusive Multicast | BUM traffic, ARP |
| RT-5 | IP Prefix Route | Subnet VRF antar PE |

Verifikasi EVPN SP:

```
show bgp l2vpn evpn summary
show bgp l2vpn evpn neighbors 10.0.0.1 routes
show nve vni
show nve peers
show l2vpn evpn evi 300 detail
show bgp l2vpn evpn route-type 2
show bgp l2vpn evpn route-type 5
show ip route vrf DC-EVPN
ping vrf DC-EVPN 192.168.30.10 source 192.168.30.1
ping vrf DC-EVPN 192.168.30.20 size 9000 df-bit
```

---

## Modul 5: Internet Edge, ASBR, BGP Filtering, RPKI, Bogon

### 5.1 Arsitektur Internet Edge SP

Internet tidak diperkenankan tercampur dengan L3VPN di routing global. Internet di VRF INTERNET. PE ke ASBR gunakan VPNv4. ASBR ke Upstream gunakan eBGP global.

```
[PE JKT] --- VPNv4 VRF INTERNET --- [ASBR JKT] --- eBGP --- [Upstream A / B]
 |
 [IX / IIX]
```

### 5.2 Template ASBR, ASR 1000-X

```
hostname ASBR-JKT-01
!
vrf definition INTERNET
 rd 64500:9000
 route-target export 64500:9000
 route-target import 64500:9000
 !
 address-family ipv4
 route-target export 64500:9000
 route-target import 64500:9000
 exit-address-family
!
interface Loopback0
 ip address 10.0.0.31 255.255.255.255
 ip ospf 100 area 0
!
router bgp 64500
 bgp router-id 10.0.0.31
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 10.0.0.1 remote-as 64500
 neighbor 10.0.0.1 description RR-JKT-01
 neighbor 10.0.0.1 update-source Loopback0
 neighbor 203.0.113.1 remote-as 135000
 neighbor 203.0.113.1 description UPSTREAM-A
 neighbor 198.51.100.1 remote-as 135001
 neighbor 198.51.100.1 description UPSTREAM-B
 !
 address-family vpnv4
 neighbor 10.0.0.1 activate
 neighbor 10.0.0.1 send-community extended
 exit-address-family
 !
 address-family ipv4 unicast
 neighbor 203.0.113.1 activate
 neighbor 203.0.113.1 send-community
 neighbor 203.0.113.1 route-map UPSTREAM-IN in
 neighbor 203.0.113.1 route-map UPSTREAM-OUT out
 neighbor 198.51.100.1 activate
 neighbor 198.51.100.1 send-community
 neighbor 198.51.100.1 route-map UPSTREAM-IN in
 neighbor 198.51.100.1 route-map UPSTREAM-OUT out
 network 203.0.113.0 mask 255.255.255.0
 exit-address-family
 !
 address-family ipv4 vrf INTERNET
 redistribute connected
 redistribute static
 exit-address-family
!
interface TenGigabitEthernet0/0/0
 description TO UPSTREAM-A
 ip address 203.0.113.2 255.255.255.252
 no shutdown
!
interface TenGigabitEthernet0/0/1
 description TO UPSTREAM-B
 ip address 198.51.100.2 255.255.255.252
 no shutdown
!
interface TenGigabitEthernet0/0/2
 description TO P-JKT - CORE
 mtu 9216
 ip address 10.1.0.6 255.255.255.254
 ip ospf 100 area 0
 mpls ip
 no shutdown
```

### 5.3 Filtering Wajib, Tidak Menerima Full Table Tanpa Filter

```
ip prefix-list BOGON deny 10.0.0.0/8 le 32
ip prefix-list BOGON deny 172.16.0.0/12 le 32
ip prefix-list BOGON deny 192.168.0.0/16 le 32
ip prefix-list BOGON deny 100.64.0.0/10 le 32
ip prefix-list BOGON deny 0.0.0.0/0 le 32
ip prefix-list SP-PREFIX permit 203.0.113.0/24
ip prefix-list DEFAULT permit 0.0.0.0/0
!
route-map UPSTREAM-IN deny 10
 match ip address prefix-list BOGON
!
route-map UPSTREAM-IN permit 20
 match ip address prefix-list DEFAULT
 set local-preference 200
!
route-map UPSTREAM-IN permit 30
 description Menerima route lain dengan validasi RPKI di produksi
 set local-preference 100
!
route-map UPSTREAM-OUT permit 10
 match ip address prefix-list SP-PREFIX
 set as-path prepend 64500 64500
```

Pada produksi, RPKI / ROA validation diaktifkan. Tidak diperkenankan menerima prefix yang invalid. Gunakan `bgp validation` dan route-map yang drop invalid. Ini mencegah hijack.

### 5.4 Anti Spoofing dan uRPF

Pada PE-CE dan ASBR, uRPF diaktifkan untuk mencegah spoofing.

```
interface GigabitEthernet0/0/0
 description TO CE-BANK
 vrf forwarding BANK
 ip address 192.168.255.2 255.255.255.252
 ip verify unicast source reachable-via rx
 no shutdown
!
interface TenGigabitEthernet0/0/0
 description TO UPSTREAM-A
 ip address 203.0.113.2 255.255.255.252
 ip verify unicast source reachable-via any
 no shutdown
```

### 5.5 NAT / CGNAT untuk Customer Internet

Customer Internet kecil sering gunakan CGNAT. PE atau ASBR yang NAT, bukan CE.

```
ip nat pool CGNAT-POOL 100.64.0.1 100.64.0.254 prefix-length 24
ip nat inside source list CGNAT-ACL pool CGNAT-POOL overload
!
ip access-list standard CGNAT-ACL
 permit 192.168.0.0 0.0.255.255
!
interface GigabitEthernet0/0/1
 description TO CE-RETAIL-INTERNET
 vrf forwarding INTERNET
 ip address 192.168.255.6 255.255.255.252
 ip nat inside
 no shutdown
!
interface TenGigabitEthernet0/0/0
 description TO UPSTREAM
 ip address 203.0.113.2 255.255.255.252
 ip nat outside
 no shutdown
```

Verifikasi Internet:

```
show bgp ipv4 unicast summary
show bgp ipv4 unicast neighbors 203.0.113.1 routes
show ip route vrf INTERNET
ping vrf INTERNET 8.8.8.8 source 203.0.113.10
show ip nat translations
show ip bgp validation
```

---

## Modul 6: QoS Service-Provider, Gold / Silver / Bronze

### 6.1 Prinsip QoS SP

SP tidak dapat jual QoS asal-asalan. QoS SP wajib dapat diukur, dipolicer di PE, dan dijamin di core.

- Policer di PE ingress, per customer, per kelas. Ini yang membedakan Gold dan Bronze.
- Core trust MPLS EXP, tidak re-mark sembarangan.
- Queuing di egress semua link core dan PE-CE.
- Shaping di PE-CE sesuai bandwidth yang dibeli customer, bukan bandwidth fisik.

### 6.2 Model QoS SP

| Kelas SP | DSCP Customer | MPLS EXP | Bandwidth Core | Contoh |
|---|---|---|---|---|
| Gold | EF, AF41 | 5 | 30 persen | Voice Bank, Transaksi |
| Silver | AF31, AF21 | 3 | 40 persen | ERP, CRM, Core App |
| Bronze | AF11, CS1, 0 | 0, 1 | 30 persen | Internet, Backup, Best Effort |

### 6.3 Class-Map SP

```
class-map match-any GOLD
 match dscp ef af41
 match mpls experimental topmost 5
class-map match-any SILVER
 match dscp af31 af32 af21 af22
 match mpls experimental topmost 3
class-map match-any BRONZE
 match dscp af11 cs1 0
 match mpls experimental topmost 0 1
class-map match-any VOICE
 match dscp ef
class-map match-any CRITICAL-DATA
 match dscp af31
```

### 6.4 Policer PE Ingress, Per Customer

Ini template yang membedakan SP profesional dan amatir. Customer Bank Gold 10 Mbps voice, 50 Mbps critical.

```
policy-map SP-PE-INGRESS-BANK-GOLD
 class VOICE
 police 10000000 conform-action transmit exceed-action drop
 set mpls experimental imposition 5
 class CRITICAL-DATA
 police 50000000 conform-action transmit exceed-action drop
 set mpls experimental imposition 3
 class class-default
 police 20000000 conform-action transmit exceed-action drop
 set mpls experimental imposition 0
!
interface GigabitEthernet0/0/0
 description TO CE-BANK-JKT - GOLD 60M
 vrf forwarding BANK
 ip address 192.168.255.2 255.255.255.252
 service-policy input SP-PE-INGRESS-BANK-GOLD
 no shutdown
```

Customer Retail Silver:

```
policy-map SP-PE-INGRESS-RETAIL-SILVER
 class CRITICAL-DATA
 police 30000000 conform-action transmit exceed-action drop
 set mpls experimental imposition 3
 class class-default
 police 20000000 conform-action transmit exceed-action drop
 set mpls experimental imposition 0
!
interface GigabitEthernet0/0/1
 description TO CE-RETAIL-JKT - SILVER 50M
 vrf forwarding RETAIL
 ip address 192.168.255.6 255.255.255.252
 service-policy input SP-PE-INGRESS-RETAIL-SILVER
 no shutdown
```

### 6.5 Egress Core SP

```
policy-map SP-CORE-EGRESS-100G
 class GOLD
 priority percent 30
 class SILVER
 bandwidth percent 40
 random-detect dscp-based
 class BRONZE
 bandwidth percent 20
 class class-default
 bandwidth percent 10
 random-detect dscp-based
!
interface HundredGigE1/0/1
 description TO P-SBY - CORE 100G
 no switchport
 mtu 9216
 ip address 10.1.0.0 255.255.255.254
 service-policy output SP-CORE-EGRESS-100G
 no shutdown
```

### 6.6 Shaping PE-CE

Shaping wajib sesuai bandwidth yang dibeli, bukan bandwidth fisik. Jika customer beli 50 Mbps di link 1G, shaping 50 Mbps.

```
policy-map SP-PE-CE-SHAPE-50M
 class class-default
 shape average 50000000
 service-policy SP-PE-EGRESS-50M
!
policy-map SP-PE-EGRESS-50M
 class GOLD
 priority percent 30
 class SILVER
 bandwidth percent 40
 class BRONZE
 bandwidth percent 30
!
interface GigabitEthernet0/0/0
 description TO CE-BANK-JKT
 vrf forwarding BANK
 ip address 192.168.255.2 255.255.255.252
 service-policy output SP-PE-CE-SHAPE-50M
 no shutdown
```

### 6.7 Verifikasi QoS SP

```
show policy-map interface GigabitEthernet0/0/0 input
show policy-map interface HundredGigE1/0/1 output
show mls qos interface HundredGigE1/0/1 statistics
show platform hardware fed switch 1 qos queue stats interface HundredGigE1/0/1
```

Drop di GOLD wajib 0. Jika drop, policer terlalu kecil atau priority percent terlalu kecil. Drop di BRONZE tinggi itu by design. Jika customer komplain voice putus, periksa policer di PE ingress dulu. 90 persen kasus QoS SP adalah policer terlalu kecil, bukan core congestion.

## Modul 7: High Availability SP, Redundancy, ISSU, Fast Reroute

### 7.1 Prinsip HA SP

SP tidak diperkenankan down karena satu device mati. Semua wajib redundant.

| Layer | Redundansi | Target |
|---|---|---|
| Core | Dua P per POP, dua link Inter-POP | Satu P mati, traffic bermigrasi otomatis |
| PE | Dua PE per POP besar | Satu PE mati, CE failover |
| RR | Dua RR, JKT dan SBY | Satu RR mati, BGP tetap beroperasi |
| Upstream | Dua Upstream, dua ASBR | Satu Upstream mati, Internet tetap beroperasi |
| Power | Dual PSU, dual feed | Satu PSU mati, device tetap beroperasi |
| CE | Dua CE untuk customer Gold | Satu CE mati, customer tetap beroperasi |

### 7.2 StackWise Virtual untuk P dan PE Agregasi

StackWise Virtual membuat dua fisik 9500 / 9600 jadi satu logical switch. Control plane aktif-standby, data plane aktif-aktif.

```
stackwise-virtual
 domain 10
!
interface TwentyFiveGigE1/0/47
 description SVL LINK TO PEER
 stackwise-virtual link 1
 no shutdown
!
interface TwentyFiveGigE1/0/48
 description SVL LINK TO PEER
 stackwise-virtual link 1
 no shutdown
```

Verifikasi: `show stackwise-virtual`, `show stackwise-virtual dual-active-detection`, `show switch`, `show redundancy`. Dual Active Detection wajib. Tanpa DAD, split brain dapat menghasilkan loop.

### 7.3 ISSU

ISSU hanya beroperasi jika dua supervisor atau StackWise Virtual aktif, versi lama dan baru support ISSU path, lisensi identik.

```
install add file flash:cat9k_iosxe.17.15.01.SPA.bin activate commit
show install summary
show redundancy
```

Jika ISSU gagal, periksa `show install log`, rollback dengan `install abort`.

### 7.4 MPLS Fast Reroute dan LFA

Pada SP, konvergensi OSPF 3 detik masih terlalu lama untuk Gold. LFA dan MPLS FRR diaktifkan.

```
router ospf 100
 fast-reroute per-prefix enable area 0 prefix-priority high
 fast-reroute per-prefix tie-break srlg-disjoint index 10
!
interface HundredGigE1/0/1
 description CORE LINK
 ip ospf 100 area 0
 mpls ip
 mpls traffic-eng tunnels
 no shutdown
!
mpls traffic-eng reoptimize timers frequency 0
```

Untuk FRR yang lebih advance, gunakan RSVP-TE. Pada IOS-XE, RSVP-TE tersedia di ASR 1000-X dan 9500 / 9600 dengan network-advantage.

Verifikasi FRR:

```
show ip ospf fast-reroute
show mpls traffic-eng fast-reroute database
show ip route 10.0.0.12 repair-paths
```

### 7.5 First Hop Redundancy untuk CE

Customer Gold wajib dual CE dengan HSRP / VRRP.

```
interface GigabitEthernet0/0/1
 description LAN BANK JKT
 ip address 172.16.10.2 255.255.255.0
 standby 10 ip 172.16.10.1
 standby 10 priority 110
 standby 10 preempt
 no shutdown
```


## Modul 8: Operasi SP, SLA, Monitoring, Telemetry, Automation, NOC Playbook

### 8.1 SLA SP

| Metrik | Target SP | Cara Ukur |
|---|---|---|
| Availability | 99.95 persen per bulan | Telemetry, interface uptime |
| Latency JKT-SBY | Di bawah 15 ms | IP SLA per VRF |
| Latency JKT-BDG | Di bawah 8 ms | IP SLA per VRF |
| Packet Loss | Di bawah 0.1 persen | IP SLA, QoS drop |
| MTTR P1 | 4 jam | NOC ticket |
| Tambah Customer Baru | 1 hari, config hanya di 2 PE | Provisioning time |

### 8.2 IP SLA Per Customer

```
ip sla 100
 udp-jitter 172.16.20.1 1000 source-ip 172.16.10.1 source-port 1000 vrf BANK
 frequency 60
!
ip sla schedule 100 life forever start-time now
!
track 100 ip sla 100 reachability
```

### 8.3 Telemetry SP

SNMP masih dipakai, namun untuk Core modern gunakan Model-Driven Telemetry.

```
telemetry ietf subscription 101
 encoding encode-kvgpb
 filter xpath /interfaces/interface/state/counters
 source-address 10.0.0.11
 stream yang-push
 update-policy periodic 1000
 receiver ip 10.0.0.100 57000 protocol grpc-tcp
```

Monitor per VRF, per customer: interface, BGP, OSPF, MPLS LDP, QoS drop, NetFlow.

NetFlow untuk billing dan deteksi anomaly:

```
flow record SP-FLOW
 match ipv4 source address
 match ipv4 destination address
 match transport source-port
 match transport destination-port
 collect counter bytes long
 collect counter packets long
!
flow exporter SP-EXPORTER
 destination 10.0.0.100
 transport udp 2055
!
flow monitor SP-MONITOR
 record SP-FLOW
 exporter SP-EXPORTER
!
interface GigabitEthernet0/0/0
 description TO CE-BANK
 vrf forwarding BANK
 ip flow monitor SP-MONITOR input
 ip flow monitor SP-MONITOR output
 no shutdown
```

### 8.4 Automation SP

Tidak diperkenankan SSH dan paste config manual untuk 100 PE. Gunakan template per role.

Struktur:

```
inventory/
 group_vars/
 pe.yml
 p.yml
 rr.yml
templates/
 pe-asr.j2
 p-9500.j2
playbooks/
 deploy-pe.yml
 add-customer.yml
```

Tambah customer baru: buat VRF di 2 PE saja, tidak perlu ubah P atau RR. Jika tambah customer butuh ubah P, desain salah.

### 8.5 NOC Playbook, Troubleshooting 5 Langkah

Gunakan urutan ini setiap insiden:

1. Fisik dan Link: `show interfaces status`, `show interfaces counters errors`, `show log`
2. Underlay: `show ip ospf neighbor`, `show ip route`, `show bfd neighbors`
3. MPLS / EVPN: `show mpls ldp neighbor`, `show bgp vpnv4 unicast summary`, `show bgp l2vpn evpn summary`, `show nve peers`
4. VRF / Policy: `show ip route vrf`, `show ip vrf detail`, `show policy-map interface`
5. Endpoint: `show mac address-table`, `show ip arp vrf`, `ping vrf`, `traceroute vrf`

Dokumentasikan setiap insiden: gejala, perintah yang dipakai, root cause, fix, pencegahan.

### 8.6 Security SP

- uRPF di PE-CE dan ASBR, sudah di Modul 5.
- BGP TTL Security di PE-CE dan RR: `neighbor 192.168.255.1 ttl-security hops 1`
- MD5 di OSPF dan BGP: `neighbor 10.0.0.1 password BGP-KEY`
- Control Plane Policing: Batasi traffic ke CPU. Tidak diperkenankan sampai DDoS ke PE menghasilkan CPU 100 persen.
- Bogon dan RPKI di ASBR, sudah di Modul 5.

```
control-plane
 service-policy input COPP-SP
!
policy-map COPP-SP
 class BGP
 police 10000000 conform-action transmit exceed-action drop
 class OSPF
 police 5000000 conform-action transmit exceed-action drop
 class class-default
 police 1000000 conform-action transmit exceed-action drop
```

## Modul 9: STUDI KASUS ISP, Implementasi Full P / PE / RR / ASBR / CE

### 9.1 Gambaran Bisnis dan Tantangan

ISP adalah ISP regional di Jawa. Sebelum MPLS, mereka gunakan OSPF + VRF-Lite + GRE per customer. Masalahnya:

- Tambah customer baru butuh config di semua router. 2 minggu per customer.
- Satu customer leak route ke customer lain karena RT manual salah.
- Internet dan L3VPN tercampur di satu routing table. Satu flapping menghasilkan semua goyang.
- QoS tidak konsisten. Voice customer bank sering terdegradasi dengan traffic video.
- Tidak dapat jual L2VPN untuk customer yang butuh VLAN stretch antar branch.

Target setelah MPLS:

- Tambah customer baru dalam 1 hari, config hanya di 2 PE.
- Isolasi VRF 100 persen, tidak terdapat leak.
- Jual 4 service: MPLS L3VPN, MPLS L2VPN VPWS, Internet Dedicated, EVPN Data Center Interconnect.
- SLA: Latency Jakarta-Surabaya di bawah 15 ms, Packet Loss di bawah 0.1 persen, Availability 99.95 persen.
- QoS SP yang dapat dijual per kelas: Gold, Silver, Bronze.

### 9.2 Topologi SP

```
 [Upstream A] [Upstream B] [IX / IIX]
 \ / /
 [ASBR JKT - ASR 1000-X]
 |
 [RR JKT - 9600] [P JKT - 9500] [PE JKT-01 - ASR 1000-X] --- [CE Bank JKT - 8200]
 | | | [PE JKT-02 - 9500] --- [CE Retail JKT - ISR 4000]
 | | |
 100G HundredGigE Inter-POP
 | | |
 [P SBY - 9500] [PE SBY-01 - ASR 1000-X] --- [CE Bank SBY - 8200]
 [RR SBY - 9600] [PE SBY-02 - 9500] --- [CE DC SBY - 9300 EVPN]
 | |
 [P BDG - 9500] [PE BDG-01 - ASR 1000-X] --- [CE Retail BDG - ISR 4000]
```

Peran device:

| Device | Platform | Role SP | ASN |
|---|---|---|---|
| P JKT / SBY / BDG | Catalyst 9500 / 9600 | P, MPLS core, LDP only, tidak mengetahui VRF | 64500 |
| PE JKT-01 / SBY-01 / BDG-01 | ASR 1000-X | PE, L3VPN, L2VPN, Internet, BGP | 64500 |
| PE JKT-02 / SBY-02 | Catalyst 9500 | PE Lite, L3VPN, EVPN Leaf | 64500 |
| RR JKT / RR SBY | Catalyst 9600 | Route Reflector VPNv4 + EVPN | 64500 |
| ASBR JKT | ASR 1000-X | Peering ke Upstream, Internet, Inter-AS | 64500 |
| CE Bank | Catalyst 8200 / ISR 4000 | CE, BGP / OSPF / Static ke PE | 65001 |
| CE Retail | ISR 4000 / 8200 | CE, BGP / Static | 65002 |
| CE DC | Catalyst 9300 | CE EVPN, VXLAN | 65003 |

Prinsip SP: P tidak mengetahui VRF. PE yang mengetahui VRF. RR yang mengetahui semua route VPNv4. Customer tidak melihat core.

### 9.3 IP Addressing Plan SP

| Fungsi | Prefix | Contoh |
|---|---|---|
| Loopback0 Core / PE / RR / P | 10.0.0.0/24, /32 per device | P JKT 10.0.0.21, PE JKT-01 10.0.0.11, RR JKT 10.0.0.1 |
| Loopback1 VTEP EVPN | 10.0.1.0/24, /32 per PE / Leaf | PE SBY-02 10.0.1.12 |
| P2P Core | 10.1.0.0/16, /31 per link | P JKT ke P SBY 10.1.0.0/31 |
| PE-CE | 192.168.255.0/24, /30 per customer | Bank JKT 192.168.255.0/30 |
| Customer LAN Bank | 172.16.0.0/16 | Bank JKT 172.16.10.0/24, Bank SBY 172.16.20.0/24 |
| Customer LAN Retail | 172.17.0.0/16 | Retail JKT 172.17.10.0/24, Retail BDG 172.17.30.0/24 |
| Internet VRF | 203.0.113.0/24, Public Pool | Pool customer 203.0.113.0/24 |
| Management | 192.168.99.0/24, Mgmt-vrf | NOC 192.168.99.100 |

ASN Plan:

| ASN | Pemilik | Keterangan |
|---|---|---|
| 64500 | ISP | ASN SP, iBGP core |
| 65001 | Bank | Customer L3VPN, eBGP ke PE |
| 65002 | Retail Jaya | Customer L3VPN + Internet |
| 65003 | DC SBY | Customer EVPN |
| 135000 | Upstream A | eBGP Internet |
| 135001 | Upstream B | eBGP Internet |

### 9.4 Implementasi Lengkap, Config Produksi SP

#### 9.4.1 P Router, Catalyst 9500, P JKT

P paling simpel. Tidak terdapat VRF customer, tidak terdapat BGP customer. Hanya OSPF + MPLS + QoS core.

```
hostname P-JKT-01
!
license boot level network-advantage
!
interface Loopback0
 description P ROUTER-ID / LDP
 ip address 10.0.0.21 255.255.255.255
 ip ospf 100 area 0
!
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
router ospf 100
 router-id 10.0.0.21
 auto-cost reference-bandwidth 100000
 passive-interface default
 no passive-interface HundredGigE1/0/1
 no passive-interface HundredGigE1/0/2
 no passive-interface HundredGigE1/0/3
!
interface HundredGigE1/0/1
 description TO P-SBY HundredGigE1/0/1 - 100G INTER-POP
 no switchport
 mtu 9216
 ip address 10.1.0.0 255.255.255.254
 ip ospf 100 area 0
 ip ospf network point-to-point
 ip ospf hello-interval 1
 ip ospf dead-interval 3
 ip ospf bfd
 mpls ip
 mpls label protocol ldp
 service-policy output SP-CORE-EGRESS-100G
 no shutdown
!
interface HundredGigE1/0/2
 description TO PE-JKT-01 TenGigabitEthernet0/0/0
 no switchport
 mtu 9216
 ip address 10.1.0.2 255.255.255.254
 ip ospf 100 area 0
 ip ospf network point-to-point
 mpls ip
 no shutdown
!
interface HundredGigE1/0/3
 description TO RR-JKT HundredGigE1/0/1
 no switchport
 mtu 9216
 ip address 10.1.0.4 255.255.255.254
 ip ospf 100 area 0
 ip ospf network point-to-point
 mpls ip
 no shutdown
!
archive
 path flash:archive-$h-$t
 maximum 14
 write-memory
```

Verifikasi P: `show ip ospf neighbor`, `show mpls ldp neighbor`, `show mpls forwarding-table`, `show ip route 10.0.0.11`. P wajib dapat ping semua Loopback PE dan RR, namun tidak diperkenankan mengetahui route customer. Periksa `show ip route vrf` di P wajib kosong.

#### 9.4.2 PE Utama, ASR 1000-X, PE JKT-01

PE paling lengkap. Ada 3 customer + Internet.

```
hostname PE-JKT-01
!
license boot level network-advantage
!
interface Loopback0
 description PE ROUTER-ID / BGP / LDP
 ip address 10.0.0.11 255.255.255.255
 ip ospf 100 area 0
!
interface Loopback1
 description VTEP EVPN
 ip address 10.0.1.11 255.255.255.255
 ip ospf 100 area 0
!
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
vrf definition BANK
 rd 64500:1001
 route-target export 64500:1001
 route-target import 64500:1001
 !
 address-family ipv4
 route-target export 64500:1001
 route-target import 64500:1001
 exit-address-family
!
vrf definition RETAIL
 rd 64500:1002
 route-target export 64500:1002
 route-target import 64500:1002
 !
 address-family ipv4
 route-target export 64500:1002
 route-target import 64500:1002
 exit-address-family
!
vrf definition INTERNET
 rd 64500:9000
 route-target export 64500:9000
 route-target import 64500:9000
 !
 address-family ipv4
 route-target export 64500:9000
 route-target import 64500:9000
 exit-address-family
!
vrf definition DC-EVPN
 rd 64500:1003
 route-target export 64500:1003
 route-target import 64500:1003
 !
 address-family ipv4
 route-target export 64500:1003
 route-target import 64500:1003
 exit-address-family
!
router ospf 100
 router-id 10.0.0.11
 passive-interface default
 no passive-interface TenGigabitEthernet0/0/0
 no passive-interface Loopback0
 no passive-interface Loopback1
!
router bgp 64500
 bgp router-id 10.0.0.11
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 10.0.0.1 remote-as 64500
 neighbor 10.0.0.1 description RR-JKT-01
 neighbor 10.0.0.1 update-source Loopback0
 neighbor 10.0.0.2 remote-as 64500
 neighbor 10.0.0.2 description RR-SBY-01
 neighbor 10.0.0.2 update-source Loopback0
 !
 address-family vpnv4
 neighbor 10.0.0.1 activate
 neighbor 10.0.0.1 send-community extended
 neighbor 10.0.0.2 activate
 neighbor 10.0.0.2 send-community extended
 exit-address-family
 !
 address-family l2vpn evpn
 neighbor 10.0.0.1 activate
 neighbor 10.0.0.1 send-community extended
 neighbor 10.0.0.2 activate
 neighbor 10.0.0.2 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf BANK
 neighbor 192.168.255.1 remote-as 65001
 neighbor 192.168.255.1 description CE-BANK-JKT
 neighbor 192.168.255.1 activate
 neighbor 192.168.255.1 as-override
 neighbor 192.168.255.1 send-community both
 redistribute connected
 exit-address-family
 !
 address-family ipv4 vrf RETAIL
 redistribute connected
 redistribute static
 exit-address-family
 !
 address-family ipv4 vrf INTERNET
 redistribute connected
 redistribute static
 exit-address-family
 !
 address-family ipv4 vrf DC-EVPN
 advertise l2vpn evpn
 redistribute connected
 exit-address-family
!
l2vpn evpn
 replication-type ingress
 router-id Loopback1
!
interface nve1
 no ip address
 source-interface Loopback1
 host-reachability protocol bgp
 member vni 10300 vrf DC-EVPN
 member vni 50300 vrf DC-EVPN
 no shutdown
!
interface TenGigabitEthernet0/0/0
 description TO P-JKT HundredGigE1/0/2
 mtu 9216
 ip address 10.1.0.3 255.255.255.254
 ip ospf 100 area 0
 ip ospf network point-to-point
 mpls ip
 service-policy output SP-CORE-EGRESS-100G
 no shutdown
!
interface GigabitEthernet0/0/0
 description TO CE-BANK-JKT - PE-CE BGP
 vrf forwarding BANK
 ip address 192.168.255.2 255.255.255.252
 ip verify unicast source reachable-via rx
 service-policy input SP-PE-INGRESS-BANK-GOLD
 service-policy output SP-PE-CE-SHAPE-50M
 no shutdown
!
interface GigabitEthernet0/0/1
 description TO CE-RETAIL-JKT - PE-CE STATIC
 vrf forwarding RETAIL
 ip address 192.168.255.6 255.255.255.252
 service-policy input SP-PE-INGRESS-RETAIL-SILVER
 no shutdown
!
ip route vrf RETAIL 172.17.10.0 255.255.255.0 192.168.255.5
ip route vrf INTERNET 0.0.0.0 0.0.0.0 10.0.0.31
!
archive
 path flash:archive-$h-$t
 maximum 14
 write-memory
```

Catatan SP: `as-override` karena Bank gunakan ASN identik di JKT dan SBY. `redistribute connected` di VRF INTERNET agar PE advertise subnet PE-CE ke ASBR. Static di RETAIL karena Retail tidak memiliki ASN.

#### 9.4.3 PE Surabaya, ASR 1000-X, PE SBY-01

```
hostname PE-SBY-01
!
vrf definition BANK
 rd 64500:1001
 route-target export 64500:1001
 route-target import 64500:1001
 !
 address-family ipv4
 route-target export 64500:1001
 route-target import 64500:1001
 exit-address-family
!
vrf definition RETAIL
 rd 64500:1002
 route-target export 64500:1002
 route-target import 64500:1002
 !
 address-family ipv4
 route-target export 64500:1002
 route-target import 64500:1002
 exit-address-family
!
router bgp 64500
 bgp router-id 10.0.0.12
 no bgp default ipv4-unicast
 neighbor 10.0.0.1 remote-as 64500
 neighbor 10.0.0.1 update-source Loopback0
 neighbor 10.0.0.2 remote-as 64500
 neighbor 10.0.0.2 update-source Loopback0
 !
 address-family vpnv4
 neighbor 10.0.0.1 activate
 neighbor 10.0.0.1 send-community extended
 neighbor 10.0.0.2 activate
 neighbor 10.0.0.2 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf BANK
 neighbor 192.168.255.9 remote-as 65001
 neighbor 192.168.255.9 description CE-BANK-SBY
 neighbor 192.168.255.9 activate
 neighbor 192.168.255.9 as-override
 redistribute connected
 exit-address-family
 !
 address-family ipv4 vrf RETAIL
 redistribute connected
 redistribute static
 exit-address-family
!
interface GigabitEthernet0/0/0
 description TO CE-BANK-SBY
 vrf forwarding BANK
 ip address 192.168.255.10 255.255.255.252
 no shutdown
!
ip route vrf RETAIL 172.17.20.0 255.255.255.0 192.168.255.13
```

#### 9.4.4 CE Bank, Catalyst 8200, CE BANK JKT

CE tidak mengetahui MPLS. CE hanya mengetahui BGP ke PE.

```
hostname CE-BANK-JKT
!
license boot level network-advantage
!
interface Loopback0
 ip address 172.16.10.1 255.255.255.255
!
interface GigabitEthernet0/0/0
 description TO PE-JKT-01 - MPLS L3VPN
 ip address 192.168.255.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/0/1
 description LAN BANK JKT
 ip address 172.16.10.1 255.255.255.0
 no shutdown
!
router bgp 65001
 bgp router-id 172.16.10.1
 neighbor 192.168.255.2 remote-as 64500
 neighbor 192.168.255.2 description PE-JKT-01
 !
 address-family ipv4 unicast
 neighbor 192.168.255.2 activate
 network 172.16.10.0 mask 255.255.255.0
 exit-address-family
```

CE Bank SBY identik, namun network 172.16.20.0/24 dan neighbor 192.168.255.10.

Verifikasi CE: `show ip bgp summary`, `show ip route bgp`, `ping 172.16.20.1 source 172.16.10.1`, `traceroute 172.16.20.1`. Traceroute wajib menunjukkan hop PE, P, PE, namun CE tidak diperkenankan dapat ping Loopback core. Itu by design.

#### 9.4.5 CE Retail, ISR 4000, Static Simpel

```
hostname CE-RETAIL-JKT
!
interface GigabitEthernet0/0/0
 description TO PE-JKT-01
 ip address 192.168.255.5 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/0/1
 description LAN RETAIL JKT
 ip address 172.17.10.1 255.255.255.0
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 192.168.255.6
```

Pada PE, static sudah tersedia. Dua arah wajib tersedia.

#### 9.4.6 ASBR Internet, ASR 1000-X

```
hostname ASBR-JKT-01
!
vrf definition INTERNET
 rd 64500:9000
 route-target export 64500:9000
 route-target import 64500:9000
 !
 address-family ipv4
 route-target export 64500:9000
 route-target import 64500:9000
 exit-address-family
!
router bgp 64500
 bgp router-id 10.0.0.31
 no bgp default ipv4-unicast
 neighbor 10.0.0.1 remote-as 64500
 neighbor 10.0.0.1 update-source Loopback0
 neighbor 203.0.113.1 remote-as 135000
 neighbor 203.0.113.1 description UPSTREAM-A
 neighbor 198.51.100.1 remote-as 135001
 neighbor 198.51.100.1 description UPSTREAM-B
 !
 address-family vpnv4
 neighbor 10.0.0.1 activate
 neighbor 10.0.0.1 send-community extended
 exit-address-family
 !
 address-family ipv4 unicast
 neighbor 203.0.113.1 activate
 neighbor 203.0.113.1 send-community
 neighbor 203.0.113.1 route-map UPSTREAM-IN in
 neighbor 203.0.113.1 route-map UPSTREAM-OUT out
 neighbor 198.51.100.1 activate
 neighbor 198.51.100.1 send-community
 neighbor 198.51.100.1 route-map UPSTREAM-IN in
 neighbor 198.51.100.1 route-map UPSTREAM-OUT out
 exit-address-family
 !
 address-family ipv4 vrf INTERNET
 redistribute connected
 redistribute static
 exit-address-family
```

#### 9.4.7 VPWS untuk Bank, L2 Stretch

Bank butuh satu VLAN identik di JKT dan SBY untuk cluster database lama. Gunakan VPWS.

PE JKT-01:

```
pseudowire-class BANK-PW
 encapsulation mpls
!
interface GigabitEthernet0/0/3
 description TO CE-BANK-L2
 no switchport
 xconnect 10.0.0.12 1001 encapsulation mpls pw-class BANK-PW
 no shutdown
```

PE SBY-01:

```
pseudowire-class BANK-PW
 encapsulation mpls
!
interface GigabitEthernet0/0/3
 description TO CE-BANK-L2-SBY
 no switchport
 xconnect 10.0.0.11 1001 encapsulation mpls pw-class BANK-PW
 no shutdown
```

Verifikasi: `show xconnect all`, `show mpls l2transport vc 1001 detail`

### 9.5 Verifikasi End to End SP

Jalankan urutan ini dari NOC:

```
! 1. Underlay
show ip ospf neighbor
show ip route 10.0.0.12
show mpls ldp neighbor
show mpls forwarding-table

! 2. BGP VPNv4
show bgp vpnv4 unicast summary
show bgp vpnv4 unicast vrf BANK summary
show bgp vpnv4 unicast rd 64500:1001 172.16.20.0

! 3. VRF
show ip route vrf BANK
show ip route vrf RETAIL
show ip vrf detail BANK

! 4. CE ke CE
ping vrf BANK 172.16.20.1 source 172.16.10.1
ping vrf RETAIL 172.17.20.1 source 172.17.10.1 size 9000 df-bit
traceroute vrf BANK 172.16.20.1

! 5. Isolasi, wajib failure
ping vrf BANK 172.17.20.1
ping vrf RETAIL 172.16.20.1

! 6. Internet
ping vrf INTERNET 8.8.8.8 source 203.0.113.10
show bgp ipv4 unicast summary
show ip route vrf INTERNET

! 7. EVPN dan VPWS
show bgp l2vpn evpn summary
show nve peers
show xconnect all
show mpls l2transport vc detail
```

### 9.6 Troubleshooting

| Gejala | Penyebab | Periksa |
|---|---|---|
| CE tidak dapat ping CE lain, BGP VPNv4 Established | RT mismatch, RD duplicate | show ip vrf detail, show bgp vpnv4 unicast rd |
| BGP VPNv4 tidak Established | Loopback tidak reach, RR salah | show ip route, show bgp vpnv4 unicast summary |
| LDP neighbor down | MTU mismatch, OSPF down, mpls ip belum ada | show mpls ldp neighbor, show ip ospf neighbor |
| Ping kecil beroperasi, ping besar gagal | MTU bukan 9216 di satu link | ping size 9000 df-bit, show interfaces mtu |
| Bank beroperasi, Retail tidak | Static di PE Retail belum ada | show ip route vrf RETAIL |
| Internet beroperasi, L3VPN down | VRF INTERNET leak ke global | show ip route vrf INTERNET |
| VPWS down | VC ID berbeda, LDP down, xconnect peer salah | show xconnect all |
| QoS Gold drop | Policer di PE terlalu kecil | show policy-map interface |

### 9.7 Operasi, SLA, Migrasi

SLA: Availability 99.95 persen, Latency JKT-SBY di bawah 15 ms, Loss di bawah 0.1 persen, MTTR 4 jam P1.

Migrasi dari VRF-Lite / GRE ke MPLS:

Fase 1: Bangun MPLS core parallel, OSPF + LDP beroperasi, customer masih di GRE lama.
Fase 2: Migrasi satu customer pilot, Bank, ke MPLS. Uji 1 minggu.
Fase 3: Migrasi per customer, satu per malam. Periksa ping, traceroute, QoS, billing.
Fase 4: Nonaktifkan GRE lama setelah semua customer bermigrasi dan stabil 2 minggu.
Fase 5: Dokumentasi dan serah menerima ke NOC.

Hasil ISP: Tambah customer baru dari 2 minggu jadi 1 hari. Voice Bank tidak lagi terdegradasi dengan video. NOC dapat troubleshooting dengan framework Modul 8 dalam 10 menit.

### 9.8 Kesimpulan Studi Kasus

ISP berhasil karena: P tidak mengetahui VRF, PE yang mengetahui VRF, RR yang reflect. VRF per customer, RD dan RT unik, isolasi 100 persen. Internet di VRF sendiri. QoS SP dijual per kelas, policer di PE. VPWS untuk L2 stretch, EVPN untuk DCI, L3VPN untuk routing. Semua change gunakan archive + configure replace.

## Appendix: Perintah Verifikasi Cepat & Safe Change Template SP


### Perintah Verifikasi Cepat, Copy Paste

```
show license summary
show ip ospf neighbor
show bfd neighbors
show mpls ldp neighbor
show mpls forwarding-table
show bgp vpnv4 unicast summary
show bgp l2vpn evpn summary
show nve peers
show nve vni
show xconnect all
show ip route vrf BANK
show ip route vrf RETAIL
show ip route vrf INTERNET
show policy-map interface HundredGigE1/0/1 output
show ip bgp validation
show archive
```

### Safe Change Template SP

```
copy running-config flash:pre-change-SP.cfg
reload in 10
configure terminal
! ... perubahan ...
reload cancel
write memory
! Jika gagal:
configure replace flash:pre-change-SP.cfg force
```
