# Tuần 1 - Nền tảng Cortex-M3

## Ngày 1: Startup code (`startup_stm32f103c8tx.s`)

### Reset_Handler làm gì
`Reset_Handler` là hàm xử lý chạy đầu tiên khi chip reset (cấp nguồn hoặc bấm RESET). Các bước theo thứ tự:
1. Nạp `_estack` (địa chỉ cuối vùng RAM) vào `r0`, rồi `mov sp, r0` để đặt con trỏ ngăn xếp (stack pointer).
2. `bl SystemInit`: gọi hàm khởi tạo hệ thống ban đầu (đưa các thanh ghi clock về trạng thái mặc định).
3. Sao chép `.data` từ Flash sang RAM: `r0 = _sdata` (đầu `.data` trong RAM), `r1 = _edata` (cuối `.data` trong RAM), `r2 = _sidata` (bản gốc trong Flash), `r3 = 0` là chỉ số, rồi chạy vòng lặp `LoopCopyDataInit` chép từng 4 byte. Cần chép vì biến toàn cục có giá trị khởi tạo phải được lưu trong Flash để không mất khi tắt nguồn.
4. Điền 0 vào vùng `.bss`.
5. `bl main`: gọi hàm `main`.

### Câu 1: `bl main` nằm ở đâu?
Dòng 99, ngay sau đoạn `LoopFillZerobss` (điền 0 cho `.bss`).

### Câu 2: Bảng vector nằm ở đâu, phần tử đầu là gì?
Bảng vector là `g_pfnVectors` (dòng 132, sau `Default_Handler`). Trong chip, nó nằm ở đầu Flash, địa chỉ `0x08000000` (section `.isr_vector`).
- Ô nhớ ở `0x08000000` chứa giá trị `_estack` (địa chỉ cuối RAM), CPU nạp vào stack pointer.
- Ô nhớ ở `0x08000004` chứa địa chỉ của `Reset_Handler`, CPU nhảy tới đó.

### Câu 3: Vòng lặp điền 0 cho `.bss`
Điền số 0 từ `_sbss` (`r2`, đầu vùng `.bss` trong RAM) đến `_ebss` (`r4`, cuối vùng). Mỗi vòng ghi 0 vào 4 byte, tăng `r2` lên 4, lặp đến khi `r2` chạm `r4`. Vùng này chứa các biến toàn cục không gán giá trị ban đầu.

### Vì sao sau reset chip chạy vào `main()`?
- **Phần cứng:** khi reset, CPU tự đọc hai ô nhớ đầu Flash. Ô thứ nhất (`0x08000000`) chứa `_estack`, được nạp vào stack pointer. Ô thứ hai (`0x08000004`) chứa địa chỉ `Reset_Handler`, CPU nhảy tới đó.
- **Phần mềm:** `Reset_Handler` chạy `SystemInit`, chép `.data` từ Flash sang RAM, điền 0 cho `.bss`, rồi gọi `main`.

### 3 điều học được
1. Chip khởi động bằng hai giá trị đầu bảng vector, phần cứng tự làm trước khi có bất kỳ dòng code C nào.
2. `.data` và `.bss` phải được khởi tạo trước `main`, nên biến toàn cục luôn có giá trị đúng khi `main` chạy.
3. Cần phân biệt "ô nhớ ở địa chỉ X" với "giá trị nằm trong ô đó".

## Ngày 2: Linker script (`STM32F103C8TX_FLASH.ld`) và file `.map`

### Bộ nhớ của chip (khối `MEMORY`)
| Vùng | Địa chỉ bắt đầu | Dung lượng | Quyền |
|---|---|---|---|
| Flash | `0x08000000` | 64 KB | đọc, chạy lệnh (rx) |
| RAM | `0x20000000` | 20 KB | đọc, ghi, chạy lệnh (xrw) |

Lưu ý: file `.ld` ghi `0x8000000` (7 chữ số), giá trị vẫn là `0x08000000`.

### Stack nằm ở đâu?
`_estack = ORIGIN(RAM) + LENGTH(RAM)` = `0x20000000` + `0x5000` = `0x20005000`. Đây là địa chỉ ngay sau byte cuối của RAM. Stack chạy từ đỉnh này xuống thấp, mỗi lần đẩy dữ liệu vào thì `sp` giảm.

### `.isr_vector` đặt ở đâu, vì sao?
Đặt ở **Flash**, để bảng vector nằm đúng `0x08000000`. Khi reset, CPU tự đọc hai ô nhớ đầu Flash: ô đầu nạp vào stack pointer, ô thứ hai là địa chỉ `Reset_Handler`.

### `>RAM AT> FLASH` của `.data` nghĩa là gì?
- `>RAM`: lúc chương trình chạy, `.data` nằm ở RAM (từ `_sdata` đến `_edata`).
- `AT> FLASH`: lúc nạp vào chip, nội dung `.data` được cất trong Flash, địa chỉ cất là `_sidata`.
- Khi khởi động, startup chép từ `_sidata` (Flash) sang `_sdata` (RAM). Cần như vậy vì Flash không ghi trực tiếp được còn RAM mất dữ liệu khi tắt nguồn.

### Kết quả từ file `.map` và Build Console
| Mục | Giá trị |
|---|---|
| Địa chỉ `main` | `0x08000170` |
| Địa chỉ `Reset_Handler` | `0x08000178` |
| `text` | 564 byte (code và bảng vector, nằm trong Flash) |
| `data` | 0 byte |
| `bss` | 1568 byte |

`bss` lớn dù chưa có biến nào vì linker script chừa chỗ cho heap và stack trong RAM.

### Điều học được
1. Địa chỉ 32 bit luôn có 8 chữ số hex, đếm lại để tránh nhầm số 0 đầu.
2. `.data` có hai địa chỉ: nơi cất (Flash) và nơi chạy (RAM). Startup chép giữa hai nơi đó.
3. Dung lượng RAM 20 KB bị linker giữ chỗ cho heap và stack ngay từ đầu.