# Bai6
Bài 6 - Giao tiếp I²C
Yêu cầu của bài tập

Cấu hình STM32 là master I²C

Giao tiếp với một EEPROM hoặc cảm biến I²C để đọc/ghi dữ liệu.

Hiển thị dữ liệu đọc được lên màn hình terminal qua UART.
1. Phần khai báo & include
#include "stm32f10x.h"
#include "stm32f10x_rcc.h"
#include "stm32f10x_gpio.h"
#include "stm32f10x_usart.h"
#include "stm32f10x_i2c.h"
#include <string.h>
#include <stdio.h>


Đây là các header của thư viện Standard Peripheral Library cho STM32F1. Nó cung cấp hàm cấu hình RCC, GPIO, USART, I2C…

<string.h> và <stdio.h> để dùng strlen, sprintf …

#define EEPROM_ADDRESS 0xA0


Địa chỉ I2C của chip EEPROM AT24C256 khi A0-A2 nối GND là 0xA0 (ghi).

2. Cấu hình ngoại vi
2.1. GPIO_Config
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);


Bật clock cho port A.

PA9: USART1_TX, chế độ Alternate Function Push-Pull, tốc độ 50MHz
PA10: USART1_RX, chế độ Input Floating.

2.2. USART_Config

Bật clock USART1.

Cấu hình USART1 tốc độ 9600, 8 bit, 1 stop bit, no parity.

Kích hoạt USART1 (USART_Cmd).

2.3. I2C_Config

Bật clock GPIOB và I2C1.

PB6 = SCL, PB7 = SDA, chế độ Alternate Function Open-Drain.

Cấu hình I2C1: 100kHz, 7-bit address, enable ACK.

3. Các hàm làm việc với EEPROM
3.1 Ghi 1 byte
void EEPROM_WriteByte(uint16_t addr, uint8_t data)


Trình tự:

Chờ bus I2C rảnh.

START

Gửi địa chỉ slave (0xA0) mode Transmitter.

Gửi 2 byte địa chỉ bộ nhớ trong EEPROM (High trước, Low sau).

Gửi dữ liệu.

STOP.

3.2 Đọc 1 byte
uint8_t EEPROM_ReadByte(uint16_t addr)


Trình tự:

Chờ bus rảnh.

START – gửi địa chỉ chip ở mode ghi.

Gửi 2 byte địa chỉ bộ nhớ muốn đọc.

Lặp lại START (Repeated START).

Gửi địa chỉ chip ở mode đọc.

Đọc 1 byte từ EEPROM.

Disable ACK, STOP.

3.3 Ghi nhiều byte
void EEPROM_WriteBuffer(uint16_t addr, uint8_t *buf, uint16_t len)


Giống ghi 1 byte nhưng sau khi gửi địa chỉ ô nhớ thì gửi liên tiếp các byte trong buf.

Lưu ý AT24C256 có page write limit (64 bytes/page). Code này chưa xử lý vụ “cắt trang” – nếu len > 64 thì có thể lỗi.

3.4 Đọc nhiều byte
void EEPROM_ReadBuffer(uint16_t addr, uint8_t *buf, uint16_t len)


Gửi địa chỉ trước, repeated START, chuyển sang mode nhận, sau đó đọc len byte.

4. Hàm hỗ trợ USART & Delay
4.1 Gửi chuỗi USART
void USART_SendString(char *str)


Lặp qua từng ký tự, chờ cờ TXE (transmit empty), rồi gửi.

4.2 Delay bằng SysTick
void DelayMs(uint32_t ms)


Cấu hình SysTick 1ms.

Đặt biến TimingDelay = ms, chờ giảm về 0.

Trong ngắt SysTick_Handler giảm TimingDelay.

5. Phần main
SystemInit();
GPIO_Config();
USART_Config();
I2C_Config();


Khởi tạo hệ thống và ngoại vi.

USART_SendString("Test EEPROM AT24C256 via I2C\r\n");


In ra PC để báo bắt đầu.

Test ghi 1 byte:

EEPROM_WriteByte(0x0010, 'X');
DelayMs(10);
uint8_t data = EEPROM_ReadByte(0x0010);


Ghi ký tự ‘X’ vào ô nhớ 0x10.

Đợi 10ms (EEPROM cần thời gian ghi).

Đọc lại byte vừa ghi.

Test ghi nhiều byte:

uint8_t msg[] = "Hello EEPROM 24C256!";
EEPROM_WriteBuffer(0x0020, msg, strlen((char*)msg));
DelayMs(20);
EEPROM_ReadBuffer(0x0020, readbuf, strlen((char*)msg));


Ghi chuỗi vào EEPROM từ địa chỉ 0x20.

Đọc lại vào readbuf và in ra.

Cuối cùng, while(1){} – vòng lặp vô hạn.

6. Ghi chú quan trọng

AT24C256 ghi theo trang (64 byte). Code trên ghi buffer chưa xử lý vượt trang – nếu dữ liệu > 64 byte nên tách ra từng trang.

Sau mỗi lần ghi nên Delay ≥ 5ms để EEPROM hoàn tất chu kỳ ghi.

Địa chỉ 0xA0 là địa chỉ ghi, đọc vẫn dùng 0xA0 vì hàm I2C_Send7bitAddress thêm bit R/W cho bạn.
