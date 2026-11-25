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