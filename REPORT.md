# Báo cáo Session 10 - Ex1: Vì sao doanh nghiệp chuyển từ VPS sang Docker

## 1. Mục tiêu

Báo cáo này phân tích lý do doanh nghiệp chuyển từ mô hình VPS truyền thống
sang Docker container, so sánh máy ảo (Virtual Machine) với container theo
5 tiêu chí, giải thích cơ chế cách ly của Linux gồm Namespaces và cgroups,
đồng thời trích xuất thông số thực tế từ lệnh `docker info` trên máy thực hành.
Toàn bộ thao tác xác thực được thực hiện bằng PowerShell trên Windows, không
dùng `sudo`, không dùng `grep`, chỉ dùng `Select-String`.

Môi trường thực hành:

- OS máy host: Windows + WSL2.
- Docker Desktop 4.87.0, Engine 29.7.2.
- Log đầy đủ lưu tại `logs/docker-info.txt`.

## 2. Vì sao doanh nghiệp chuyển từ VPS sang Docker

VPS (Virtual Private Server) bản chất là một máy ảo chia sẻ tài nguyên trên
một máy chủ vật lý, mỗi VPS chạy một hệ điều hành khách đầy đủ. Mô hình này
từng phổ biến vì rẻ hơn máy chủ vật lý và dễ quản trị, nhưng bộc lộ nhiều
hạn chế khi doanh nghiệp cần triển khai nhanh, mở rộng linh hoạt và tối ưu
chi phí trên quy mô lớn.

Thứ nhất, Docker giúp sử dụng tài nguyên hiệu quả hơn. Mỗi VPS phải gánh một
kernel riêng, RAM cho OS khách, disk cho toàn bộ hệ điều hành, dù ứng dụng
thực tế chỉ cần vài trăm MB. Ngược lại, các container chia sẻ chung kernel
của host, image chỉ chứa ứng dụng và thư viện cần thiết. Nhờ Storage Driver
dạng layer như `overlayfs`, nhiều container dùng chung layer cơ sở, dung lượng
tăng thêm rất ít. Doanh nghiệp chạy được nhiều workload hơn trên cùng một
cụm máy chủ, giảm chi phí thuê hạ tầng cloud.

Thứ hai, tốc độ triển khai và khả năng mở rộng vượt trội. Khởi động một VPS
thường mất hàng chục giây tới vài phút vì phải boot toàn bộ OS. Container chỉ
khởi động tiến trình ứng dụng, thường mất dưới một giây tới vài giây. Khi lưu
lượng tăng đột biến, hệ thống điều phối như Docker Swarm hoặc Kubernetes có
thể nhân bản container theo chiều ngang trong vài giây, còn VPS phải tạo máy
ảo mới, cài đặt, cấu hình, tốn nhiều thời gian và dễ lỗi.

Thứ ba, tính nhất quán môi trường giữa開発, kiểm thử và vận hành. Với VPS,
câu nói kinh điển "trên máy tôi chạy được" xảy ra thường xuyên vì môi trường
dev, staging, production khác nhau về phiên bản thư viện, biến môi trường,
cấu hình OS. Docker đóng gói ứng dụng cùng dependencies thành image bất biến,
định nghĩa bằng Dockerfile, chạy ở đâu cũng giống nhau. Điều này đơn giản hóa
quy trình CI/CD, rollback nhanh bằng cách đổi tag image, kiểm thử đáng tin cậy
hơn.

Thứ tư, Docker phù hợp kiến trúc microservices và DevOps. Một VPS thường chạy
nhiều dịch vụ lẫn lộn, khó tách biệt vòng đời, khó giới hạn tài nguyên từng
dịch vụ. Docker khuyến khích mỗi container một tiến trình chính, mỗi
microservice có image, version, pipeline riêng, dễ thay thế, dễ mở rộng độc
lập. Kết hợp với Compose và Kubernetes, doanh nghiệp quản lý hàng chục tới
hàng trăm service bằng file khai báo, tự động restart, health-check, rolling
update không downtime.

Thứ năm, khả năng di động và tránh phụ thuộc nhà cung cấp. Image Docker tuân
theo chuẩn OCI, chạy được trên laptop, server on-premise, AWS, Azure, GCP mà
không cần chỉnh sửa nhiều. VPS thường gắn với template và API riêng của từng
nhà cung cấp, di chuyển workload tốn công chuyển đổi. Với Docker, doanh nghiệp
dễ thực hiện chiến lược multi-cloud, hybrid-cloud và disaster recovery.

Tóm lại, Docker không thay thế hoàn toàn VPS trong mọi trường hợp cần cách ly
mạnh ở mức kernel, nhưng với đa số ứng dụng web, API, worker, batch job, nó
mang lại chi phí thấp hơn, triển khai nhanh hơn, nhất quán hơn và mở rộng tốt
hơn.

## 3. Bảng so sánh VM và Container

| Tiêu chí | Virtual Machine (VM / VPS) | Container (Docker) |
|---|---|---|
| Dung lượng Image | Lớn, thường vài GB vì chứa toàn bộ OS khách, kernel, drivers, system packages. Template Ubuntu/CentOS tối thiểu cũng khoảng 1 đến 2 GB, snapshot VPS còn lớn hơn. | Nhỏ gọn, từ vài MB tới vài trăm MB. Image chỉ chứa ứng dụng và thư viện runtime, dùng chung kernel host. Layer dùng chung qua `overlayfs` giúp tiết kiệm disk khi chạy nhiều container. |
| Thời gian khởi động | Chậm, từ hàng chục giây tới vài phút. Phải khởi động BIOS ảo, bootloader, kernel khách, systemd, rồi mới tới ứng dụng. | Nhanh, thường dưới 1 giây tới vài giây. Chỉ cần tạo namespaces, cgroups và chạy tiến trình ứng dụng, không boot OS mới. |
| Hao phí OS | Cao. Mỗi VM chạy một kernel và tiến trình hệ thống riêng, tốn RAM và CPU ngay cả khi idle. Hypervisor như KVM, Hyper-V, VMware cũng tiêu tốn thêm tài nguyên. | Thấp. Tất cả container chia sẻ một kernel host, không có OS khách dư thừa. Engine Docker chỉ tốn một lượng nhỏ tài nguyên để quản lý lifecycle container. |
| Mức độ cô lập | Rất mạnh. Cô lập ở mức phần cứng ảo hóa, mỗi VM có kernel, virtual NIC, virtual disk riêng. Lỗi kernel hoặc tấn công escape khó lan sang VM khác. Phù hợp multi-tenant yêu cầu bảo mật cao. | Trung bình, ở mức tiến trình. Cô lập bằng Namespaces và cgroups, vẫn dùng chung kernel. Nhẹ và đủ cho đa số workload, nhưng nếu cần cách ly kernel tuyệt đối thì phải bổ sung Kata Containers, gVisor hoặc giữ lại VM. |
| Khả năng mở rộng | Kém linh hoạt. Scale phải tạo VM mới, cấp phát disk, boot OS, cấu hình network. Tốn thời gian, khó scale theo giây, chi phí cao khi scale ngang hàng trăm instance. | Rất tốt. Image nhẹ nên pull nhanh, start nhanh, dễ nhân bản hàng chục container trên cùng host. Điều phối bằng Swarm hoặc Kubernetes cho phép auto-scaling, rolling update, self-healing tự động. |

## 4. Cơ chế cách ly: Namespaces và cgroups

Docker trên Linux không dùng hypervisor mà dùng hai tính năng của kernel là
Namespaces để cách ly tầm nhìn và cgroups để giới hạn tài nguyên.

### 4.1. Namespaces

Namespaces tạo ra cho mỗi container một góc nhìn riêng về hệ thống, tiến trình
bên trong tưởng như đang chạy trên máy riêng.

- **Process Namespace (`pid`):** Mỗi container có không gian PID riêng. Tiến
  trình số 1 trong container thường là entrypoint của ứng dụng, không nhìn thấy
  tiến trình của host hay container khác qua `ps`. Khi PID 1 kết thúc,
  container dừng. Điều này giúp cô lập vòng đời tiến trình và tín hiệu `kill`.
- **Network Namespace (`net`):** Mỗi container có stack mạng riêng gồm
  interface, địa chỉ IP, bảng routing, iptables, port. Docker tạo cặp
  `veth pair`, một đầu gắn vào container, một đầu gắn vào bridge `docker0`,
  nhờ đó container giao tiếp ra ngoài qua NAT mà không xung đột port trực tiếp
  với host. Muốn public service mới cần `-p host:container`.
- **Mount Namespace (`mnt`):** Mỗi container có cây filesystem riêng. Docker
  kết hợp image layers chỉ đọc với một layer ghi được phía trên nhờ
  `overlayfs`, sau đó `chroot` hoặc `pivot_root` vào mount namespace đó. Nhờ
  vậy container thấy `/app`, `/etc` của riêng mình, không thấy filesystem của
  host, trừ khi cố ý mount volume bằng `-v`.

Ngoài ba loại trên, Docker còn dùng UTS namespace để cách ly hostname, IPC
namespace để cách ly shared memory, User namespace để ánh xạ UID, giúp tăng
cường bảo mật.

### 4.2. cgroups (control groups)

Nếu Namespaces trả lời câu hỏi "nhìn thấy gì", cgroups trả lời câu hỏi "được
dùng bao nhiêu". cgroups giới hạn, thống kê và ưu tiên tài nguyên CPU, RAM,
I/O cho từng nhóm tiến trình.

- **Giới hạn CPU:** Docker cho phép đặt `--cpus`, `--cpu-shares`,
  `--cpu-quota` và `--cpu-period`. Ví dụ `--cpus=1.5` nghĩa là container dùng
  tối đa 1.5 core trong mỗi chu kỳ. Khi host quá tải, scheduler CFS của kernel
  phân phối CPU theo weight, container vượt quota sẽ bị throttle chứ không
  chiếm hết CPU của container khác.
- **Giới hạn RAM:** Docker cho phép đặt `-m` hoặc `--memory` và `--memory-swap`.
  Ví dụ `--memory=512m` giới hạn container chỉ dùng 512 MB RAM, vượt quá sẽ bị
  OOM killer kết thúc tiến trình đó thay vì làm treo cả host. cgroups còn thống
  kê `memory.usage_in_bytes`, giúp `docker stats` hiển thị mức tiêu thụ thực tế.

Chính nhờ kết hợp Namespaces để cách ly và cgroups để giới hạn, nhiều container
chạy chung một kernel mà vẫn an toàn và công bằng tài nguyên, vừa nhẹ hơn VM
vừa dễ quản trị hơn VPS thủ công.

## 5. Thông số thực tế từ `docker info` trên máy này

Lệnh `docker info` và `docker version` đã được chạy bằng PowerShell và lưu
nguyên văn vào `logs/docker-info.txt`. Bảng dưới đây trích các trường quan
trọng bằng `Select-String`:

| Thông số | Giá trị đo được | Ý nghĩa |
|---|---|---|
| Server Version | 29.7.2 | Phiên bản Docker Engine phía server, trùng Client 29.7.2, API 1.55. |
| Cgroup Driver | cgroupfs | Driver quản lý cgroups, Docker trực tiếp ghi vào filesystem cgroup. |
| Cgroup Version | 2 | Kernel dùng cgroups v2, mô hình cây thống nhất hiện đại. |
| Containers | 4 | Tổng 4 container, 0 running, 4 stopped, không có container paused. |
| Images | 9 | Có 9 image lưu local, chia sẻ layer qua overlayfs. |
| Storage Driver | overlayfs | Driver lưu trữ dạng layer, driver-type `io.containerd.snapshotter.v1`. |
| Kernel Version | 6.18.33.2-microsoft-standard-WSL2 | Kernel Linux chạy bên trong WSL2 của Docker Desktop. |
| Operating System | Docker Desktop | Docker chạy chế độ Desktop, VM Linux nhẹ quản lý bởi WSL2. |
| Architecture | x86_64 | Kiến trúc CPU 64-bit, image `amd64` chạy native. |
| CPUs | 12 | Docker được cấp 12 CPU logic từ host Windows. |
| Total Memory | 5.674GiB | Tổng RAM cấp cho Docker VM khoảng 5.67 GB. |
| Runtimes | io.containerd.runc.v2, runc, nvidia | Runtime mặc định là `runc`, quản lý qua containerd, có thêm runtime `nvidia` cho GPU. |
| Logging Driver | json-file | Log container mặc định ghi file JSON. |
| Swarm | inactive | Chưa bật chế độ Swarm, đang chạy standalone. |

Nhận xét nhanh: cấu hình 12 CPU và 5.67 GB RAM là đủ để chạy đồng thời nhiều
container nhẹ cho mục đích học tập. `overlayfs` và Cgroup v2 cho thấy đây là
cấu hình Docker hiện đại, không còn dùng `aufs` hay cgroups v1 cũ.

## 6. Log lệnh xác thực

Toàn bộ lệnh dưới đây chạy bằng PowerShell trên Windows, không dùng `sudo`,
không dùng `grep`.

```powershell
# Tạo thư mục logs nếu chưa có
New-Item -ItemType Directory -Force -Path logs | Out-Null

# Lưu output docker version và docker info vào file log
docker version | Out-File -FilePath logs/docker-info.txt -Encoding utf8
'===== docker info =====' | Out-File -Append -FilePath logs/docker-info.txt -Encoding utf8
docker info | Out-File -Append -FilePath logs/docker-info.txt -Encoding utf8

# Trích xuất các trường quan trọng bằng Select-String
Get-Content logs/docker-info.txt | Select-String -Pattern 'Server Version|Cgroup Driver|Cgroup Version|Storage Driver|Kernel Version|Operating System|Architecture|CPUs|Total Memory|Runtimes|Containers:|Images:'
```

Kết quả trích xuất thực tế:

```text
Containers: 4
Images: 9
Server Version: 29.7.2
Storage Driver: overlayfs
Cgroup Driver: cgroupfs
Cgroup Version: 2
Runtimes: runc io.containerd.runc.v2 nvidia
Kernel Version: 6.18.33.2-microsoft-standard-WSL2
Operating System: Docker Desktop
Architecture: x86_64
CPUs: 12
Total Memory: 5.674GiB
```

File log đầy đủ: `logs/docker-info.txt` (khoảng 4.5 KB, gồm cả `docker version`
và `docker info`). Có thể mở lại bằng `Get-Content logs/docker-info.txt` hoặc
`notepad logs/docker-info.txt`.

## 7. Kết luận

Docker nhẹ hơn VPS nhờ chia sẻ kernel, khởi động nhanh hơn hàng chục lần, tốn
ít disk hơn nhờ image phân lớp, mở rộng ngang dễ dàng nhờ image bất biến và
công cụ điều phối. Cơ chế Namespaces gồm Process, Network, Mount mang lại khả
năng cách ly tiến trình, mạng và filesystem, trong khi cgroups đảm bảo giới hạn
công bằng CPU và RAM. Thông số thực tế trên máy với Engine 29.7.2, overlayfs,
Cgroup v2, 12 CPU và 5.67 GB RAM cho thấy môi trường đã sẵn sàng để thực hành
các bài tiếp theo về Dockerfile, volume và network.
