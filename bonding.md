#Lab OpenvSwitch - Bonding

###1.Mô hình bài lab

<img src="http://i.imgur.com/tHiWVjC.png">

###2.Chuẩn bị:

-1 máy ảo VMware cài đặt openvswitch và kvm (có hỗ trợ ảo hóa )

-1 card mạng kết nối được với internet

###3.Mục tiêu:

-Tìm hiểu về cơ chế Bonding

-Khi sử dụng cơ chế Bonding hệ thống mạng sẽ được tăng băng thông và có tính backup: khi bond 2 đường với nhau thì 1 đường chết, đường kia sẽ vẫn giúp mạng hđ bình thường.

-Tránh được vấn đề loop khi tắt tính năng STP


###4.Các định nghĩa, thuật ngữ sử dụng
- Br0: Bản chất là 1 Switch ảo tạo ra bởi công nghệ OpenVSwitch

(Lưu ý: Br0 là 1 switch, sau khi tạo ra switch này thì mặc định 1 cổng cũng tên là br0 được tạo ra. (cần phân biệt rõ 2 khái niệm này)
    
Br0 chỉ là tên gọi tượng trưng, có thể thay đổi tên này tùy theo mục đích người sử dụng.)
     
- Tag: Mỗi vlan có 1 tag tương ứng (ví dụ: vlan 2 có tag =2, vlan3 có tag = 3), tag được gán cho các port trên switch, các port cùng tag mới kết nối được với nhau (có thể hiểu ý nghĩa tương ứng với câu lệnh switch port access VLAN)

- Tap: Mỗi VM khi được tạo ra sẽ được gán 1 interface như  1 driver vật lý. Trong ubuntu, các VM được gán tap có tên vnet0, vnet1...


###5.Các bước cài đặt:
####a.Cấu hình trên host 1

#####-Bước 1: cập nhật hệ thống trước khi cài đặt:
    #sudo apt-get update

#####-Bước 2: Kiểm tra khả năng hỗ trợ ảo hóa của host:

- Tích chọn 2 options Virtualize trên VMware
- Cài đặt gói kiểm tra:

        #apt-get install cpu-checker 
        #kvm-ok

   
Kết quả trả về :

    INFO: /dev/kvm exists
    KVM acceleration can be used


Chứng tỏ máy chủ có khả năng hỗ trợ ảo hóa

#####-Bước 3: Cài đặt các gói cần thiết:
    #apt-get -y install qemu-kvm libvirt-bin virtinst bridge-utils
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

    #brctl show
    #brctl delif virbr0 vnet0

<img src="http://i.imgur.com/Qb9G3rY.png">


#####-Bước 5: Tạo các switch ảo bằng openvswitch:
    #ovs-vsctl add-br br0
    #ovs-vsctl add-br br1

Nối 2 switch này với nhau bằng 2 dây nối. Để tạo ra kết nối này, cần tạo ra 2 port riêng rẽ trên 2 switch và nối chúng với nhau:

    #ovs-vsctl add-port br0 a
    #ovs-vsctl add-port br1 b
    #ovs-vsctl add-port br0 a1
    #ovs-vsctl add-port br1 b1
    #ovs-vsctl set interface a type=patch options:peer=b
    #ovs-vsctl set interface a1 type=patch options:peer=b1

#####-Bước 6: Gắn các VM vào switch thông qua dây nhảy mạng tap:
    #ovs-vsctl add-port br0 vnet0
    #ovs-vsctl add-port br1 vnet1

#####-Bước 7: Sử dụng câu lệnh sau để tạo bonding:

    #ovs-vsctl add-bond br1 bond0 a,a1 lacp=active
    #ovs-vsctl add-bond br1 bond0 b,b1 lacp=active


###6.Tiến hành test:

Sau khi cấu hình bonding sử dụng câu lệnh:

    #ovs-vsctl show
 
Ta thấy 2 đường a, a1 được bond với nhau, b và b1 được bond với nhau.

<img src="http://i.imgur.com/fYtlX6m.png">

Sau khi ngắt 1 trong 2 đường được bond với nhau, kết quả sẽ là 2 mạng vẫn có thể thông bình thường tuy nhiên thời gian downtime trong OpenVswitch khá lâu:


<img src="http://i.imgur.com/bC7m9wl.png">
