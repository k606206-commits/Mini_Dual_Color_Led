
# Mini Dual Color LED Module Test - NUCLEO-F103RB

* 소형(3mm) 2색 LED 모듈을 STM32F103 NUCLEO 보드에서 PWM으로 제어하는 프로젝트입니다.

## 📌 개요

* 소형 2색 LED 모듈은 일반 5mm 2색 LED보다 작은 3mm 패키지를 사용하여 공간이 제한된 장치에서 상태 표시용으로 활용됩니다.
* Red와 Green LED를 독립적으로 제어하며, PWM을 통해 다양한 색상 표현이 가능합니다.


## 🛠 하드웨어 구성

### 필요 부품
| 부품 | 수량 | 비고 |
|------|------|------|
| NUCLEO-F103RB | 1 | STM32F103RB 탑재 |
| 소형 2색 LED 모듈 | 1 | KY-029 또는 3mm 2색 LED |
| 점퍼 와이어 | 3 | Female-Female |

### 핀 연결

<img width="396" height="360" alt="F103RB-pin" src="https://github.com/user-attachments/assets/18da491e-c129-4a8c-b8b2-d2ffaeaae9bb" />

```
Mini Dual Color LED     NUCLEO-F103RB
┌─────────────┐        ┌─────────────┐
│     R  ─────┼────────┤ PB1 (TIM3_CH4) (S)
│     G  ─────┼────────┤ PB0 (TIM3_CH3) (S)
│   GND  ─────┼────────┤ GND (-)
└─────────────┘        └─────────────┘
```

### 회로도

```
        ┌─────────────────────┐
        │  Mini 2-Color LED   │
        │                     │
PB0 ────┤ R    ┌─┐    G ──────┼──── PB1
        │      │ │            │
        │      └┬┘            │
        └───────┼─────────────┘
               GND
```

> ⚠️ **공통 애노드 타입 사용 시**: `COMMON_CATHODE`를 0으로 변경하고 GND 대신 3.3V에 연결

## 💻 소프트웨어

### 표현 가능한 색상

| 색상 | Red | Green | 용도 |
|------|-----|-------|------|
| OFF | 0 | 0 | 꺼짐 |
| RED | 255 | 0 | 오류, 정지 |
| GREEN | 0 | 255 | 정상, 완료 |
| YELLOW | 255 | 255 | 대기, 주의 |
| ORANGE | 255 | 100 | 경고 |
| LIME | 100 | 255 | 진행 중 |

### 시스템 상태 패턴

| 상태 | 색상 | 패턴 |
|------|------|------|
| OK | Green | 정상 점등 |
| BUSY | Yellow | 빠른 점멸 |
| WARNING | Orange | 느린 점멸 |
| ERROR | Red | 아주 빠른 점멸 |
| STANDBY | Green | 호흡 효과 |

### 주요 함수

```c
// 상태 설정
void MiniLED_SetState(LED_State_t state);

// Red/Green 개별 PWM 설정
void MiniLED_SetRGB(uint8_t red, uint8_t green);

// 펄스 효과
void MiniLED_Pulse(LED_State_t color, uint8_t count);

// 데모 함수
void MiniLED_BootSequence(void);      // 부팅 시퀀스
void MiniLED_StatusDemo(void);         // 상태 표시
void MiniLED_DataTransfer(void);       // 데이터 전송 시뮬레이션
void MiniLED_BatteryCharging(void);    // 충전 시뮬레이션
```

### PWM 설정

```c
Timer: TIM3
Prescaler: 63 (64MHz / 64 = 1MHz)
Period: 999 (1kHz PWM)
Channels: CH3(PB0)=Red, CH4(PB1)=Green
```

## 📂 프로젝트 구조

```
04_Mini_Dual_Color_LED/
├── main.c          # 메인 소스 코드
└── README.md       # 프로젝트 설명서
```

## 🔧 빌드 및 실행

### STM32CubeIDE 사용 시
1. 새 STM32 프로젝트 생성 (NUCLEO-F103RB 선택)
2. `main.c` 내용을 프로젝트에 복사
3. 빌드 후 보드에 플래시

```c
/* USER CODE BEGIN Header */
/**
  ******************************************************************************
  * @file           : main.c
  * @brief          : Main program body
  ******************************************************************************
  * @attention
  *
  * Copyright (c) 2026 STMicroelectronics.
  * All rights reserved.
  *
  * This software is licensed under terms that can be found in the LICENSE file
  * in the root directory of this software component.
  * If no LICENSE file comes with this software, it is provided AS-IS.
  *
  ******************************************************************************
  */
/* USER CODE END Header */
/* Includes ------------------------------------------------------------------*/
#include "main.h"

/* Private includes ----------------------------------------------------------*/
/* USER CODE BEGIN Includes */
#include "stm32f1xx_hal.h"
#include <string.h>
#include <stdio.h>
/* USER CODE END Includes */

/* Private typedef -----------------------------------------------------------*/
/* USER CODE BEGIN PTD */
/* Private defines */
#define RED_PIN         GPIO_PIN_0
#define GREEN_PIN       GPIO_PIN_1
#define LED_PORT        GPIOB
#define PWM_PERIOD      999

/* Common Type - 공통 캐소드/애노드 설정 */
#define COMMON_CATHODE  1   // 1: 공통 캐소드, 0: 공통 애노드

/* LED States */
typedef enum {
    LED_OFF = 0,
    LED_RED,
    LED_GREEN,
    LED_YELLOW,
    LED_ORANGE,
    LED_LIME
} LED_State_t;

/* System Status */
typedef enum {
    STATUS_OK = 0,
    STATUS_WARNING,
    STATUS_ERROR,
    STATUS_BUSY,
    STATUS_STANDBY
} SystemStatus_t;
/* USER CODE END PTD */

/* Private define ------------------------------------------------------------*/
/* USER CODE BEGIN PD */

/* USER CODE END PD */

/* Private macro -------------------------------------------------------------*/
/* USER CODE BEGIN PM */

/* USER CODE END PM */

/* Private variables ---------------------------------------------------------*/
TIM_HandleTypeDef htim3;

UART_HandleTypeDef huart2;

/* USER CODE BEGIN PV */

/* USER CODE END PV */

/* Private function prototypes -----------------------------------------------*/
void SystemClock_Config(void);
static void MX_GPIO_Init(void);
static void MX_USART2_UART_Init(void);
static void MX_TIM3_Init(void);
/* USER CODE BEGIN PFP */
void SystemClock_Config(void);
static void MX_GPIO_Init(void);
static void MX_TIM3_Init(void);
static void MX_USART2_UART_Init(void);
void MiniLED_SetState(LED_State_t state);
void MiniLED_SetRGB(uint8_t red, uint8_t green);
void MiniLED_Pulse(LED_State_t color, uint8_t count);
void MiniLED_StatusDemo(void);
void MiniLED_BootSequence(void);
void MiniLED_DataTransfer(void);
void MiniLED_BatteryCharging(void);

/* UART printf 리다이렉션 */
int __io_putchar(int ch) {
    HAL_UART_Transmit(&huart2, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
/* USER CODE END PFP */

/* Private user code ---------------------------------------------------------*/
/* USER CODE BEGIN 0 */
void MiniLED_SetState(LED_State_t state)
{
    switch (state) {
        case LED_OFF:    MiniLED_SetRGB(0, 0);       break;
        case LED_RED:    MiniLED_SetRGB(255, 0);     break;
        case LED_GREEN:  MiniLED_SetRGB(0, 255);     break;
        case LED_YELLOW: MiniLED_SetRGB(255, 255);   break;
        case LED_ORANGE: MiniLED_SetRGB(255, 100);   break;
        case LED_LIME:   MiniLED_SetRGB(100, 255);   break;
    }
}

void MiniLED_SetRGB(uint8_t red, uint8_t green)
{
#if COMMON_CATHODE
    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_3, (red * PWM_PERIOD) / 255);
    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_4, (green * PWM_PERIOD) / 255);
#else
    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_3, PWM_PERIOD - (red * PWM_PERIOD) / 255);
    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_4, PWM_PERIOD - (green * PWM_PERIOD) / 255);
#endif
}

void MiniLED_Pulse(LED_State_t color, uint8_t count)
{
    for (uint8_t i = 0; i < count; i++) {
        for (int b = 0; b <= 255; b += 15) {
            switch (color) {
                case LED_RED:    MiniLED_SetRGB(b, 0); break;
                case LED_GREEN:  MiniLED_SetRGB(0, b); break;
                case LED_YELLOW: MiniLED_SetRGB(b, b); break;
                default: break;
            }
            HAL_Delay(5);
        }
        for (int b = 255; b >= 0; b -= 15) {
            switch (color) {
                case LED_RED:    MiniLED_SetRGB(b, 0); break;
                case LED_GREEN:  MiniLED_SetRGB(0, b); break;
                case LED_YELLOW: MiniLED_SetRGB(b, b); break;
                default: break;
            }
            HAL_Delay(5);
        }
        HAL_Delay(150);
    }
}

void MiniLED_BootSequence(void)
{
    for (int i = 0; i < 6; i++) {
        MiniLED_SetState(LED_RED);
        HAL_Delay(80);
        MiniLED_SetState(LED_GREEN);
        HAL_Delay(80);
    }

    for (int b = 0; b <= 255; b += 5) {
        MiniLED_SetRGB(b, b);
        HAL_Delay(8);
    }
    HAL_Delay(200);

    for (int r = 255; r >= 0; r -= 5) {
        MiniLED_SetRGB(r, 255);
        HAL_Delay(8);
    }

    for (int i = 0; i < 2; i++) {
        MiniLED_SetState(LED_OFF);
        HAL_Delay(150);
        MiniLED_SetState(LED_GREEN);
        HAL_Delay(150);
    }

    MiniLED_SetState(LED_OFF);
    printf("  Boot complete!\r\n");
}

void MiniLED_StatusDemo(void)
{
    printf("  Status: OK\r\n");
    MiniLED_SetState(LED_GREEN);
    HAL_Delay(2000);

    printf("  Status: BUSY\r\n");
    for (int j = 0; j < 8; j++) {
        MiniLED_SetState(LED_YELLOW);
        HAL_Delay(150);
        MiniLED_SetState(LED_OFF);
        HAL_Delay(150);
    }

    printf("  Status: WARNING\r\n");
    for (int j = 0; j < 4; j++) {
        MiniLED_SetState(LED_ORANGE);
        HAL_Delay(300);
        MiniLED_SetState(LED_OFF);
        HAL_Delay(300);
    }

    printf("  Status: ERROR\r\n");
    for (int j = 0; j < 10; j++) {
        MiniLED_SetState(LED_RED);
        HAL_Delay(100);
        MiniLED_SetState(LED_OFF);
        HAL_Delay(100);
    }

    printf("  Status: STANDBY\r\n");
    for (int cycle = 0; cycle < 2; cycle++) {
        for (int b = 0; b <= 200; b += 5) {
            MiniLED_SetRGB(0, b);
            HAL_Delay(10);
        }
        for (int b = 200; b >= 0; b -= 5) {
            MiniLED_SetRGB(0, b);
            HAL_Delay(10);
        }
    }

    MiniLED_SetState(LED_OFF);
}

void MiniLED_DataTransfer(void)
{
    printf("  Connecting...\r\n");
    for (int i = 0; i < 6; i++) {
        MiniLED_SetState(LED_YELLOW);
        HAL_Delay(100);
        MiniLED_SetState(LED_OFF);
        HAL_Delay(100);
    }

    printf("  Transferring data...\r\n");
    for (int i = 0; i < 30; i++) {
        MiniLED_SetState(LED_GREEN);
        HAL_Delay(30 + (i % 5) * 20);
        MiniLED_SetState(LED_OFF);
        HAL_Delay(20 + (i % 3) * 15);
    }

    printf("  Transfer complete!\r\n");
    MiniLED_SetState(LED_GREEN);
    HAL_Delay(500);

    for (int b = 255; b >= 0; b -= 5) {
        MiniLED_SetRGB(0, b);
        HAL_Delay(15);
    }
}

void MiniLED_BatteryCharging(void)
{
    printf("  Charging: ");

    for (int level = 0; level <= 100; level += 20) {
        printf("%d%% ", level);

        if (level < 20) {
            for (int i = 0; i < 2; i++) {
                MiniLED_SetState(LED_RED);
                HAL_Delay(100);
                MiniLED_SetState(LED_OFF);
                HAL_Delay(100);
            }
        } else if (level < 50) {
            for (int b = 100; b <= 255; b += 20) {
                MiniLED_SetRGB(b, b * 40 / 100);
                HAL_Delay(15);
            }
            for (int b = 255; b >= 100; b -= 20) {
                MiniLED_SetRGB(b, b * 40 / 100);
                HAL_Delay(15);
            }
        } else if (level < 80) {
            for (int b = 150; b <= 255; b += 15) {
                MiniLED_SetRGB(b, b);
                HAL_Delay(12);
            }
            for (int b = 255; b >= 150; b -= 15) {
                MiniLED_SetRGB(b, b);
                HAL_Delay(12);
            }
        } else {
            for (int b = 200; b <= 255; b += 10) {
                MiniLED_SetRGB(0, b);
                HAL_Delay(15);
            }
            for (int b = 255; b >= 200; b -= 10) {
                MiniLED_SetRGB(0, b);
                HAL_Delay(15);
            }
        }
    }

    printf("\r\n  Fully charged!\r\n");
    MiniLED_SetState(LED_GREEN);
    HAL_Delay(1500);
}
/* USER CODE END 0 */

/**
  * @brief  The application entry point.
  * @retval int
  */
int main(void)
{

  /* USER CODE BEGIN 1 */

  /* USER CODE END 1 */

  /* MCU Configuration--------------------------------------------------------*/

  /* Reset of all peripherals, Initializes the Flash interface and the Systick. */
  HAL_Init();

  /* USER CODE BEGIN Init */

  /* USER CODE END Init */

  /* Configure the system clock */
  SystemClock_Config();

  /* USER CODE BEGIN SysInit */

  /* USER CODE END SysInit */

  /* Initialize all configured peripherals */
  MX_GPIO_Init();
  MX_USART2_UART_Init();
  MX_TIM3_Init();
  /* USER CODE BEGIN 2 */
    HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_3);
    HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_4);

    printf("\r\n================================================\r\n");
    printf("  Mini Dual Color LED Module Test - NUCLEO-F103RB\r\n");
    printf("================================================\r\n\n");

    printf("[Boot] Starting...\r\n");
    MiniLED_BootSequence();
    HAL_Delay(500);
  /* USER CODE END 2 */

  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
      while (1)
      {
          printf("\r\n[Test 1] Basic Colors\r\n");

          printf("  OFF\r\n");
          MiniLED_SetState(LED_OFF);
          HAL_Delay(800);

          printf("  RED\r\n");
          MiniLED_SetState(LED_RED);
          HAL_Delay(800);

          printf("  GREEN\r\n");
          MiniLED_SetState(LED_GREEN);
          HAL_Delay(800);

          printf("  YELLOW\r\n");
          MiniLED_SetState(LED_YELLOW);
          HAL_Delay(800);

          printf("  ORANGE\r\n");
          MiniLED_SetState(LED_ORANGE);
          HAL_Delay(800);

          printf("  LIME\r\n");
          MiniLED_SetState(LED_LIME);
          HAL_Delay(800);

          MiniLED_SetState(LED_OFF);
          HAL_Delay(500);

          printf("\r\n[Test 2] Pulse Patterns\r\n");

          printf("  Red pulse x3\r\n");
          MiniLED_Pulse(LED_RED, 3);
          HAL_Delay(500);

          printf("  Green pulse x3\r\n");
          MiniLED_Pulse(LED_GREEN, 3);
          HAL_Delay(500);

          printf("  Yellow pulse x2\r\n");
          MiniLED_Pulse(LED_YELLOW, 2);
          HAL_Delay(500);

          printf("\r\n[Test 3] System Status Indicators\r\n");
          MiniLED_StatusDemo();
          HAL_Delay(500);

          printf("\r\n[Test 4] Data Transfer Simulation\r\n");
          MiniLED_DataTransfer();
          HAL_Delay(500);

          printf("\r\n[Test 5] Battery Charging Simulation\r\n");
          MiniLED_BatteryCharging();

          MiniLED_SetState(LED_OFF);

          printf("\r\n--- Cycle Complete ---\r\n");
          HAL_Delay(2000);
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
}

/**
  * @brief System Clock Configuration
  * @retval None
  */
void SystemClock_Config(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  /** Initializes the RCC Oscillators according to the specified parameters
  * in the RCC_OscInitTypeDef structure.
  */
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSI;
  RCC_OscInitStruct.HSIState = RCC_HSI_ON;
  RCC_OscInitStruct.HSICalibrationValue = RCC_HSICALIBRATION_DEFAULT;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSI_DIV2;
  RCC_OscInitStruct.PLL.PLLMUL = RCC_PLL_MUL16;
  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  /** Initializes the CPU, AHB and APB buses clocks
  */
  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
                              |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_2) != HAL_OK)
  {
    Error_Handler();
  }
}

/**
  * @brief TIM3 Initialization Function
  * @param None
  * @retval None
  */
static void MX_TIM3_Init(void)
{

  /* USER CODE BEGIN TIM3_Init 0 */

  /* USER CODE END TIM3_Init 0 */

  TIM_ClockConfigTypeDef sClockSourceConfig = {0};
  TIM_MasterConfigTypeDef sMasterConfig = {0};
  TIM_OC_InitTypeDef sConfigOC = {0};

  /* USER CODE BEGIN TIM3_Init 1 */

  /* USER CODE END TIM3_Init 1 */
  htim3.Instance = TIM3;
  htim3.Init.Prescaler = 63;
  htim3.Init.CounterMode = TIM_COUNTERMODE_UP;
  htim3.Init.Period = 999;
  htim3.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
  htim3.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_ENABLE;
  if (HAL_TIM_Base_Init(&htim3) != HAL_OK)
  {
    Error_Handler();
  }
  sClockSourceConfig.ClockSource = TIM_CLOCKSOURCE_INTERNAL;
  if (HAL_TIM_ConfigClockSource(&htim3, &sClockSourceConfig) != HAL_OK)
  {
    Error_Handler();
  }
  if (HAL_TIM_PWM_Init(&htim3) != HAL_OK)
  {
    Error_Handler();
  }
  sMasterConfig.MasterOutputTrigger = TIM_TRGO_RESET;
  sMasterConfig.MasterSlaveMode = TIM_MASTERSLAVEMODE_DISABLE;
  if (HAL_TIMEx_MasterConfigSynchronization(&htim3, &sMasterConfig) != HAL_OK)
  {
    Error_Handler();
  }
  sConfigOC.OCMode = TIM_OCMODE_PWM1;
  sConfigOC.Pulse = 0;
  sConfigOC.OCPolarity = TIM_OCPOLARITY_HIGH;
  sConfigOC.OCFastMode = TIM_OCFAST_DISABLE;
  if (HAL_TIM_PWM_ConfigChannel(&htim3, &sConfigOC, TIM_CHANNEL_3) != HAL_OK)
  {
    Error_Handler();
  }
  if (HAL_TIM_PWM_ConfigChannel(&htim3, &sConfigOC, TIM_CHANNEL_4) != HAL_OK)
  {
    Error_Handler();
  }
  /* USER CODE BEGIN TIM3_Init 2 */

  /* USER CODE END TIM3_Init 2 */
  HAL_TIM_MspPostInit(&htim3);

}

/**
  * @brief USART2 Initialization Function
  * @param None
  * @retval None
  */
static void MX_USART2_UART_Init(void)
{

  /* USER CODE BEGIN USART2_Init 0 */

  /* USER CODE END USART2_Init 0 */

  /* USER CODE BEGIN USART2_Init 1 */

  /* USER CODE END USART2_Init 1 */
  huart2.Instance = USART2;
  huart2.Init.BaudRate = 115200;
  huart2.Init.WordLength = UART_WORDLENGTH_8B;
  huart2.Init.StopBits = UART_STOPBITS_1;
  huart2.Init.Parity = UART_PARITY_NONE;
  huart2.Init.Mode = UART_MODE_TX_RX;
  huart2.Init.HwFlowCtl = UART_HWCONTROL_NONE;
  huart2.Init.OverSampling = UART_OVERSAMPLING_16;
  if (HAL_UART_Init(&huart2) != HAL_OK)
  {
    Error_Handler();
  }
  /* USER CODE BEGIN USART2_Init 2 */

  /* USER CODE END USART2_Init 2 */

}

/**
  * @brief GPIO Initialization Function
  * @param None
  * @retval None
  */
static void MX_GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  /* USER CODE BEGIN MX_GPIO_Init_1 */

  /* USER CODE END MX_GPIO_Init_1 */

  /* GPIO Ports Clock Enable */
  __HAL_RCC_GPIOC_CLK_ENABLE();
  __HAL_RCC_GPIOD_CLK_ENABLE();
  __HAL_RCC_GPIOA_CLK_ENABLE();
  __HAL_RCC_GPIOB_CLK_ENABLE();

  /*Configure GPIO pin Output Level */
  HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);

  /*Configure GPIO pin : B1_Pin */
  GPIO_InitStruct.Pin = B1_Pin;
  GPIO_InitStruct.Mode = GPIO_MODE_IT_RISING;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  HAL_GPIO_Init(B1_GPIO_Port, &GPIO_InitStruct);

  /*Configure GPIO pin : LD2_Pin */
  GPIO_InitStruct.Pin = LD2_Pin;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(LD2_GPIO_Port, &GPIO_InitStruct);

  /* EXTI interrupt init*/
  HAL_NVIC_SetPriority(EXTI15_10_IRQn, 0, 0);
  HAL_NVIC_EnableIRQ(EXTI15_10_IRQn);

  /* USER CODE BEGIN MX_GPIO_Init_2 */

  /* USER CODE END MX_GPIO_Init_2 */
}

/* USER CODE BEGIN 4 */

/* USER CODE END 4 */

/**
  * @brief  This function is executed in case of error occurrence.
  * @retval None
  */
void Error_Handler(void)
{
  /* USER CODE BEGIN Error_Handler_Debug */
  /* User can add his own implementation to report the HAL error return state */
  __disable_irq();
  while (1)
  {
  }
  /* USER CODE END Error_Handler_Debug */
}

#ifdef  USE_FULL_ASSERT
/**
  * @brief  Reports the name of the source file and the source line number
  *         where the assert_param error has occurred.
  * @param  file: pointer to the source file name
  * @param  line: assert_param error line source number
  * @retval None
  */
void assert_failed(uint8_t *file, uint32_t line)
{
  /* USER CODE BEGIN 6 */
  /* User can add his own implementation to report the file name and line number,
     ex: printf("Wrong parameters value: file %s on line %d\r\n", file, line) */
  /* USER CODE END 6 */
}
#endif /* USE_FULL_ASSERT */

```
## 
## 📊 시리얼 출력 예시
<img width="642" height="837" alt="image" src="https://github.com/user-attachments/assets/b4c056e7-36e6-41a3-a58e-8fc1fda0c030" />

## 이미지

1. Red/Green 빠른 교대 점멸 (6회)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; : <img width="642" height="837" alt="image" src="https://github.com/user-attachments/assets/90b146d3-5e61-4672-bc61-c9de44b79660" /><br>
2. Yellow 페이드 >Yellow > Green 2번깜빡임:<img width="642" height="837" alt="image" src="https://github.com/user-attachments/assets/d18ee01c-9bb7-4e91-8a20-cb9dad82fe21" />





## 📝 데모 패턴 상세

### 부팅 시퀀스
```
1. Red/Green 빠른 교대 점멸 (6회)
2. Yellow 페이드 인
3. Yellow → Green 전환 (부팅 완료)
4. Green 2번 깜빡임 (확인)
```

### 배터리 충전 시뮬레이션
```
0-20%:  Red 빠른 점멸 (위험)
20-50%: Orange 호흡 (충전 중)
50-80%: Yellow 호흡 (충전 중)
80-100%: Green 호흡 (거의 완료)
100%:   Green 정상 점등 (완료)
```

### 데이터 전송 시뮬레이션
```
1. Yellow 점멸 (연결 중)
2. Green 불규칙 점멸 (데이터 전송)
3. Green 페이드 아웃 (완료)
```

## 🔍 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|------|------|----------|
| LED가 켜지지 않음 | 배선 오류 | 핀 연결 확인 |
| 색상이 반대 | 공통 타입 오류 | COMMON_CATHODE 설정 변경 |
| Yellow가 주황색 | LED 특성 차이 | Green 밝기 증가 |
| 밝기가 약함 | 전류 부족 | 저항값 확인 |

## 💡 응용 예제

### IoT 연결 상태 표시기
```c
void ShowConnectionStatus(uint8_t status) {
    switch (status) {
        case 0: // 연결 끊김
            MiniLED_SetState(LED_RED);
            break;
        case 1: // 연결 시도 중
            MiniLED_Pulse(LED_YELLOW, 1);
            break;
        case 2: // 연결됨
            MiniLED_SetState(LED_GREEN);
            break;
    }
}
```

### 간단한 레벨 미터
```c
void ShowLevel(uint8_t percent) {
    if (percent < 25)      MiniLED_SetState(LED_RED);
    else if (percent < 50) MiniLED_SetState(LED_ORANGE);
    else if (percent < 75) MiniLED_SetState(LED_YELLOW);
    else                   MiniLED_SetState(LED_GREEN);
}
```

## 📚 참고 자료

- [STM32F103 Reference Manual](https://www.st.com/resource/en/reference_manual/rm0008-stm32f101xx-stm32f102xx-stm32f103xx-stm32f105xx-and-stm32f107xx-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)
- [KY-029 Mini Dual Color LED Module](https://arduinomodules.info/ky-029-3mm-two-color-led-module/)

## 📜 라이선스

MIT License
