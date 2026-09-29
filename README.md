---
Daniel Zakhari: b00104107
Hamad Siddiqi: b00101467
Asfa Ahmad: g00101137
---

# COE411L-02 Report 2

## Introduction

This report documents the design, hardware interfacing, and firmware implementation for Lab 3: Matrix Keypad Interfacing and I2C Character LCD Security System, and Lab 4: Pulse Width Modulation (PWM), RGB LED Color Synthesis, and Piezoelectric Tone Generation.

The hardware platform used throughout both laboratories is the STM32 Nucleo-L476RG development board, powered by an ARM Cortex-M4 microcontroller running at an 80 MHz system clock derived from the internal high-speed oscillator (HSI) and Phase-Locked Loop (PLL). Firmware development and peripheral initialization were conducted in STM32CubeIDE utilizing the Hardware Abstraction Layer (HAL).

In Lab 3, the microcontroller interfaces with a 4x4 matrix keypad and an HD44780-compatible 16x2 character LCD via an I2C expansion backpack (PCF8574). The system implements an access control state machine featuring dynamic character masking, buffer clearing, asynchronous serial telemetry transmission over USART2 at 115,200 baud, and credential validation against a predefined security code (`"A7A3"`).

In Lab 4, hardware timers (`TIM1` and `TIM3`) are configured in edge-aligned PWM Generation mode. The system demonstrates multi-channel PWM duty-cycle modulation across three independent timer channels (`TIM1_CH1`, `TIM1_CH2`, `TIM1_CH3`) to mix colors across the full 8-bit RGB color space. Additionally, dynamic auto-reload register (`ARR`) manipulation on `TIM3_CH1` is implemented to vary the output square-wave frequency, driving a piezoelectric buzzer to synthesize musical notes with fixed duty cycle upon an active-low pushbutton trigger (`PB0`).

---

## Description

### 1. Lab 3 System: Keypad Access Control & I2C LCD Display

The Lab 3 system implements a secure digital passcode authentication terminal composed of two primary interface subroutines:

* **Matrix Keypad Scanning & Real-Time Serial Echoing:** The MCU polls a 4x4 matrix keypad via `Keypad_Scan()`. When a valid key closure is registered, the character is encapsulated into an ASCII string buffer and transmitted over `USART2` (`115,200 baud, 8N1`) via `HAL_UART_Transmit` for remote terminal logging on TeraTerm.
* **Display Management & Passcode Validation:** The system interacts with a 16x2 character LCD through the `I2C1`/`I2C2` peripheral bus. Upon boot, the display prompts the user with `"ACCESS CODE: "`.
* As alphanumeric characters are keyed in, the firmware prints masking asterisks (`"*"`) on row 1 to hide sensitive input.
* Pressing the clear key (`'#'`) resets the input buffer, displays `"CLEARED"` for 3,000 ms, and returns to the base entry prompt.
* Once four characters are collected, the string is null-terminated and evaluated against the master credential (`"A7A3"`) using `strcmp()`. If verified, `"ACCESS GRANTED"` is displayed for 5,000 ms; otherwise, `"ACCESS DENIED"` is shown. The buffer and cursor index reset automatically following each validation cycle.

### 2. Lab 4 System: Hardware PWM, RGB Color Mixing, and Audio Synthesis

The Lab 4 system implements multi-peripheral timer control to generate variable duty cycles and variable frequencies:

* **RGB LED Color Space Mixing via TIM1:** Advanced-control timer `TIM1` is configured with a prescaler of 79 and an Auto-Reload Register (ARR) value of 999, yielding an operational counter clock of 1 MHz and a base PWM frequency of 1 kHz. Three capture/compare channels (`CH1`, `CH2`, `CH3`) drive the red, green, and blue cathodes/anodes. The helper function `SetColor(R, G, B)` maps standard 8-bit color intensities ($0\text{–}255$) proportionally to the timer counter period ($0\text{–}\text{ARR}$) via:

$$\text{CCR} = \frac{\text{Color} \times \text{ARR}}{255}$$

* **Dynamic Tone Synthesis via TIM3:** General-purpose timer `TIM3` generates acoustic frequencies by driving a piezoelectric transducer on Channel 1. To alter pitch, the `play_tone(freq, duration_ms)` function dynamically reconfigures `TIM3->ARR` according to:

$$\text{Period} = \frac{f_{\text{timer-clk}}}{f_{\text{target}}} = \frac{1{,}000{,}000}{\text{freq}}$$

The compare register (`CCR1`) is loaded with $\frac{\text{Period}}{4}$ to maintain a 25% duty cycle for acoustic efficiency. A 30 ms quiet delay with $\text{CCR1} = 0$ is inserted between notes to articulate note boundaries.
* **Pushbutton Execution Routine:** The microcontroller continuously monitors pin `PB0` (configured as a digital input with an internal pull-up resistor). When grounded via a tactile button press (`GPIO_PIN_RESET`), the MCU triggers a sequential cycle consisting of basic color cycling (Red, Green, Blue, Off) followed by a musical melody routine played through the buzzer.

---

## Circuit Diagram

### Lab 3

The CirkitDesigner schematic confirmation showing that the STM32 Nucleo-L476RG connected to the 4x4 matrix keypad and the I2C LCD module is correct:

![Cirkit Verification](/assets/lab_3_a/lab3-cirkitverification.png)


### Lab 4

The CirkitDesigner schematic diagram showing the STM32 Nucleo-L476RG connected to the RGB LED (TIM1 CH1-CH3), the piezoelectric buzzer (TIM3 CH1), and the external trigger pushbutton (PB0):

![Lab 4 CirkitDesigner Schematic](/assets/lab_4_a/lab4-cirkitdesigner.png)


---

## CubeMX (.ioc file) screenshots showing all relevant configurations

### Lab 3

The overall pinout setup for Lab 3:

![Lab 3 Pinout Configuration](/assets/lab_3_a/lab3-ioc.png)

The I2C configuration for I2C1 and I2C2:

![I2C Configuration](/assets/lab_3_a/lab3-ic2.png)

The GPIO matrix keypad row/column pin assignments:

![GPIO Configuration](/assets/lab_3_a/lab3-gpio.png)

The USART2 peripheral configuration [PA2 (TX) & PA3 (RX)]:

![USART2 GPIO Pin Configuration](/assets/lab_3_a/lab3-usart2-1.png)

![USART2 Parameter Settings](/assets/lab_3_a/lab3-usart2-2.png)

### Lab 4

The overall pinout setup for Lab 4:

![Lab 4 Pinout Configuration](/assets/lab_4_a/lab4-ioc.png)

The GPIO setup for PB0 (Input with Pull-Up):

![Lab 4 GPIO Configuration](/assets/lab_4_a/lab4-gpio.png)

The TIM1 PWM configuration (Channels 1, 2, 3 for RGB LED control):

![TIM1 Channel Parameter Configuration](/assets/lab_4_a/lab4-tim1-conf.png)

![TIM Pin Routing Configuration](/assets/lab_4_a/lab4-timx-1.png)

The TIM3 PWM configuration (Channel 1 for buzzer frequency modulation):

![TIM3 Counter and PWM Configuration](/assets/lab_4_a/lab4-tim3-conf.png)

The USART2 peripheral configuration:

![Lab 4 USART2 Pin Configuration](/assets/lab_4_a/lab4-usartx-1.png)


---

## Key Code Segments

### 1. Matrix Keypad Scanning, Buffer Population, and Serial Echoing (Lab 3)

The main operational loop polls the keypad matrix. When a valid press is detected, the key is echoed asynchronously over USART2, stored in an accumulation buffer, and visually masked on the second row of the LCD display using an asterisk (`"*"`).

```c
char key = Keypad_Scan();

if (key)
{
    if (key == '#') {
        attempt[0] = '\0';
        i = 0;
        LCD_Clear();
        LCD_Print("CLEARED");
        HAL_Delay(3000);
        LCD_Clear();
        LCD_Print("ACCESS CODE: ");
    }
    else {
        char out[16];
        attempt[i] = key;
        LCD_SetCursor(1, i);
        LCD_Print("*");
        i++;
        
        sprintf(out, "Key: %c\r\n", key);
        HAL_UART_Transmit(&huart2, (uint8_t*)out, strlen(out), HAL_MAX_DELAY);
        ...
    }
}

```

---

### 2. Passcode Evaluation and LCD State Feedback (Lab 3)

Once four characters have been received, the string is null-terminated and evaluated against the hardcoded access string (`"A7A3"`). The result is written to the LCD, held for 5,000 ms, and the buffer is cleared for subsequent attempts.

```c
if (i == 4)
{
    attempt[4] = '\0';
    if (strcmp(attempt, access) == 0) {
        LCD_Clear();
        LCD_SetCursor(0, 0);
        LCD_Print("ACCESS GRANTED");
        HAL_Delay(5000);
        LCD_Clear();
        LCD_Print("ACCESS CODE: ");
    }
    else {
        LCD_Clear();
        LCD_SetCursor(0, 0);
        LCD_Print("ACCESS DENIED");
        HAL_Delay(5000);
        LCD_Clear();
        LCD_Print("ACCESS CODE: ");
    }
    attempt[0] = '\0';
    i = 0;
}

```

---

### 3. Multi-Channel PWM RGB Color Space Mapping (Lab 4)

This function converts standard 8-bit RGB color parameters ($0\text{–}255$) to proportional capture/compare duty cycles against the auto-reload value of `TIM1`. This enables precise linear intensity control across all three channels.

```c
void SetColor(uint8_t red, uint8_t green, uint8_t blue)
{
    uint32_t arr = __HAL_TIM_GET_AUTORELOAD(&htim1);

    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, (red   * arr) / 255);
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_2, (green * arr) / 255);
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_3, (blue  * arr) / 255);
}

```

---

### 4. Dynamic Frequency Audio Synthesis via Timer Modulation (Lab 4)

To generate distinct acoustic frequencies on the passive buzzer, `play_tone` calculates the required period based on a 1 MHz timer count clock, modifies `TIM3->ARR` at runtime, sets a 25% duty cycle on Channel 1, and inserts a 30 ms quiet gap to articulate successive notes.

```c
void play_tone(uint32_t freq, uint32_t duration_ms) {
    uint32_t timer_clk = 1000000;  // 1 MHz based on 79 prescaler
    uint32_t period = timer_clk / freq;
    
    __HAL_TIM_SET_AUTORELOAD(&htim3, period - 1);
    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, period / 4);  // 25% duty cycle
    HAL_Delay(duration_ms);
    
    // Silence between notes
    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, 0);
    HAL_Delay(30);
}

```

---

### 5. Input Trigger and Coordinated Sequence Execution (Lab 4)

The application loop monitors active-low digital input pin `PB0`. Upon a detected button press, it triggers a sequenced RGB primary color test followed by musical tone reproduction using predetermined note frequencies (e.g., E5, G5, D5, B5).

```c
if (HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_0) == GPIO_PIN_RESET) {
    SetColor(255, 0, 0);
    HAL_Delay(500);
    SetColor(0, 255, 0);
    HAL_Delay(500);
    SetColor(0, 0, 255);
    HAL_Delay(500);
    SetColor(0, 0, 0);
    HAL_Delay(500);

    // Note sequence playback
    play_tone(659, 180);   // E5
    play_tone(659, 180);   // E5
    play_tone(784, 180);   // G5
    play_tone(659, 180);   // E5
    ...
}

```

---

## System Photo

### Lab 3

Physical hardware circuit wiring and interconnect prototype:

![Lab 3 Circuit Wiring Setup](/assets/lab_3_a/lab3-circuit-design.jpeg)

Idle state displaying the access code entry prompt on the I2C LCD:

![Idle Access Code Prompt](/assets/lab_3_a/lab3-coderequest.jpeg)

LCD display masking characters with asterisks during active passcode entry:

![Keypad Input Masked with Asterisks](/assets/lab_3_a/lab3-codestars.jpeg)

LCD display output showing the "ACCESS GRANTED" authentication state:

![Access Granted State](/assets/lab_3_a/lab3-granted.jpeg)

LCD display output showing the "ACCESS DENIED" state upon incorrect credential submission:

![Access Denied State](/assets/lab_3_a/lab3-denied.jpeg)

LCD display output showing the "CLEARED" state upon buffer reset:

![Buffer Cleared State](/assets/lab_3_a/lab3-cleared.jpeg)

TeraTerm terminal window capturing the asynchronous keypress telemetry stream:

![TeraTerm Serial Terminal Output](/assets/lab_3_a/lab3-teraterm.png)

Workbench capture showing live serial transmission in TeraTerm alongside firmware execution in STM32CubeIDE:

![TeraTerm Workbench Execution](/assets/lab_3_a/lab3-teraterm-pic.jpeg)

### Lab 4

#### 1. Hardware Interconnect & Trigger Execution

Physical breadboard hardware connection showing the Nucleo-L476RG, RGB LED, passive buzzer, and pushbutton:

![Lab 4 Hardware Breadboard Interconnect](/assets/lab_4_a/lab4-image4.jpeg)

Complete circuit operational test setup with RGB module and piezoelectric buzzer:

![Lab 4 Active Testbench Setup](/assets/lab_4_a/lab4-image1.jpeg)

User tactile pushbutton press on pin PB0 triggering the color sequence and tone routine:

![Pushbutton Trigger Execution on PB0](/assets/lab_4_a/lab4-image13.jpeg)

#### 2. Red PWM Generation (TIM1 Channel 1)

RGB LED driven to Red via PWM duty cycle on TIM1 Channel 1:

![RGB LED Driven Red - Close-Up](/assets/lab_4_a/lab4-image3.jpeg)

#### 3. Green PWM Generation (TIM1 Channel 2)

RGB LED driven to Green via PWM duty cycle on TIM1 Channel 2:

![RGB LED Driven Green - Close-Up](/assets/lab_4_a/lab4-image6.jpeg)

#### 4. Blue PWM Generation (TIM1 Channel 3)

RGB LED driven to Blue via PWM duty cycle on TIM1 Channel 3:

![RGB LED Driven Blue - Close-Up](/assets/lab_4_a/lab4-image2.jpeg)

## Getting Started & Build Instructions

### Prerequisites & Dependencies
* **IDE:** STM32CubeIDE (v1.12.0 or later)
* **Toolchain:** GNU Arm Embedded Toolchain (`arm-none-eabi-gcc`)
* **Terminal Emulator:** TeraTerm / PuTTY (115,200 baud, 8-N-1, CR+LF)

### Pinout Mapping Summary
| Peripheral | Pin | Mode / Configuration | Notes |
| :--- | :--- | :--- | :--- |
| **LCD SCL** | PB8 | I2C1_SCL (Pull-up) | 100 kHz Standard Mode |
| **LCD SDA** | PB9 | I2C1_SDA (Pull-up) | 100 kHz Standard Mode |
| **Keypad Rows (R1-R4)** | PC0 - PC3 | GPIO Output | Row driving |
| **Keypad Cols (C1-C4)**| PC4 - PC7 | GPIO Input (Pull-down) | Column detection |
| **Buzzer** | PA6 | TIM3_CH1 (PWM Generation)| Variable ARR frequency |
| **RGB LED (R, G, B)** | PA8, PA9, PA10 | TIM1_CH1, CH2, CH3 (PWM)| 1 kHz base frequency |
| **Trigger Pushbutton**| PB0 | GPIO Input (Internal Pull-Up)| Active-low trigger |
| **UART Telemetry** | PA2 (TX), PA3 (RX)| USART2 (115200 8N1) | Nucleo ST-LINK VCP |

### How to Build and Flash
1. Clone the repository:
   ```bash
   git clone <repo-url>
    ```
2. Open STM32CubeIDE and select **File > Open Projects from File System...**, then browse to the project directory.
3. Open `Core/Inc/keypad.h` and verify pin mappings if using custom board revisions.
4. Click **Project > Build All** (`Ctrl + B`).
5. Connect the STM32 Nucleo-L476RG board via USB.
6. Click **Run > Debug** (`F11`) or **Run** (`Ctrl + F11`) to flash the firmware.
7. Open TeraTerm connected to the ST-Link COM port at 115,200 baud to monitor serial output.
