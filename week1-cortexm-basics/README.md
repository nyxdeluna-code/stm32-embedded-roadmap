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

## Ngày 3: Blink PC13 bằng thanh ghi (`w1_blink_reg`)

### Cấu hình
- Bật clock GPIOC: bit 4 (`IOPCEN`) của `RCC_APB2ENR`.
- PC13 thuộc `GPIOC_CRH`, chiếm bit 20-23: xóa 4 bit rồi đặt `MODE = 10` (output 2 MHz), `CNF = 00` (push-pull).
- Điều khiển bằng `GPIOC_BSRR`: bit 29 (`BR13`) kéo PC13 xuống Low (LED sáng), bit 13 (`BS13`) đưa PC13 lên High (LED tắt).

### Code chính
```c
SET_BIT(RCC_APB2ENR, 4);
GPIOC_CRH &= ~(0xFU << 20);
GPIOC_CRH |=  (0x2U << 20);
for (;;) {
    GPIOC_BSRR = (1U << 29);   /* LED sáng */
    delay(500000);
    GPIOC_BSRR = (1U << 13);   /* LED tắt */
    delay(500000);
}
```

### Cách nạp
[Điền: nạp qua CP2102 + bootloader UART (BOOT0 = 1) hay SWD, và cách khắc phục lỗi "Activating device: KO" nếu có.]

### Kết quả đo (logic analyzer, chip chạy HSI 8 MHz)
| `delay(n)` | Nửa chu kỳ | Chu kỳ |
|---|---|---|
| 500000 | 436,2 ms | 872,4 ms |
| 1000000 | 872,5 ms | 1,745 s |

Tăng gấp đôi `n` thì nửa chu kỳ tăng gấp đôi, xác nhận code chạy là code của mình. Tính ngược: 0,436 s × 8 MHz ≈ 3,49 triệu chu kỳ chia 500 000 vòng ≈ **7 chu kỳ mỗi vòng `while`**. Chưa ghi dự đoán trước khi nạp. Sai lệch nhỏ còn lại do HSI không chính xác (sai số vài phần trăm).

### Câu hỏi và trả lời
1. **Vì sao bật clock GPIOC trước?** Sau reset clock ngoại vi bị tắt để tiết kiệm điện, chưa bật thì ghi vào thanh ghi không có tác dụng.
2. **PC13 thuộc CRL hay CRH?** CRH, bit 20-23 (chân 8-15 thuộc CRH, chân n nằm ở bit `(n-8)×4`).
3. **Ghi 1 vào bit 29 của BSRR?** LED sáng: bit 29 là `BR13` kéo PC13 xuống Low, LED nối từ 3.3V nên có dòng chạy qua.
4. **Vì sao `delay` dùng `volatile`?** Không có `volatile`, compiler thấy vòng lặp rỗng không có tác dụng và có thể xóa cả vòng lặp (khi tối ưu), làm delay biến mất.
5. **Kết quả so với dự đoán?** Chưa có dự đoán trước. Tính ra khoảng 7 chu kỳ mỗi vòng, hợp lý vì `n` là `volatile` nên mỗi vòng phải đọc RAM, trừ, ghi lại, so sánh, nhảy.

### Điều học được
1. Chân GPIO chỉ hoạt động sau khi bật clock của cổng đó.
2. Trước khi đặt giá trị mới vào một nhóm bit, phải xóa nhóm bit đó, các bit khác không đụng tới.
3. Số đo thay đổi theo đúng tỉ lệ khi đổi tham số là cách kiểm tra chương trình đang chạy chính là code của mình.

## Ngày 4: Clock tree lên 72 MHz (`w1_clock72`)

### Mục tiêu
Chuyển chip từ HSI 8 MHz sang 72 MHz bằng thanh ghi: HSE 8 MHz → PLL ×9 → SYSCLK.

### Bảng bit
| Việc | Thanh ghi, bit | Giá trị |
|---|---|---|
| Bật HSE, đợi `HSERDY` | `RCC_CR` bit 16, đợi bit 17 | 1 |
| Flash wait states | `FLASH_ACR` bit 2:0 (`LATENCY`) | `010` (2 wait states) |
| Prefetch | `FLASH_ACR` bit 4 | 1 |
| Nguồn PLL = HSE | `RCC_CFGR` bit 16 (`PLLSRC`) | 1 |
| Hệ số nhân ×9 | `RCC_CFGR` bit 21:18 (`PLLMUL`) | `0111` |
| APB1 chia 2 | `RCC_CFGR` bit 10:8 (`PPRE1`) | `100` |
| Bật PLL, đợi `PLLRDY` | `RCC_CR` bit 24, đợi bit 25 | 1 |
| SYSCLK = PLL, đợi `SWS` | `RCC_CFGR` bit 1:0 (`SW`), đọc bit 3:2 | `10` |

### Thứ tự bắt buộc
HSE → đợi `HSERDY` → Flash wait states → cấu hình PLL và APB1 → bật PLL → đợi `PLLRDY` → `SW = PLL` → đợi `SWS`.

- Flash đặt wait state **trước khi tăng clock**: Flash đọc chậm hơn CPU, ở 72 MHz cần 2 wait states, nếu không CPU đọc lệnh sai và chip treo.
- Không sửa cấu hình PLL khi PLL đang chạy.
- Phải đợi cờ `RDY` trước khi dùng vì phần cứng cần thời gian ổn định.
- APB1 chia 2 vì ngoại vi APB1 chịu tối đa 36 MHz.

### Lỗi đã gặp
Điền nhầm địa chỉ thanh ghi (`0x40021004`) vào chỗ cần điền giá trị trường bit (`0x7` cho PLLMUL, `0x4` cho PPRE1). Giá trị đặt vào luôn là số nhỏ của trường bit, không phải địa chỉ.

### Kết quả đo (logic analyzer, nửa chu kỳ blink PC13)
| Clock | Wait state | `delay(n)` | Nửa chu kỳ | Chu kỳ máy mỗi vòng | Thời gian mỗi vòng |
|---|---|---|---|---|---|
| 8 MHz (HSI) | 0 | 500000 | 436,2 ms | ≈ 7 | 0,87 µs |
| 8 MHz (HSI) | 2 | 100000 | 112,2 ms | ≈ 9 | 1,12 µs |
| 72 MHz (PLL) | 2 | 1000000 | 125,0 ms | ≈ 9 | 0,125 µs |

- Cùng code vòng lặp và cùng 2 wait states, thời gian mỗi vòng giảm từ 1,12 µs xuống 0,125 µs, tức giảm **đúng 9 lần**, khớp với 72 / 8.
- Chỉ bật wait states (clock vẫn 8 MHz) đã làm vòng lặp chậm từ 7 lên 9 chu kỳ mỗi vòng (+29%), vì mỗi lần nhảy ngược CPU phải đọc lại Flash.
- Bằng chứng này là **gián tiếp**: giả định số chu kỳ mỗi vòng không đổi. Kiểm chứng trực tiếp bằng chân MCO ở Ngày 5.
- Chưa ghi dự đoán trước khi nạp.

### Điều học được
1. Muốn đổi tần số phải dựng chuỗi HSE → PLL → SYSCLK, không có thanh ghi nào nhận trực tiếp số MHz.
2. Thứ tự cấu hình quan trọng: Flash trước, PLL sau, đổi nguồn SYSCLK cuối cùng.
3. Đo thời gian bằng vòng `delay` không đáng tin vì phụ thuộc cấu hình Flash, cần SysTick hoặc đo clock trực tiếp.