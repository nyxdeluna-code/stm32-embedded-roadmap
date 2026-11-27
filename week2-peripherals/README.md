# Tuần 2: Ngoại vi cơ bản

## Ngày 1: Nút nhấn và ngắt ngoài EXTI

### Phần cứng
Nút nhấn nối giữa PA0 và GND. PA0 ở chế độ input pull-up: thả nút thì mức 1, nhấn thì mức 0.

### Chuỗi đi của một lần nhấn
```
PA0 (cạnh xuống) → EXTI đường 0 (FTSR chọn cạnh, IMR cho phép) → NVIC (ISER0 bit 6) → CPU → EXTI0_IRQHandler
```

### Cấu hình
| Việc | Thanh ghi | Giá trị |
|---|---|---|
| PA0 input pull-up | `GPIOA_CRL` bit 3:0 | `0x8` (CNF = 10, MODE = 00) |
| Chọn pull-up | `GPIOA_ODR` bit 0 | 1 |
| Bật clock AFIO | `RCC_APB2ENR` bit 0 | 1 |
| Nối PA0 với đường EXTI0 | `AFIO_EXTICR1` bit 3:0 | `0000` |
| Kích hoạt ở cạnh xuống | `EXTI_FTSR` bit 0 | 1 |
| Cho phép ngắt đường 0 | `EXTI_IMR` bit 0 | 1 |
| Bật ngắt trong NVIC | `NVIC_ISER0` bit 6 | 1 (EXTI0 là IRQ 6) |

Thứ tự: cấu hình xong rồi bật NVIC ở dòng cuối.

### Câu hỏi và trả lời
1. **Polling và interrupt khác nhau thế nào?** Polling là CPU tự đọc chân liên tục trong vòng lặp, tốn chu kỳ khi chưa có sự kiện. Interrupt là phần cứng báo CPU khi có sự kiện, CPU rảnh làm việc khác hoặc ngủ.
2. **Vì sao phải xóa cờ pending trong ISR?** Chừng nào `EXTI_PR` còn bằng 1, khối EXTI còn giữ yêu cầu gửi lên NVIC, nên vừa thoát ISR là vào lại ngay.
3. **Nút nhấn nảy là gì?** Tiếp điểm kim loại bật nảy trong khoảng 1 đến 20 ms khi đóng hoặc mở, tạo nhiều cạnh nhiễu.

### Thí nghiệm nút nảy (Ngày 1)
Logic analyzer 8 MS/s, D0 = PC13 (LED), D1 = PA0 (nút).

| Bản code | Số lần nhấn | Số cạnh D0 | Cạnh thừa |
|---|---|---|---|
| Không chống dội | 20 | 36 (19 lên, 17 xuống) | 16 |
| `delay_ms(20)` trong ISR | 20 | 33 (17 lên, 16 xuống) | 13 |

- Nhiều xung D0 hẹp trùng lúc D1 ở mức thấp: LED đổi lúc nhấn rồi đổi lại lúc thả.
- Ảnh phóng to 0,27 ms mỗi pixel không thấy xung nhiễu, nhưng EXTI vẫn bắt được.
- Nhận định ban đầu (nhìn ảnh 20 s bằng mắt rồi cho rằng chống dội hiệu quả) là sai, số đếm cạnh mới đáng tin.

## Ngày 2 (Phần A): Chống dội bằng Timer

### Ý tưởng
`delay_ms(20)` trong ISR bắt CPU đứng đợi và chỉ che nảy lúc nhấn. Cách mới:
1. `EXTI0_IRQHandler`: xóa pending, chặn ngắt đường 0 (xóa bit 0 của `EXTI_IMR`), đặt `debounce_ms = 20`, rồi thoát ngay.
2. `TIM2_IRQHandler` chạy mỗi 1 ms: giảm `debounce_ms`. Khi về 0, **đọc lại mức PA0**: nếu còn thấp thì nhấn thật, đảo LED. Rồi xóa pending thừa và bật lại ngắt đường 0.

### Cấu hình TIM2 (ngắt mỗi 1 ms)
`f = f_clk / ((PSC + 1) × (ARR + 1))`. Clock của TIM2 là **72 MHz**: APB1 chạy 36 MHz, nhưng khi APB1 bị chia thì clock timer được nhân đôi trở lại.

| Việc | Thanh ghi, địa chỉ | Giá trị |
|---|---|---|
| Bật clock TIM2 | `RCC_APB1ENR` `0x4002101C` bit 0 | 1 |
| Chia tần | `TIM2_PSC` `0x40000028` | `72 - 1` (1 tick = 1 µs) |
| Đếm tối đa | `TIM2_ARR` `0x4000002C` | `1000 - 1` (1000 tick = 1 ms) |
| Cho phép ngắt update | `TIM2_DIER` `0x4000000C` bit 0 (`UIE`) | 1 |
| Bật bộ đếm | `TIM2_CR1` `0x40000000` bit 0 (`CEN`) | 1 |
| Cờ update | `TIM2_SR` `0x40000010` bit 0 (`UIF`) | xóa trong ISR |
| Bật trong NVIC | `NVIC_ISER0` bit 28 | 1 (TIM2 là IRQ 28) |
| Đọc mức PA0 | `GPIOA_IDR` `0x40010808` bit 0 | 0 là đang nhấn |

Kiểm tra: 72 MHz / (72 × 1000) = 1000 Hz, một ngắt mỗi 1 ms. `PSC` và `ARR` lưu giá trị trừ 1 vì bộ đếm chạy từ 0.

### Code ISR
```c
static volatile uint32_t debounce_ms = 0;

void EXTI0_IRQHandler(void)
{
    if (EXTI_PR & (1U << 0)) {
        EXTI_PR = (1U << 0);          /* xóa pending */
        EXTI_IMR &= ~(1U << 0);       /* chặn ngắt đường 0 */
        debounce_ms = 20U;            /* hẹn 20 ms */
    }
}

void TIM2_IRQHandler(void)
{
    if (TIM2_SR & (1U << 0)) {
        TIM2_SR &= ~(1U << 0);        /* xóa UIF */
        if (debounce_ms > 0U) {
            debounce_ms--;
            if (debounce_ms == 0U) {
                if (!(GPIOA_IDR & (1U << 0))) {
                    GPIOC_ODR ^= (1U << 13);   /* nhấn thật: đảo LED */
                }
                EXTI_PR = (1U << 0);           /* xóa nhiễu trong 20 ms */
                EXTI_IMR |= (1U << 0);         /* bật lại ngắt */
            }
        }
    }
}
```
Thứ tự an toàn: xóa `EXTI_PR` **trước** khi bật lại `EXTI_IMR`, nếu không cạnh nhiễu đã ghi nhớ sẽ gây ngắt ngay.

### Kết quả đo (20 lần nhấn, analyzer 8 MS/s)
| Bản code | Số cạnh D0 | Cạnh thừa |
|---|---|---|
| Không chống dội | 36 (19 lên, 17 xuống) | 16 |
| `delay_ms(20)` trong ISR | 33 (17 lên, 16 xuống) | 13 |
| **Timer, đọc lại mức sau 20 ms** | **20 (10 lên, 10 xuống)** | **0** |

Dự đoán trước khi đo: 10 cạnh lên và 10 cạnh xuống, đúng với kết quả.

### Vì sao cách này hiệu quả
- Đọc lại mức PA0 sau khi nút ổn định phân biệt được nhấn thật (mức thấp) với cạnh xuống nhiễu (mức cao, ví dụ nảy lúc thả).
- Che nảy ở cả hai cạnh. Kết quả ủng hộ giả thuyết ở Ngày 1 (nảy lúc thả gây đổi thừa), nhưng chưa phóng to trực tiếp cạnh thả nên chưa chứng minh được.
- ISR chỉ chạy vài lệnh rồi thoát, CPU không bị chặn.

### Giới hạn của thí nghiệm
- Chỉ một lần chạy 20 lần nhấn, với một nút và một ngưỡng 20 ms. Chưa đo thời gian nảy thật của nút.
- TIM2 ngắt mỗi 1 ms liên tục kể cả khi không có nút nhấn, tốn một ít CPU. Cải tiến: chỉ bật timer khi cần.

### Câu hỏi và trả lời
1. **Vì sao không dùng `delay_ms` trong ISR mà giao cho timer?** `delay_ms` bắt CPU đứng đợi 20 ms, vòng chính bị đóng băng và các ngắt ưu tiên thấp hơn bị chặn. Với timer, ISR chỉ hẹn giờ rồi thoát, CPU làm việc khác trong lúc đó.
2. **Vì sao đọc lại `GPIOA_IDR` sau 20 ms?** Sau khi nút hết nảy, mức PA0 ổn định cho biết nút đang thật sự bị nhấn hay không: thấp là nhấn thật, cao là cạnh ban đầu chỉ là nhiễu.
3. **`PSC` và `ARR` chọn thế nào?** `PSC + 1 = 72` đưa 72 MHz xuống 1 MHz (1 tick = 1 µs), `ARR + 1 = 1000` tick là 1 ms.

### Lỗi đã gặp
1. Nhầm `PSC` với tần số: ghi `PSC = 72 MHz`, thực tế `PSC` là số để chia và lưu giá trị trừ 1.
2. `debounce_ms = 10U` trong khi comment ghi 20 ms: cần đọc lại số trong code khớp với ghi chú.
3. Bật NVIC cho TIM2 trước khi cấu hình timer xong: nên đặt ở dòng

## Ngày 3: UART gửi dữ liệu (TX)

### Phần cứng
CP2102 chân RXD nối **PA9** (USART1 TX), nối GND chung. Chưa dùng chân RX. Terminal: 115200, 8 data bit, no parity, 1 stop bit, flow control none.

### Cấu hình USART1 (clock APB2 = 72 MHz)
| Việc | Thanh ghi, địa chỉ | Giá trị |
|---|---|---|
| Bật clock USART1 | `RCC_APB2ENR` bit 14 (`USART1EN`) | 1 |
| PA9 alternate function push-pull 50 MHz | `GPIOA_CRH` bit 7:4 | `0xB` |
| Tốc độ 115200 | `USART1_BRR` `0x40013808` | `0x271` |
| Bật USART và bộ gửi | `USART1_CR1` `0x4001380C` | `0x2008` (bit 13 `UE` + bit 3 `TE`) |
| Cờ trạng thái | `USART1_SR` `0x40013800` | bit 7 `TXE` |
| Thanh ghi dữ liệu | `USART1_DR` `0x40013804` | ghi ký tự |

`BRR = 72 000 000 / 115 200 = 625 = 0x271`. Phần nguyên `0x27` = 39, phần lẻ `0x1` = 1/16, nên 39,0625 và tốc độ thật = 72 MHz / (16 × 39,0625) = đúng 115 200 baud, không sai số.

### Code gửi
```c
static void uart_putc(char c)
{
    while (!(USART1_SR & (1U << 7))) { }   /* đợi TXE */
    USART1_DR = c;
}

static void uart_puts(const char *s)
{
    while (*s) {
        uart_putc(*s);
        s++;
    }
}
```
Vòng chính: `uart_puts("Hello STM32\r\n"); delay_ms(500);`

### Kết quả
- Terminal hiện `Hello STM32` mỗi 0,5 giây.
- Analyzer (Async Serial 115200 8N1) giải mã đúng 13 byte: `0x48 0x65 0x6C 0x6C 0x6F 0x20 0x53 0x54 0x4D 0x33 0x32 0x0D 0x0A`.

| Đại lượng | Tính | Đo |
|---|---|---|
| Một bit | 1 / 115200 = 8,68 µs | chưa đo trực tiếp |
| Một frame (10 bit) | 86,8 µs | 86,75 µs (khoảng cách giữa hai start liên tiếp) |
| Cả chuỗi 13 ký tự | 1,128 ms | 1,128 ms (1,0415 ms đến start cuối, cộng 86,8 µs) |
| Chu kỳ gửi chuỗi | khoảng 500 ms | 499,939 ms |

- Cột Duration 82,375 µs của Logic 2 không dùng làm thời gian frame (suy đoán là tính đến giữa stop bit, chưa kiểm chứng).
- 499,939 ms trùng với số đo SysTick ở Tuần 1 (lệch 0,061 ms): nguồn sai số nhiều khả năng là đồng hồ của thạch anh hoặc analyzer, khoảng 122 ppm.
- Suy luận từ số đo (chưa kiểm chứng): chu kỳ ngắn hơn 501,1 ms mong đợi vì `COUNTFLAG` của SysTick bật trong lúc gửi nên lần đợi đầu của `delay_ms` trả về ngay, nghĩa là `delay_ms(n)` có thể ngắn hơn tối đa 1 ms.

### Câu hỏi và trả lời
1. **Một frame 8N1 gồm những bit nào, mất bao lâu?** 10 bit: 1 start (mức 0) + 8 data + 1 stop (mức 1), mỗi bit 8,68 µs nên một frame 86,8 µs. Cả chuỗi 13 ký tự khoảng 1,13 ms.
2. **Vì sao đợi `TXE` trước khi ghi `DR`?** `TXE = 1` nghĩa là dữ liệu trước đã chuyển sang thanh ghi dịch, `DR` trống. Ghi khi còn bận thì đè mất ký tự trước. `TC` khác: báo toàn bộ frame đã ra khỏi chân.
3. **`BRR` tính thế nào?** `72 000 000 / 115 200 = 625 = 0x271`.

### Lỗi đã gặp
1. Câu 1 lần đầu: bỏ sót start bit và nhầm 498,8 ms (khoảng cách giữa hai lần gửi) với thời gian của frame.

## Lỗi đã gặp trong tuần (Ngày 1, 2)
| Lỗi | Vì sao sai | Quy tắc nhớ |
|---|---|---|
| `CNF` cho PA0 điền `0xF` thay vì `0x8` | `0xF` = output 50 MHz + alternate open-drain, không phải input | Đọc `MODE` và `CNF` từng cặp bit |
| `EXTI_IMR &= ~(0xFU << 0)` | `0xF` là 4 bit, tắt luôn đường 0 đến 3 | `1U` cho 1 bit, `0x7` cho 3 bit, `0xF` cho 4 bit |
| Gọi `debounce_ms(20)` như hàm trong khi nó là biến | Cùng một tên dùng hai kiểu | Khai báo biến `static volatile`, gán `debounce_ms = 20U` |
| Dòng xóa `UIF` nằm trong comment | Cờ không bao giờ được xóa, ISR chạy lặp, chip treo | Sau khi viết ISR, đọc lại xem dòng xóa cờ có thật chạy không |
| `GPIOC_ORD` | Gõ sai tên thanh ghi, trình biên dịch báo ngay | Đọc tên từng chữ cái |
| `^=` để bật lại `EXTI_IMR` | Nhận xét của AI cho rằng MISRA yêu cầu `\|=`, điều này không đúng. Lý do thật: `\|=` luôn đặt bit lên 1, `^=` đảo bit nên sai nếu bit đã là 1 | `\|=` để bật, `&= ~` để tắt, `^=` để đảo |
| `PSC = 72 MHz` | `PSC` là số chia, lưu giá trị trừ 1 | `PSC = 72 - 1` |
| `debounce_ms = 10U` trong khi ghi chú 20 ms | Số trong code và ghi chú lệch nhau | Đọc lại số trong code khớp với ghi chú |
| Dự đoán 1 ISR = 1 lần đổi LED, nhận định từ ảnh 20 s | Ảnh độ phân giải thấp không thấy nhiễu | Đếm cạnh bằng số liệu, không nhìn bằng mắt |
