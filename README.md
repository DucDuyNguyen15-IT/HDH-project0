# Hệ Điều Hành - Lab 0: System Call Tracing (xv6 RISC-V)

Repository lưu trữ toàn bộ tài liệu hướng dẫn, giải thích lý thuyết, bộ câu hỏi ôn tập vấn đáp và file nộp bài cho Lab 0 (System Call `trace`) môn Hệ Điều Hành.

---

## 📂 Cấu trúc thư mục

- [`Lab0_BaoCao.md`](./Lab0_BaoCao.md): Tóm tắt yêu cầu, các bước thực hiện và checklist nộp bài.
- [`Lab0_GiaiThich.md`](./Lab0_GiaiThich.md): Giải thích chi tiết luồng xử lý và thay đổi mã nguồn kernel.
- [`Lab0_OnTapVanDap.md`](./Lab0_OnTapVanDap.md): Bộ câu hỏi & hướng dẫn trả lời phỏng vấn / vấn đáp demo.
- [`Lab0_OnTapVanDap.pdf`](./Lab0_OnTapVanDap.pdf): Bản PDF tài liệu ôn tập vấn đáp.
- [`Lab0_System_Call.pdf`](./Lab0_System_Call.pdf): Đề bài gốc Lab 0.
- `submit/`:
  - `xv6_source.zip`: Mã nguồn xv6 sau khi hoàn thành và `make clean`.
  - `xv6_trace.patch`: File patch chứa toàn bộ thay đổi cho tính năng syscall `trace`.

---

## 🛠️ Hướng dẫn kiểm tra và chạy thử

1. **Chuẩn bị môi trường (Ubuntu / WSL2):**
   ```bash
   sudo apt update
   sudo apt install -y git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
   ```

2. **Lấy mã nguồn gốc xv6:**
   ```bash
   git clone git://g.csail.mit.edu/xv6-labs-2023 xv6
   cd xv6
   git checkout syscall
   ```

3. **Áp dụng file patch:**
   ```bash
   git apply /đường_dẫn_tới/submit/xv6_trace.patch
   ```

4. **Biên dịch và chạy QEMU:**
   ```bash
   make qemu
   ```

5. **Kiểm tra kết quả trong xv6 shell:**
   ```bash
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
   ```

6. **Chấm điểm tự động:**
   ```bash
   ./grade-lab-syscall trace
   ```
