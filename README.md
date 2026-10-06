[README.md](https://github.com/user-attachments/files/33105610/README.md)
# LAB 5 – Thiết lập mô hình tường lửa pfSense

Báo cáo thực hành môn **An toàn hệ thống thông tin** – Bộ môn An toàn Thông tin.

## Thông tin sinh viên

| Thông tin | Nội dung |
| :-- | :-- |
| Họ và tên | Huỳnh Hữu Hoàng |
| MSSV | 1150080095 |
| Lớp | 11CNPM2 |
| Giảng viên | Phạm Trọng Huynh |
| Ngày thực hiện | 06/10/2026 |
| File báo cáo | [11CNPM2-LAB5_1150080095-HuynhHuuHoang.docx](11CNPM2-LAB5_1150080095-HuynhHuuHoang.docx) |
| Video thực hành | *(chưa cập nhật)* |

## Mục tiêu bài lab

- Xây dựng mô hình mạng có tường lửa pfSense bảo vệ LAN và vùng DMZ.
- Cấu hình WAN, LAN, DMZ, NAT và firewall rule.
- Dùng Windows Server làm máy trong LAN để kiểm thử lưu lượng.
- Thực hành nhiều tình huống firewall thay vì chỉ tạo một rule đơn giản.

## Môi trường thực hành

| Thành phần | Thực tế |
| :-- | :-- |
| Máy thật | Windows 11, Intel Core i5-11400H, RAM 8 GB; ra Internet qua Wi-Fi 192.168.1.0/24 |
| Ảo hóa | VMware Workstation 26 (tài liệu gốc viết cho VirtualBox, đã ánh xạ mạng tương đương) |
| Firewall | pfSense CE 2.7.2-RELEASE (amd64), hostname `pfSense-1150080095`; 1 GB RAM, 2 vCPU, ổ 20 GB, 3 card e1000 |
| Domain Controller | Windows Server 2025 Standard Evaluation – 10.0.0.2/8, domain `vietnam.local`, DNS forwarder 8.8.8.8 |
| DMZ-Web | Windows Server 2025 (clone từ bản mới cài, không join domain) – 172.16.0.2/16, chạy IIS |
| Bộ cài | `pfSense-CE-2.7.2-RELEASE-amd64.iso.gz` từ mirror Netgate, SHA-256 `883fb7bc…cd9e4` khớp |

| Card pfSense | Tài liệu (VirtualBox) | Thực tế (VMware) |
| :-- | :-- | :-- |
| Adapter 1 – WAN (em0) | Bridged Adapter | Bridged – VMnet0 bridge vào card Wi-Fi |
| Adapter 2 – LAN (em1) | Host-only, máy thật 10.0.0.100/8, tắt DHCP | Custom VMnet10 (Host-only 10.0.0.0/8, tắt DHCP); card VMnet10 của máy thật đặt tay 10.0.0.100/8 |
| Adapter 3 – DMZ (em2) | Internal Network `dmz-net` | LAN segment `dmz-net` |

```
                          Internet
                             │
              Modem Wi-Fi 192.168.1.1 (192.168.1.0/24)
                             │  Bridged: VMnet0 → card Wi-Fi
                   WAN em0: 192.168.1.2 (DHCP)
                 ┌───────────┴───────────┐
                 │   pfSense CE 2.7.2    │
                 │  pfSense-1150080095   │
                 └─────┬───────────┬─────┘
       LAN em1: 10.0.0.1/8       DMZ em2: 172.16.0.1/16
       VMnet10 (Host-only)       LAN segment "dmz-net"
         ┌─────┴──────┐                 │
   DC 10.0.0.2   Máy thật 10.0.0.100   DMZ-Web 172.16.0.2
 (vietnam.local)  (quản trị WebGUI)     (IIS – cổng 80)
```

## Tóm tắt kết quả

| Mục | Cấu hình chính | Kết quả |
| :-- | :-- | :-- |
| A.2 Kiểm tra trùng dải | `ipconfig`, `route print -4` trên máy thật | VMnet2 (172.16.16.0/24) và VMnet3 (172.16.17.0/24) còn từ bài Sophos trùng dải DMZ → tạm Disable; Wi-Fi, VMnet1, VMnet8 không trùng |
| A.2 Mạng VMware | Virtual Network Editor | VMnet10 Host-only 10.0.0.0/8 không DHCP; máy thật 10.0.0.100/8, không gateway/DNS |
| B.1 Máy ảo pfSense | FreeBSD 14 64-bit, 3 card mạng | Bridged / VMnet10 / LAN segment dmz-net |
| B.2 Cài đặt | Auto (ZFS) → stripe → da0 | Cài thành công 2.7.2-RELEASE, tháo ISO sau khi cài |
| B.3 Console | Assign Interfaces, Set interface IP | Đối chiếu MAC `…:51 / …:5b / …:65`; WAN em0 = 192.168.1.2, LAN em1 = 10.0.0.1/8, không bật DHCP server |
| B.4 Domain Controller | IP tĩnh, AD DS, forest `vietnam.local`, forwarder 8.8.8.8 | `ping 8.8.8.8` và `Resolve-DnsName example.com` thành công |
| B.5 WebGUI | `https://10.0.0.1`, Setup Wizard | Dashboard đúng 2.7.2-RELEASE; máy thật ping 10.0.0.1 TTL = 64 |
| B.6 DMZ | OPT1 = em2, Static 172.16.0.1/16, upstream gateway None | DMZ-Web đặt 172.16.0.2/16, cài IIS, `curl.exe http://localhost` trả về trang web |
| B.7 Outbound NAT | Chế độ Hybrid | Automatic Rules NAT 10.0.0.0/8 và 172.16.0.0/16 ra WAN address |
| B.8 Chuẩn hóa LAN | Disable 2 rule Default allow, Reset States, thêm Pass LAN net → Any | `Diagnostics → Ping 8.8.8.8`: 3/3 gói, 0% loss |
| B.9 Kiểm thử từ DC | Bật / tắt rule nền tảng + Reset States | Rule bật: ping và HTTPS ra Internet được; rule tắt: `ping 8.8.8.8` 100% loss |

## Nội dung file báo cáo

| Mục | Nội dung |
| :-- | :-- |
| A.1 – A.3 | Mục tiêu, ánh xạ VirtualBox → VMware, bảng IP, kiểm tra trùng dải mạng, phân bổ tài nguyên |
| B.1 – B.3 | Tạo VM pfSense, kiểm tra SHA-256, cài đặt, gán interface và đặt LAN trên console |
| B.4 | Domain Controller `vietnam.local` và DNS forwarder |
| B.5 | Truy cập WebGUI, Setup Wizard, Dashboard |
| B.6 | Cấu hình vùng DMZ và dựng máy DMZ-Web (IIS) |
| B.7 | Outbound NAT tự động (Hybrid) |
| B.8 – B.9 | Chuẩn hóa ruleset LAN, rule nền tảng và kiểm thử bật/tắt rule từ DC |
| B.10 | Máy LAN-Test: vai trò, cấu hình dự kiến, lý do không dùng máy thật / không clone DC |
| D | Trả lời 6 câu hỏi |

## Ảnh minh chứng (thư mục `AnhLab5/`)


## Ghi chú

- Tài liệu hướng dẫn viết cho VirtualBox; bài làm dùng VMware Workstation với mạng tương đương (Bridged, Host-only VMnet10, LAN segment `dmz-net`).
- Dùng Windows Server 2025 thay cho 2019/2022 (thao tác AD DS, DNS, IIS giống nhau). pfSense cấp 1 GB RAM thay vì 2 GB do máy thật chỉ có 8 GB.
- Đã thực hiện đầy đủ phần cấu hình nền tảng (mục B.1 – B.9) và trả lời 6 câu hỏi mục D.
- Chưa thực hiện: mục C (5 tình huống firewall) và máy LAN-Test; mục B.10 trong báo cáo chỉ giải thích vai trò và cấu hình dự kiến của LAN-Test.
- pfSense CE 2.7.2 là bản cũ, chỉ dùng trong mạng lab ảo hóa, cô lập; trong lab tạm Disable hai card VMnet2/VMnet3 của máy thật để tránh trùng dải DMZ.
- Phạm vi: chỉ cấu hình trên các máy ảo do chính sinh viên dựng, phục vụ học tập và nghiên cứu, không nhằm mục đích phá hoại.
