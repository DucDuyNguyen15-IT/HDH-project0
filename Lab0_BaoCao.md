# Lab 0 – System Call (xv6): Cần gì để chạy & ra kết quả

## 1. Tóm tắt đề

- **Nhiệm vụ duy nhất (10 điểm):** thêm system call `trace(int mask)` vào kernel xv6 (RISC-V) và viết chương trình user `user/trace.c`.
- `trace(mask)` bật tracing cho **tiến trình gọi nó**: mỗi khi một syscall có bit tương ứng trong `mask` sắp trả về, kernel in ra:
  ```
  syscall <tên> -> <giá trị trả về>
  ```
- Không được ảnh hưởng tiến trình khác.
- Làm nhóm tối đa 3 người, chấm theo **mức hiểu code khi phỏng vấn demo**. Bài giống nhau thì cả nhóm 0 điểm toàn bộ phần thực hành.
- Tài liệu gốc: https://pdos.csail.mit.edu/6.1810/2023/labs/syscall.html
- Cần đọc trước: xv6 book chương 2, mục 4.3 và 4.4.

## 2. Môi trường cần có

xv6 chạy trên **Linux** (dùng WSL trên Windows là ổn), cần các công cụ sau:

| Công cụ | Dùng để | Trạng thái trên máy bạn |
|---|---|---|
| WSL2 + Ubuntu | Môi trường Linux | ✅ Có (Ubuntu 26.04, WSL2) |
| `git` | Clone repo, chuyển branch, tạo file `.patch` | ✅ Có |
| `make`, `perl` | Build; `usys.pl` sinh ra `usys.S` | ✅ Có |
| `gcc-riscv64-linux-gnu` | Trình biên dịch chéo cho RISC-V | ❌ **Chưa có** |
| `qemu-system-misc` (`qemu-system-riscv64`) | Máy ảo chạy xv6 | ❌ **Chưa có** |
| `gdb-multiarch` | Debug kernel (không bắt buộc) | ❌ Chưa có |

### Cài đặt (gõ trong terminal Ubuntu/WSL)

```bash
sudo apt update
sudo apt install -y git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
```

Kiểm tra lại:

```bash
qemu-system-riscv64 --version
riscv64-linux-gnu-gcc --version
```

> **Lưu ý:** nên để mã nguồn trong ổ Linux (`~/...`), **không** đặt trong `/mnt/d/...`. Build trên ổ Windows chậm hơn nhiều và dễ lỗi quyền file hoặc ký tự xuống dòng CRLF. Khi xong có thể copy file nộp sang `D:\HK1-26-27\OS`.

## 3. Lấy mã nguồn

```bash
cd ~
git clone git://g.csail.mit.edu/xv6-labs-2023 xv6
# nếu cổng git:// bị chặn, dùng: https://github.com/mit-pdos/xv6-riscv (khác branch, ưu tiên link trên)
cd xv6
git fetch
git checkout syscall
make clean
make qemu        # chạy thử: thấy dấu nhắc "$" là OK
```

Thoát QEMU: bấm **Ctrl-A**, thả ra, rồi bấm **X**.

## 4. Các file phải sửa (theo đúng gợi ý trong đề)

| # | File | Việc cần làm |
|---|---|---|
| 1 | `Makefile` | Thêm `$U/_trace\` vào `UPROGS` |
| 2 | `user/trace.c` | **Tạo mới**: `argv[1]` là mask; gọi `trace(atoi(argv[1]))`, sau đó `exec(argv[2], &argv[2])` |
| 3 | `user/user.h` | Thêm prototype `int trace(int);` |
| 4 | `user/usys.pl` | Thêm `entry("trace");` |
| 5 | `kernel/syscall.h` | Thêm `#define SYS_trace 22` (số tiếp theo sau `SYS_close` = 21) |
| 6 | `kernel/proc.h` | Thêm trường `int trace_mask;` vào `struct proc` |
| 7 | `kernel/sysproc.c` | Viết `sys_trace()`: đọc mask bằng `argint(0, &mask)` rồi lưu vào `myproc()->trace_mask` |
| 8 | `kernel/syscall.c` | Thêm `extern uint64 sys_trace(void);`, thêm `[SYS_trace] sys_trace` vào bảng `syscalls[]`, thêm **mảng tên** `syscallnames[]`, sửa hàm `syscall()` để in kết quả |
| 9 | `kernel/proc.c` | *(Nên làm, theo đề gốc của MIT)*: trong `fork()` copy `trace_mask` sang tiến trình con. Trong `freeproc()`/`allocproc()` đặt lại về 0 |

### Ý tưởng sửa `syscall()` trong `kernel/syscall.c`

```c
num = p->trapframe->a7;
if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
  p->trapframe->a0 = syscalls[num]();          // giá trị trả về nằm trong a0
  if(p->trace_mask & (1 << num))
    printf("syscall %s -> %d\n", syscallnames[num], (int)p->trapframe->a0);
}
```

Trong đó `syscallnames[]` là mảng chuỗi được đánh chỉ số theo số syscall, ví dụ `[SYS_read] "read"`, ..., `[SYS_trace] "trace"`.

> Ở bản xv6-labs-2023, các hàm `sys_*` có kiểu trả về `uint64`, còn `argint()` trả về `void`. Đoạn code mẫu `int sys_hello(void)` trong đề chỉ để minh hoạ, khi viết thật phải theo đúng kiểu trong mã nguồn.

## 5. Chạy và kiểm tra kết quả

```bash
make qemu
```

Trong xv6, chạy:

```
$ trace 32 grep hello README
syscall read -> 1023
syscall read -> 966
syscall read -> 70
syscall read -> 0
$ trace 2147483647 grep hello README
syscall trace -> 0
syscall exec -> 3
syscall open -> 3
syscall read -> 1023
...
syscall close -> 0
$ grep hello README
$
```

Kết quả **đúng** khi output giống như trên:
- `32 = 1<<5` (chỉ trace `read`).
- `2147483647` bật toàn bộ 31 bit (trace mọi syscall).
- Lệnh không có `trace` thì không in gì.

Nếu muốn tự chấm, có thể chạy `./grade-lab-syscall trace`. Script này có sẵn trong repo của MIT.

## 6. Hồ sơ nộp lên Moodle

Nộp một file `MSSV1_MSSV2_MSSV3.zip`, MSSV sắp xếp tăng dần. Bên trong gồm:

1. **`MSSV1_MSSV2_MSSV3_Report.pdf`**: viết ngắn gọn, **không chép source code**. Nội dung gồm: em hiểu luồng syscall như thế nào, gặp khó khăn gì, đã thử cách nào mà không được.
2. **File patch**:
   ```bash
   git add user/trace.c        # file mới phải add thì mới xuất hiện trong diff
   git diff HEAD > MSSV1_MSSV2_MSSV3.patch
   ```
3. **Mã nguồn xv6 đã `make clean`**:
   ```bash
   make clean
   cd .. && zip -r xv6.zip xv6
   ```

## 7. Chuẩn bị cho phỏng vấn demo

Vì điểm dựa vào mức hiểu code, mỗi thành viên nên tự giải thích được các ý sau:

- Đường đi của một syscall: `trace()` ở user → stub trong `usys.S` (đặt số syscall vào `a7`, gọi lệnh `ecall`) → `uservec`/`usertrap` → `syscall()` → `sys_trace()` → giá trị trả về được ghi vào `trapframe->a0`.
- Tại sao kernel phải dùng `argint()` để đọc tham số, mà không đọc trực tiếp?
- Tại sao mask được lưu trong `struct proc`? (Để mỗi tiến trình có mask riêng, không ảnh hưởng tiến trình khác.)
- Tại sao dòng `syscall exec -> 3` in ra 3? (Đó là `argc` của `grep hello README`.)
- `usys.pl` sinh ra `usys.S` như thế nào?
