# Lab 0 – System Call `trace`: Giải thích toàn bộ để ôn vấn đáp

> Mã nguồn đã làm nằm trong WSL: `~/xv6` (repo `xv6-labs-2023`, branch `syscall`).
> File nộp đã xuất ra `D:\HK1-26-27\OS\submit\` gồm `xv6_trace.patch` và `xv6_source.zip`. Nhớ đổi tên theo MSSV nhóm.

---

## 0. Bài toán tóm tắt

Viết system call `trace(mask)`. Sau khi một tiến trình gọi `trace(mask)`, mỗi lần tiến trình đó gọi một syscall có số hiệu `n` mà **bit thứ `n` của `mask` đang bật**, kernel sẽ in ra dòng sau ngay trước khi syscall trả về:

```
syscall <tên> -> <giá trị trả về>
```

Ví dụ với `trace 32 grep hello README`: `32 = 0b100000 = 1 << 5`, mà `SYS_read = 5`, nên chỉ in các lần gọi `read`.

---

## 1. Kiến thức nền: một system call đi qua những đâu?

Đây là câu hỏi **gần như chắc chắn gặp** khi vấn đáp. Bạn nên thuộc sơ đồ này:

```
 USER SPACE                                    KERNEL SPACE
 ──────────                                    ────────────
 trace.c:  trace(32)
    │  (gọi như hàm C bình thường, tham số 32 nằm trong thanh ghi a0)
    ▼
 usys.S (sinh ra từ usys.pl):
    li a7, SYS_trace     # a7 = 22 (số hiệu syscall)
    ecall                # lệnh "bẫy" chuyển sang chế độ kernel ──►  trampoline.S: uservec
                                                                     lưu toàn bộ thanh ghi user
                                                                     vào p->trapframe
                                                                        │
                                                                        ▼
                                                                     trap.c: usertrap()
                                                                     thấy scause = 8 (do ecall)
                                                                        │
                                                                        ▼
                                                                     syscall.c: syscall()
                                                                     num = trapframe->a7 (= 22)
                                                                     a0  = syscalls[22]()
                                                                        │
                                                                        ▼
                                                                     sysproc.c: sys_trace()
                                                                     argint(0) đọc trapframe->a0
                                                                     lưu vào p->trace_mask
                                                                     return 0
                                                                        │
    ◄──────────────────────────────── usertrapret() / userret ◄─────────┘
    ret                      (khôi phục thanh ghi, a0 = 0 là giá trị trả về)
```

**Các ý chính cần nói được:**
- User **không thể gọi thẳng** hàm trong kernel, vì hai bên khác không gian địa chỉ và khác mức đặc quyền. Phải dùng lệnh `ecall` để CPU chuyển sang chế độ kernel.
- **Quy ước thanh ghi của RISC-V:** số hiệu syscall đặt ở `a7`, các tham số ở `a0` đến `a5`, giá trị trả về được ghi lại vào `a0`.
- Khi vào kernel, các thanh ghi của user được cất vào **`p->trapframe`**, nên kernel đọc tham số từ đó.

---

## 2. Từng thay đổi: làm gì và vì sao

### 2.1 `user/user.h`: khai báo hàm cho user
```c
int trace(int);
```
**Vì sao:** để `trace.c` biên dịch được, vì C cần prototype của hàm. Đây chỉ là *khai báo*, còn phần thân hàm là đoạn assembly trong `usys.S`.

### 2.2 `user/usys.pl`: sinh "stub" để chuyển vào kernel
```perl
entry("trace");
```
**Vì sao:** khi chạy `make`, script Perl này sinh ra file `user/usys.S`. Với mỗi dòng `entry("x")`, nó tạo ra:
```asm
.global trace
trace:
 li a7, SYS_trace
 ecall
 ret
```
Đây chính là "cầu nối" từ user sang kernel. Thiếu nó thì bước link sẽ báo lỗi `undefined reference to trace`.

### 2.3 `kernel/syscall.h`: cấp số hiệu
```c
#define SYS_trace  22
```
**Vì sao là 22:** syscall cuối cùng có sẵn là `SYS_close = 21`. Số hiệu này vừa dùng làm **chỉ số của mảng** `syscalls[]`, vừa là **vị trí bit** trong mask. File này được include ở cả phía user (`usys.S`) lẫn phía kernel, nên hai bên dùng chung một con số.

### 2.4 `kernel/proc.h`: thêm biến lưu mask vào tiến trình
```c
struct proc {
  ...
  int trace_mask;   // Bitmask of syscalls to trace
};
```
**Vì sao lưu trong `struct proc`:** đề yêu cầu *"chỉ bật trace cho tiến trình gọi nó, không ảnh hưởng tiến trình khác"*. Mỗi tiến trình có một `struct proc` riêng, nên mỗi tiến trình có mask riêng. Nếu dùng biến toàn cục thì mọi tiến trình đều bị trace, như vậy là sai.

**Vì sao không cần khoá (`p->lock`):** chỉ chính tiến trình đó đọc và ghi mask của nó. Các trường như `name` hay `trapframe` cũng vậy, comment trong `proc.h` ghi là "private to the process".

### 2.5 `kernel/sysproc.c`: hàm xử lý `sys_trace`
```c
uint64
sys_trace(void)
{
  int mask;

  argint(0, &mask);
  myproc()->trace_mask = mask;
  return 0;
}
```
**Giải thích từng dòng:**
- `uint64 sys_trace(void)`: mọi hàm syscall trong xv6 đều **không nhận tham số kiểu C**, vì tham số nằm trong trapframe chứ không được truyền như hàm thường. Hàm trả về `uint64` vì giá trị sẽ được ghi vào thanh ghi 64-bit `a0`.
- `argint(0, &mask)`: lấy **tham số thứ 0**, tức là đọc `p->trapframe->a0`. Bên trong, `argint` gọi `argraw(0)`, mà `argraw(0)` trả về `p->trapframe->a0`.
- `myproc()`: trả về `struct proc` của tiến trình đang chạy trên CPU hiện tại.
- `return 0`: báo thành công. `syscall()` sẽ ghi 0 vào `a0`, nên phía user nhận được `trace()` trả về 0.

### 2.6 `kernel/syscall.c`: đăng ký và in kết quả (phần quan trọng nhất)

**(a) Khai báo và đăng ký vào bảng:**
```c
extern uint64 sys_trace(void);
static uint64 (*syscalls[])(void) = {
  ...
  [SYS_trace]   sys_trace,
};
```
`syscalls[]` là **mảng con trỏ hàm**, chỉ số chính là số hiệu syscall. Cú pháp `[SYS_trace] sys_trace` là *designated initializer* của C, dùng để gán đúng vào ô số 22.

**(b) Mảng tên để in:**
```c
static char *syscallnames[] = {
  [SYS_fork] "fork", [SYS_exit] "exit", ..., [SYS_trace] "trace",
};
```
**Vì sao cần:** kernel chỉ biết *số hiệu* (ví dụ 5), trong khi đề yêu cầu in *tên* ("read"). Mảng này dùng để tra từ số sang tên. Nó dùng cùng kiểu chỉ số với `syscalls[]` nên hai mảng không bị lệch nhau.

**(c) In sau khi syscall chạy xong:**
```c
num = p->trapframe->a7;
if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
  p->trapframe->a0 = syscalls[num]();
  if(p->trace_mask & (1 << num))
    printf("syscall %s -> %d\n", syscallnames[num], (int)p->trapframe->a0);
}
```
- `p->trace_mask & (1 << num)`: kiểm tra **bit thứ `num`** của mask có bật hay không.
- **Vì sao in *sau* khi gọi:** đề yêu cầu in giá trị trả về, mà giá trị này chỉ có sau khi `syscalls[num]()` chạy xong.
- **Vì sao ép kiểu `(int)`:** `a0` có kiểu `uint64`. Ép về `int` thì giá trị lỗi `-1` mới in ra là `-1`, thay vì một số rất lớn.
- **Vì sao đặt trong `syscall()`:** đây là **điểm chung duy nhất** mà mọi syscall đều đi qua. Chỉ cần sửa một chỗ, thay vì sửa cả 22 hàm.

### 2.7 `kernel/proc.c`: kế thừa và dọn dẹp mask
```c
// trong fork():
np->trace_mask = p->trace_mask;   // tiến trình con kế thừa mask của cha

// trong freeproc():
p->trace_mask = 0;                // dọn mask khi tiến trình kết thúc
```
**Vì sao `fork` phải copy mask:** theo đề gốc của MIT, khi chạy `trace 2 usertests forkforkfork` thì các tiến trình con cũng phải được trace. `fork` tạo ra một `struct proc` mới nhưng không tự copy các trường mới thêm vào, nên phải copy tay.

**Vì sao phải đặt lại về 0 trong `freeproc`:** các ô `struct proc` được **tái sử dụng**, vì `proc[NPROC]` là một mảng có kích thước cố định. Nếu không đặt lại, tiến trình mới chiếm ô cũ sẽ "dính" mask của tiến trình đã kết thúc và bị trace oan. Như vậy là vi phạm yêu cầu "không ảnh hưởng tiến trình khác".

### 2.8 `Makefile`: thêm chương trình `trace`
```make
UPROGS=\
  ...
  $U/_trace\
```
**Vì sao:** `UPROGS` là danh sách các chương trình sẽ được biên dịch và chép vào ổ đĩa ảo `fs.img`. Nếu không thêm, gõ `trace` trong xv6 sẽ báo `exec trace failed`.

### 2.9 `user/trace.c`: có sẵn trong repo, nhưng cần hiểu
```c
if (trace(atoi(argv[1])) < 0) { ... }        // bật trace với mask = argv[1]
for(i = 2; i < argc && i < MAXARG; i++)
  nargv[i-2] = argv[i];                       // dời argv: bỏ "trace" và mask
exec(nargv[0], nargv);                        // thay chính mình bằng chương trình cần trace
```
**Điểm mấu chốt:** `exec` **thay toàn bộ bộ nhớ** của tiến trình nhưng **giữ nguyên `struct proc`** (pid không đổi). Vì vậy `trace_mask` vẫn còn, và chương trình `grep` được trace mà không cần sửa gì trong `grep`.

---

## 3. Sửa lỗi môi trường (không thuộc đề, nhưng bắt buộc phải làm trên máy này)

Máy dùng **Ubuntu 26.04**, với bộ công cụ mới hơn nhiều so với thời điểm xv6 được viết, nên gặp 2 lỗi. Cả hai đã được sửa trong `Makefile`. **Nên ghi phần này vào report**, vì đề nói report là nơi ghi các vấn đề gặp phải.

| Lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| Khi build báo `usertests.c: incompatible pointer type` | GCC 15 mặc định dùng chuẩn **C23**. Trong chuẩn này, `void f()` nghĩa là "hàm không có tham số", nên không khớp kiểu với chỗ sử dụng | Thêm `-std=gnu17` vào `CFLAGS` |
| `make qemu` bị treo, không in ra gì | Trình biên dịch mặc định sinh mã cho **RVA23**, có các lệnh mới như lệnh nén `c.mul` (phần mở rộng Zcb) và lệnh vector. CPU ảo mặc định của QEMU không hỗ trợ các lệnh này, nên báo `illegal_instruction` ngay ở lệnh thứ 6 của `_entry` | Thêm `-march=rv64gc -mabi=lp64d` vào cả `CFLAGS` **và** `ASFLAGS`, vì các file `.S` không được biên dịch với `CFLAGS` |
| Không có lệnh `qemu-system-riscv64` | Ubuntu 26.04 đã tách RISC-V ra khỏi gói `qemu-system-misc` | `sudo apt install qemu-system-riscv` |

> Cách tìm ra lỗi thứ 2: chạy QEMU với tuỳ chọn `-d int,guest_errors`. Log báo `illegal_instruction, epc 0x80000010`. Dùng `objdump` xem lệnh tại địa chỉ đó thì thấy lệnh `mul` dạng 16-bit (`9d4d`), tức là một lệnh nén mới.

---

## 4. Kết quả chạy thực tế

```
$ trace 32 grep hello README
syscall read -> 1023
syscall read -> 961
syscall read -> 321
syscall read -> 0
$ trace 2147483647 grep hello README
syscall trace -> 0
syscall exec -> 3
syscall open -> 3
syscall read -> 1023
syscall read -> 961
syscall read -> 321
syscall read -> 0
syscall close -> 0
$ grep hello README
$ trace 2 usertests forkforkfork
syscall fork -> 7
syscall fork -> 8
...
```

- Số byte đọc được là `961/321`, khác với `966/70` trong PDF. Lý do là **file README trong repo khác bản giáo viên dùng**, không phải lỗi.
- Định dạng in theo **đúng PDF của lớp** (`syscall read -> 1023`). Script chấm `grade-lab-syscall` của MIT lại mong đợi định dạng có pid (`3: syscall read -> 1023`), nên nó sẽ báo FAIL. Điều này bình thường, vì lớp chấm theo PDF. Nếu giáo viên muốn có pid, chỉ cần đổi dòng `printf` thành `printf("%d: syscall %s -> %d\n", p->pid, ...)`.

---

## 5. Câu hỏi vấn đáp hay gặp và cách trả lời

**H: `exec -> 3` nghĩa là gì?**
Khi thành công, `exec` trả về `argc` của chương trình mới. Lệnh `grep hello README` có 3 đối số.

**H: `2147483647` là gì?**
Đó là `2^31 - 1`, tức là 31 bit thấp đều bằng 1. Vì vậy mọi syscall có số hiệu từ 0 đến 30 đều được trace.

**H: Vì sao dòng `syscall trace -> 0` vẫn xuất hiện, dù lúc gọi `trace` thì mask đang bằng 0?**
Vì việc in diễn ra **sau** khi `sys_trace()` chạy xong. Lúc đó mask đã được gán giá trị mới.

**H: Vì sao không thấy dòng `syscall exit`?**
`exit` không bao giờ trả về, vì tiến trình đã kết thúc. Do đó chương trình không chạy tới dòng `printf`.

**H: Vì sao không đọc tham số trực tiếp từ stack của user?**
Tham số nằm trong thanh ghi, và được cất vào `trapframe`. Với tham số là con trỏ, kernel phải dùng `copyin`/`copyout`, vì kernel và user dùng **page table khác nhau**. Kernel không thể dereference trực tiếp con trỏ của user.

**H: Thêm một syscall mới cần sửa những file nào?**
`user/user.h`, `user/usys.pl`, `kernel/syscall.h`, `kernel/syscall.c`, file chứa phần hiện thực (`sysproc.c` hoặc `sysfile.c`), và `Makefile` nếu có chương trình test.

**H: Nếu tiến trình A bật trace, tiến trình B có bị in không?**
Không, vì mask nằm trong `struct proc` của A. B chỉ bị in nếu B là tiến trình con được `fork` từ A sau khi A đã bật trace.

**H: `usys.S` ở đâu ra, sao không thấy trong git?**
File này được sinh tự động bằng lệnh `perl user/usys.pl > user/usys.S` khi chạy `make`.

**H: `1 << num` có vấn đề gì nếu `num >= 32` không?**
Có. Dịch bit vượt quá độ rộng của `int` là undefined behavior. Hiện xv6 chỉ có 22 syscall nên vẫn an toàn. Nếu sau này có nhiều syscall hơn thì dùng kiểu `uint64` và `1L << num`.

---

## 6. Việc bạn còn phải làm

1. Đổi tên các file trong `D:\HK1-26-27\OS\submit\`: `xv6_trace.patch` thành `MSSV1_MSSV2_MSSV3.patch`, và file zip mã nguồn cũng đặt tên theo quy định.
2. Viết report PDF ngắn, không chép code. Nội dung gồm: luồng đi của syscall (mục 1), tóm tắt cách làm (mục 2), các vấn đề đã gặp (mục 3).
3. Gộp cả 3 thứ vào file `MSSV1_MSSV2_MSSV3.zip` rồi nộp lên Moodle.
4. Tự chạy lại một lần cho quen tay: mở Ubuntu, chạy `cd ~/xv6 && make qemu`, gõ các lệnh ở mục 4, rồi thoát bằng `Ctrl-A` sau đó bấm `X`.
