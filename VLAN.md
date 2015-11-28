#Lab OpenvSwitch - VLAN

###1.Mô hình bài lab

<img src="http://i.imgur.com/dZ8vDIP.png">

###2.Chuẩn bị: 

-1 máy ảo VMware cài đặt openvswitch và kvm (có hỗ trợ ảo hóa )

-1 card mạng kết nối được với internet

###3.Mục tiêu:

-Tạo ra 2 switch nối với nhau bởi đường trunk. Trên    mỗi switch cắm 2 VM thuộc 2 VLAN khác nhau.

-Các VM thuộc cùng VLAN có thể ping được với nhau

-Các VM khác VLAN sẽ không nhìn thấy nhau 

###4.Các định nghĩa, thuật ngữ sử dụng
- Br0: Bản chất là 1 Switch ảo tạo ra bởi công nghệ OpenVSwitch

(Lưu ý: Br0 là 1 switch, sau khi tạo ra switch này thì mặc định 1 cổng cũng tên là br0 được tạo ra. (cần phân biệt rõ 2 khái niệm này)
	
Br0 chỉ là tên gọi tượng trưng, có thể thay đổi tên này tùy theo mục đích người sử dụng.)
	 
- Tag: Mỗi vlan có 1 tag tương ứng (ví dụ: vlan 2 có tag =2, vlan3 có tag = 3), tag được gán cho các port trên switch, các port cùng tag mới kết nối được với nhau (có thể hiểu ý nghĩa tương ứng với câu lệnh switch port access VLAN)

- Tap: Mỗi VM khi được tạo ra sẽ được gán 1 interface như  1 driver vật lý. Trong ubuntu, các VM được gán tap có tên vnet0, vnet1...

###5.Các bước cài đặt:
#####-Bước 1: Kiểm tra khả năng hỗ trợ ảo hóa của host:

- Tích chọn 2 options Virtualize trên VMware
- Cài đặt gói kiểm tra:
 
        apt-get install cpu-checker 
        kvm-ok

    
Kết quả trả về :

    INFO: /dev/kvm exists
    KVM acceleration can be used


Chứng tỏ máy chủ có khả năng hỗ trợ ảo hóa 

#####-Bước 2: Cài đặt các gói cần thiết:
	#apt-get update && apt-get upgrade
	#apt-get install -y qemu-kvm libvirt-bin virtinst bridge-utils
	#apt-get install -y openvswitch-switch openvswitch-datapath-dkms

#####-Bước 3:Tạo ra các máy ảo bằng kvm: 


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

	#brctl show
	#brctl delif virbr0 vnet0


Sửa ip cho các máy ảo sao cho VM1 và VM3 cùng 1 dải mạng, VM2 và VM4 cùng 1 dải mạng.

#####-Bước 4: Tạo các switch ảo bằng openvswitch:
	#ovs-vsctl add-br br0
	#ovs-vsctl add-br br1

Nối 2 switch này với nhau, đường nối này sẽ mặc định là đường trunk. Để tạo ra kết nối này, cần tạo ra 2 port riêng rẽ trên 2 switch và nối chúng với nhau:

    #ovs-vsctl add-port br0 P0
    #ovs-vsctl add-port br1 P1
    #ovs-vsctl set interface P0 type=patch options:peer=P1
    #ovs-vsctl set interface P1 type=patch options:peer=P0


#####-Bước 5: Gắn các VM vào switch thông qua dây nhảy mạng tap:
	#ovs-vsctl add-port br0 vnet0
	#ovs-vsctl add-port br0 vnet1
	#ovs-vsctl add-port br1 vnet2
	#ovs-vsctl add-port br1 vnet3
 
#####-Bước 6: Gán các VM vào các VLAN: 
Với VM1, VM3 thuộc VLAN2. VM2, VM4 thuộc VLAN3

    #ovs-vsctl set port vnet0 tag=2
    #ovs-vsctl set port vnet2 tag=2
    #ovs-vsctl set port vnet1 tag=3
    #ovs-vsctl set port vnet3 tag=3


###6.Tiến hành test:
Ping cùng VLAN và khác VLAN, ta có kết quả như sau: 


| STT | Thực hiện | kiểm tra| kết quả |
|--------------|-------|------|-------|
| 1 | VM1 – br0 gắn vào VLAN2, VM3 – br1 gắn vào VLAN2 | Ping VM1 và VM3 | Thành công |
| 2 | VM2 – br0 gắn vào VLAN3, VM4 – br1 gắn vào VLAN3 | Ping VM2 và VM4 | Thành công |
| 3 | VM1 – br0 gắn vào VLAN2, VM2 – br0 gắn vào VLAN3 | Ping VM1 và VM2 | Không thành công do khác Vlan. |
| 4 | VM3 – br1 gắn vào VLAN2, VM4 – br1 gắn vào VLAN3 | Ping  VM3 và VM4 | Không thành công do khác Vlan. |

