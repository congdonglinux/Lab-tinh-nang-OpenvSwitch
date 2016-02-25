#Lab OpenvSwitch - GRE tunnel
###1.Mô hình bài lab
<img src="http://i.imgur.com/dkWq6pe.png">
###2.Chuẩn bị: 
-1 máy ảo VMware cài đặt openvswitch và kvm (có hỗ trợ ảo hóa )
-1 card mạng kết nối được với internet
###3.Mục tiêu:
-Tìm hiểu về cơ chế GRE
-Cho phép máy ảo trên 2 host vật lý khác nhau thông qua gre tunnel có thể truyền thông với nhau
###4.Các định nghĩa, thuật ngữ sử dụng
- Br0: Bản chất là 1 Switch ảo tạo ra bởi công nghệ OpenVSwitch
(Lưu ý: Br0 là 1 switch, sau khi tạo ra switch này thì mặc định 1 cổng cũng tên là br0 được tạo ra. (cần phân biệt rõ 2 khái niệm này)
	
Br0 chỉ là tên gọi tượng trưng, có thể thay đổi tên này tùy theo mục đích người sử dụng.)
	 
- Tag: Mỗi vlan có 1 tag tương ứng (ví dụ: vlan 2 có tag =2, vlan3 có tag = 3), tag được gán cho các port trên switch, các port cùng tag mới kết nối được với nhau (có thể hiểu ý nghĩa tương ứng với câu lệnh switch port access VLAN)
- Tap: Mỗi VM khi được tạo ra sẽ được gán 1 interface như  1 driver vật lý. Trong ubuntu, các VM được gán tap có tên vnet0, vnet1...

###5. Tìm hiểu cơ chế GRE tunnel
####a. Giới thiệu:
- GRE – Generic Routing Encapsulation là giao thức được phát triển đầu tiên bởi Cisco, với mục đích chính tạo ra kênh truyền ảo (tunnel) để mang các giao thức lớp 3 thông qua mạng IP.
- Gói tin sau khi đóng gói được truyền qua mạng IP và sử dụng GRE header để định tuyến.
- Mỗi lần gói tin GRE đi đến đích, lần lượt các header ngoài cùng sẽ được gỡ bỏ cho đến khi gói tin ban đầu được mở ra.

####b.Cơ chế hoạt động:
- Để tạo ra các kênh truyền, GRE cũng thực hiện việc đóng gói gói tin tương tự như giao thức IPSec hoạt động ở Tunnel mode.
- Trong quá trình đóng gói, GRE sẽ thêm các header mới vào gói tin, và header mới này cung cấp các thông số cần thiết dùng để truyền các gói tin thông qua môi trường trung gian.
- GRE chèn ít nhất 24 byte vào đầu gói tin:
- Trong đó 20 byte là IP header mới dùng để định tuyến.
- Còn 4 byte là GRE header.
- Ngoài ra GRE còn có thể tùy chọn thêm 12 byte mở rộng để cung cấp tính năng tin cậy như: checksum, key chứng thực, sequence number.
<img src="http://i.imgur.com/3X7gsy9.png">
- Hai byte đầu tiên trong phần GRE header là GRE flag 2-byte. Phần bày chứa các cờ dùng để chỉ định những tính năng tùy chọn của GRE :
- Bit 0 (checksum): Nếu bit này được bật lên (giá trị bằng 1) thỉ phần checksum được thêm vào sau trường Protocol type của GRE header.
- Bit 2 (key): Nếu bit này được bật lên thì phần Key tính năng chứng thực sẽ được áp dụng, đây như dạng password ở dạng clear text. Khi một thiết bị tạo nhiều tunnel đến nhiều thiết bị đích thì key này được dùng để xác định các thiết bị đích.
- Bit 3 (Sequence number): Khi bit này được bật thì phần sequence number được thêm vào GRE header.
- Hai byte tiếp theo là phần Protocol Type chỉ ra giao thức lớp 3 của gói tin ban đầu được GRE đóng gói và truyền qua kênh truyền.
- Khi gói tin đi qua tunnel nó sẽ được thêm GRE header như trên và sau khi tới đầu bên kia của kênh truyền, gói tin sẽ được loại bỏ GRE header để trả lại gói tin ban đầu.

###6.Các bước cài đặt:
####a.Cấu hình trên host 1
#####-Bước 1: cập nhật hệ thống trước khi cài đặt:
    #sudo apt-get update
#####-Bước 2: Kiểm tra khả năng hỗ trợ ảo hóa của host:
- Tích chọn 2 options Virtualize trên VMware
- Cài đặt gói kiểm tra:
 
        apt-get install cpu-checker 
        kvm-ok
    
Kết quả trả về :
    INFO: /dev/kvm exists
    KVM acceleration can be used
Chứng tỏ máy chủ có khả năng hỗ trợ ảo hóa 
#####-Bước 3: Cài đặt các gói cần thiết:
	#apt-get install -y install qemu-kvm libvirt-bin virtinst bridge-utils
	#apt-get install -y openvswitch-switch openvswitch-datapath-dkms
#####-Bước 4:Tạo ra các máy ảo bằng kvm: 
    #virt-install \
    --name template \
    --ram 2048 \
    --disk path=/var/kvm/images/template.img,size=30 \
    --vcpus 2 \
    --os-type linux \
    --os-variant ubuntutrusty \
    --graphics none \
    --console pty,target_type=serial \
    --location 'http://jp.archive.ubuntu.com/ubuntu/dists/trusty/main/installer-amd64/' \
    --extra-args 'console=ttyS0,115200n8 serial'
   Sau khi tạo, các máy ảo sẽ tự động được cắm vào switch virbr0 được tạo ra bởi linux bridge và nhận ip tự động thuộc dải này. Ta cần tiến hành ngắt kết nối các máy ảo tới switch này để không ảnh hưởng kết quả bài lab:
   
	#    brctl show
	#    brctl delif virbr0 vnet0
	
<img src="http://i.imgur.com/Qb9G3rY.png">

#####-Bước 5: Tạo các switch ảo bằng openvswitch:
	#ovs-vsctl add-br br0
#####-Bước 6: Gắn các VM vào switch thông qua dây nhảy mạng tap:
	#ovs-vsctl add-port br0 vnet0
 
#####-Bước 7: Tạo Gre Tunnel với câu lệnh:
    #ovs-vsctl add-port br0 gre0 -- set interface gre0 type=gre options:remote_ip=<IP eth trên host2>
####b. Cấu hình trên host 2:
Các bước 1,2,3,4,5 tương tự như với host1

#####Bước 6. Tạo Gre tunnel với câu lệnh:

    #ovs-vsctl add-port br0 gre0 -- set interface gre0 type=gre options:remote_ip=<IP eth trên host1>
    
###7.Tiến hành test:
VM 1 và VM3 ping được với nhau do đã tạo GRE Tunnel

| STT | Thực hiện | kiểm tra| kết quả |
|--------------|-------|------|-------|
| 1 | VM1 – Host 1 gắn vào br0 có cấu hình GRE và VM3 - Host 2 gắn vào br0 có cấu hình GRE | Ping VM1 và VM3 | Thành công |
| 2 | VM2 – Host 1 gắn vào br1 không cấu hình GRE và VM4 -Hos2 gắn vào br1 không cấu hình GRE| Ping VM2 và VM4 | Không thành công |

Bắt gói tin trên 1 host để phân tích:

    #tcpdump -i eth0 -w testgre.pcap
    
Sử dụng wireshark để đọc những gói tin bắt đước thấy các gói tin đã được đóng thêm header GRE:
<img src="http://i.imgur.com/6E0SR7B.png">
