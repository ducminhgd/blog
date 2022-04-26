# Điều gì xảy ra khi một hệ thống Linux được khởi động

Bài blog này được viết lại theo cách hiểu của mình khi đọc một bài viết khác[^1] trên [opensource.com][https://opensource.com] và trong bài viết đó thì đề cập đến việc sử dụng bootup sequence [GRUB2 Bootloader](https://en.wikipedia.org/wiki/GNU_GRUB) và quá trình khởi động được thực hiện bởi [systemd initialization system](https://en.wikipedia.org/wiki/Systemd).

Trên thực tế thì để một máy Linux có thể sử dụng được thì bao gồm 2 giai đoạn là: *boot*[^2] và *startup*[^3]. *Boot* là quá trình khởi động khi một máy tính được mở (turn on) và hoàn thành giai đoạn OS Kernel được khởi tạo và systemd được chạy. *Startup* là quá trình từ giai đoạn còn lại đến lúc một máy Linux có thể sử dụng được.

Có thể tóm tắt quá trình khởi động của một máy linux gồm những bước sau:

- BIOS POST (Basic IO System Power On Self Test)
- Boot loader (GRUB2)
- Khởi tạo Kernel
- Khởi động systemd.

## Boot

### BIOS POST

Bước đầu tiên khi khởi động một máy Linux, máy sẽ chạy POST (Power On Self Test). POST là một phần của BIOS, dùng để đảm bảo các phần cứng của máy tính được thực hiện đúng. Nếu việc chạy POST thất bại thì máy có thể không sử dụng được, do đó quá trình boot sẽ bị dừng lại.

BIOS POST kiểm tra khả năng hoạt động cơ bản của phần cứng và gửi tín hiệu `INT 13H`, một vector interupt của BIOS, để xác định boot sectors trên các thiết bị có thể boot được. Boot sector chứa boot record đầu tiên được tìm thấy sẽ được load lên RAM và gửi code ghi nhận được load từ boot sector.

Có ba boot laders được dùng nhiều nhất trên các bản phân phối Linux (Linux distributions) là GRUB, GRUB2 và LILO. Trong đó, GRUB2 là mới nhất và thường được sử dụng ở các bản phân phối Linux.

### GRUB2

GRUB2, viết tắt của _GRand Unified Bootloader, version 2_, là chương trình giúp máy tính tìm kiếm kernel của hệ điều hành và load lên bộ nhớ. GRUB2 được thiết kế tương thích với các đặc tính của multiboot, cho phép GRUB2 boot nhiều phiên bản của Linux và các OS miễn phía khác; ngoài ra có thể "chain load" boot record của các hệ điều hành độc quyền.

GRUB1 hay GRUB2 thì đều có 3 giai đoạn để load Linux Kernel lên bộ nhớ và chạy.

#### Bước 1 (stage 1)

Khi bước BIOS POST kết thúc, BIOS tìm kiếm boot record trong các ổ đĩa, thường là được đặt tại Master Boot Record (MBR), load cái đầu tiên tìm được lên memory và bắt đầu thực thi boot record đó. Đoạn code bootstrap rất nhỏ vì chỉ có 512-byte để chứa, thực tế thì chỉ có 446 byte dùng để chứa code[^4].

Do dung lượng của boot record nhỏ, nên nó không đủ thông minh để hiểu được cấu trúc file system, nhiệm vụ duy nhất của nó là xác định và load bước 1.5. Sau khi load bước 1.5, bước 1 sẽ chuyển quyền điều khiển về buóc 1.5

#### Bước 1.5 (stage 1.5)

Bước 1.5 của GRUB2 phải được đặt giữa boot record và phân vùng đầu tiên của đĩa, vùng trống này từng được chừa lại vì một số lý do kỹ thuật. Phân vùng đầu tiên của ổ cứng được bắt đầu tại vị trí sector 63 và MBR thì có vị trí sector 0, do đó còn 62 sectors còn trống (512 bytes/sector) để chứ file `core.img`. File `core.img` có dung lượng 25,389 bytes, nên 62 sectors là dư để chứ.

Phần này có thể chứa code của một vài filesystem drivers phổ biến, như chuẩn EXT của Linux, FAT và NTFS. So với GRUB1 thì code trong bước 1.5 của GRUB2 phức tạp hơn, nên có thể xác định được địa chỉ của EXT Filesystem, chỉ là không xác định được trên ổ logic (logical volume). Địa chỉ tiêu chuẩn cho các file bước 2 là trong `/boot`, cụ thể hơn là `/boot/grub2`

#### Bước 2 (stage 2)

Tất cả file để chạy GRUB2 bước 2 của được đặt trong `/boot/grub2` và nhiều thư mục con. GRUB2 không có file image như bước 1 và 1.5, thay vào đó là chứa các module kernel được tải khi cần thiết từ thư mục `/boot/grub2/i386-pc`.

Tính năng chính của GRUB2 bước 2 là xác định và load Linux Kernel lên RAM và chuyển quyền điều khiển về cho kernel. Kernel và các file liên quan được đặt trong thư mục `/boot`, các file kernel thường bắt đầu với `vmlinuz`.'

GRUB hỗ trợ boot một lựa chọn từ danh sách các Kernels, và thường cung cấp một menu pre-boot để chọn các kernels đã được cài đặt.

Bước 2 của GRUB2 load kernel được chọn lên bộ nhớ và chuyển quyền điều khiển máy về cho kernel. Kernel

### Kernel

Tất cả các kernel đều là dạng file nén có thể tự bung (self-extracting compressed file) để tối ưu lưu trữ. Kernel được đặt trong `/boot` cùng với initial RAM disk image, và bảng map các ổ cứng.

Sau khi được chọn để load, kernel sẽ tự giải nén và load *systemd*[^5], và chuyển quyền điều khiển cho *systemd*

## Startup

Startup là đoạn tiếp theo sau boot, để đưa một máy tính vào trạng thái có thể sử dụng được.

### systemd

*systemd* là chương trình mẹ của tất cả các tiếng trình, nó có nhiệm vụ đưa một máy Linux lên trạng thái có thể sử dụng được. Một vài tính năng của nó được mở rộng ra so với các chương trình cũ, có thể làm được nhiều việc như mouting filesystems, khởi động và quản lý các dịch vụ hệ thống[^6]

*systemd* mount các file hệ thống được khai báo trong `/etc/fstab`, bao gồm swap files và các phân vùng. Sử dụng file cấu hình tại `/etc/systemd/system/default.target` để xác định trạng thái hoặc đối tượng cần được boot vào. `default.target` chỉ là symbolic link trỏ đến file target thật sự.

Các đối tượng và dịch vụ chỉ là các đơn vị của *systemd*

Các đối tượng (target) có một tập các dependencies được mô tả trong file config của chính nó. *systemd* khởi động các dependencies được yêu cầu. Khi các dependencies được load hết thì hệ thống được chạy với cấp bậc của target đó.

*systemd* có một tính năng trong thiết kế là có thể start nhiều target một các song song được, nên thông thường có một vài target trước đó để làm checkpoint cho việc startup, checkpoints được xem là không thành công nếu các dịch vụ và target được yêu cầu trước đó không đủ. Thường có 2 checkpoints là `sysinit.target` và `basic.target`.

```
local-fs-pre.target
        |
        v
(various mounts and   (various swap   (various cryptsetup
fsck services...)     devices...)        devices...)       (various low-level   (various low-level
        |                  |                  |             services: udevd,     API VFS mounts:
        v                  v                  v             tmpfiles, random     mqueue, configfs,
    local-fs.target      swap.target     cryptsetup.target    seed, sysctl, ...)      debugfs, ...)
        |                  |                  |                    |                    |
        \__________________|_________________ | ___________________|____________________/
                                                \|/
                                                v
                                        sysinit.target
                                                |
            ____________________________________/|\________________________________________
        /                  |                  |                    |                    \
        |                  |                  |                    |                    |
        v                  v                  |                    v                    v
    (various           (various               |                (various          rescue.service
    timers...)          paths...)              |               sockets...)               |
        |                  |                  |                    |                    v
        v                  v                  |                    v              rescue.target
    timers.target      paths.target             |             sockets.target
        |                  |                  |                    |
        v                  \_________________ | ___________________/
                                                \|/
                                                v
                                        basic.target
                                                |
            ____________________________________/|                                 emergency.service
        /                  |                  |                                         |
        |                  |                  |                                         v
        v                  v                  v                                 emergency.target
    display-        (various system    (various system
manager.service         services           services)
        |             required for            |
        |            graphical UIs)           v
        |                  |           multi-user.target
        |                  |                  |
        \_________________ | _________________/
                            \|/
                            v
                    graphical.target
```

## Đọc thêm

1. https://en.wikipedia.org/wiki/GNU_GRUB
1. https://en.wikipedia.org/wiki/Systemd
1. https://en.wikipedia.org/wiki/INT_13H
1. https://en.wikipedia.org/wiki/Master_boot_record

[^1] https://opensource.com/article/17/2/linux-boot-and-startup
[^2] *Boot* là quá trình tải OS Kernel, Drivers, khởi tạo phần cứng và có thể là khởi động GUI
[^3] *Startup* là quá trình mà hệ điều hành tải và khởi tạo các services và chương trình để có thể chạy được, ví dụ: khởi động thiết bị mạng. Quá trình starup được gọi là kết thúc khi mức sử dụng CPU và đọc ghi đĩa trở về mức thấp.
[^4] https://en.wikipedia.org/wiki/Master_boot_record
[^5] *systemd* là chương trình thay thế cho chương trình *SysV init*.
[^6] Những task nào của *systemd* không liên quan đến việc startup sẽ không được nói đến trong bài này.
