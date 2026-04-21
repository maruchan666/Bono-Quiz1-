[main.c](https://github.com/user-attachments/files/26916754/main.c)

# Bono-Quiz1-#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "driver/timer.h"
#include "esp_timer.h"
#include "esp_adc/adc_cali_scheme.h"
#include "esp_adc/adc_cali.h"
#include "esp_adc/adc_oneshot.h"
#include <math.h>
#include "driver/ledc.h"

#define DIG1 15
#define DIG2 2
#define DIG3 13

#define A 18
#define B 19
#define C 21
#define D 22
#define E 23
#define F 5
#define G 4

#define LED_RED 27
#define LED_GREEN 14

#define Left_button 16
#define right_button 17
#define Left 33
#define right 26

#define potenciometro 34
#define SAMPLE_PERIOD_US 20000

// VARIABLES
static volatile int value1 = 0;
static volatile int value2 = 0;
static volatile int value3 = 0;
static volatile int direccion = 0;


static volatile int64_t antireizq = 0;
static volatile int64_t antireder = 0;

// Codigo para multiplexar en displays 7 segmentos
    const uint8_t digitos[10] = {
        0b00111111, // 0
        0b00000110, // 1
        0b01011011, // 2
        0b01001111, // 3
        0b01100110, // 4
        0b01101101, // 5
        0b01111101, // 6
        0b00000111, // 7
        0b01111111, // 8
        0b01101111  // 9
    };
    void mostrar_segmentos(uint8_t patron) {
        gpio_set_level(A, !((patron >> 0) & 1));
        gpio_set_level(B, !((patron >> 1) & 1));
        gpio_set_level(C, !((patron >> 2) & 1));
        gpio_set_level(D, !((patron >> 3) & 1));
        gpio_set_level(E, !((patron >> 4) & 1));
        gpio_set_level(F, !((patron >> 5) & 1));
        gpio_set_level(G, !((patron >> 6) & 1));
    }
    void multiplexar_display(int num1, int num2, int num3) {
        // Display 1
        gpio_set_level(DIG1, 0);
        gpio_set_level(DIG2, 0);
        gpio_set_level(DIG3, 0);
        mostrar_segmentos(digitos[num1]);
        gpio_set_level(DIG1, 1);
        vTaskDelay(pdMS_TO_TICKS(5));

        // Display 2
        gpio_set_level(DIG1, 0);
        gpio_set_level(DIG2, 0);
        gpio_set_level(DIG3, 0);
        mostrar_segmentos(digitos[num2]);
        gpio_set_level(DIG2, 1);
        vTaskDelay(pdMS_TO_TICKS(5));

        // Display 3
        gpio_set_level(DIG1, 0);
        gpio_set_level(DIG2, 0);
        gpio_set_level(DIG3, 0);
        mostrar_segmentos(digitos[num3]);
        gpio_set_level(DIG3, 1);
        vTaskDelay(pdMS_TO_TICKS(5));
    }

    static void IRAM_ATTR izquierda(void *arg){
    int64_t now = esp_timer_get_time();
    if (now - antireizq >200000){
        direccion=1;
    antireizq = now;
    }
    }

    static void IRAM_ATTR derecha(void *arg){
    int64_t now = esp_timer_get_time();
    if (now - antireder >200000){
        direccion=2;
    antireder = now;
    }
    }

void app_main() {

    gpio_set_level(LED_RED, 1);
            gpio_set_level(LED_GREEN, 0);
            gpio_set_level(Left, 1);
            gpio_set_level(right, 0);
            direccion = 0;

    /////// SALIDAS ///////
    gpio_config_t leds = {
        .pin_bit_mask = (
            (1ULL << A) |
            (1ULL << B) |
            (1ULL << C) |
            (1ULL << D) |
            (1ULL << E) |
            (1ULL << F) |
            (1ULL << G) |
            (1ULL << DIG1) |
            (1ULL << DIG2) |
            (1ULL << DIG3) |
            (1ULL << LED_RED) |
            (1ULL << LED_GREEN) |
            (1ULL << Left) |
            (1ULL << right)
        ),
        .mode = GPIO_MODE_OUTPUT,
        .pull_up_en = GPIO_PULLUP_DISABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type = GPIO_INTR_DISABLE
    };

    gpio_config(&leds);
    gpio_set_level(DIG1, 1);
    gpio_set_level(DIG2, 1);

    ////////Configuro las entrada
    gpio_config_t inputs = {
        .pin_bit_mask = (1ULL << Left_button | 1ULL << right_button),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type = GPIO_INTR_NEGEDGE
    };
    gpio_config(&inputs);
    gpio_install_isr_service(0); // habilitar interrupciones
    gpio_isr_handler_add(Left_button, izquierda, NULL);
    gpio_isr_handler_add(right_button, derecha, NULL);


 // TImer para el ADC
    ////// timer0
    timer_config_t timer_pro= {
        .divider = 80,
        .counter_dir = TIMER_COUNT_UP,
        .counter_en = TIMER_PAUSE,
        .alarm_en = TIMER_ALARM_DIS,
        .auto_reload = false,
    };

    timer_init(TIMER_GROUP_0, TIMER_0, &timer_pro);
    timer_set_counter_value(TIMER_GROUP_0,TIMER_0,0);
    timer_start(TIMER_GROUP_0, TIMER_0);
    uint64_t timer_value = 0;
    int adc_raw;


 // CONFIGURACIÓN DEL ADC
    adc_oneshot_unit_handle_t adc1_handle; /* Handle */ 

    adc_oneshot_unit_init_cfg_t init_config = {
        .unit_id = ADC_UNIT_1,
    };
    adc_oneshot_new_unit(&init_config, &adc1_handle);

    adc_oneshot_chan_cfg_t config = {
        .atten = ADC_ATTEN_DB_12,   // 0–3.3V
        .bitwidth = ADC_BITWIDTH_DEFAULT,
    };

    adc_oneshot_config_channel(adc1_handle, ADC_CHANNEL_6, &config);



  //PWM
    ledc_timer_config_t ledc_timer = {
    .speed_mode       = LEDC_LOW_SPEED_MODE,
    .timer_num        = LEDC_TIMER_0,
    .duty_resolution  = LEDC_TIMER_8_BIT,
    .freq_hz          = 5000,
    .clk_cfg          = LEDC_AUTO_CLK,
    };
    ledc_timer_config(&ledc_timer);

    ledc_channel_config_t ledc_channel = {

        .speed_mode = LEDC_LOW_SPEED_MODE,
        .channel    = LEDC_CHANNEL_0,
        .timer_sel  = LEDC_TIMER_0,
        .intr_type  = LEDC_INTR_DISABLE,
        .gpio_num   = 25,
        .duty       = 0, // Duty inicial
        .hpoint     = 0, // Punto de inicio del pulso: 0 (normal)
    };
    ledc_channel_config(&ledc_channel);
    
    while(1){
        multiplexar_display(value1, value2, value3);
        
        //Leo la frecuencia
        timer_get_counter_value(TIMER_GROUP_0, TIMER_0, &timer_value);

        if (timer_value >= SAMPLE_PERIOD_US) { //comparo el valor del timer con el periodo de muestreo

            adc_oneshot_read(adc1_handle, ADC_CHANNEL_6, &adc_raw); //lectura del valor del potenciometro

            int porcentaje = (adc_raw * 100) / 4095; //se hace la conversion a porcentaje

            value1 = porcentaje / 100;
            value2 = (porcentaje / 10) % 10;
            value3 = porcentaje % 10; 

            timer_set_counter_value(TIMER_GROUP_0, TIMER_0, 0); // se reinicia el timer para la siguiente lectura
            int duty = (porcentaje * 4095) / 100;

            ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, duty);
            ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
        }

        if (direccion == 1){
            ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 0);
            ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
            vTaskDelay(pdMS_TO_TICKS(500));

            gpio_set_level(LED_RED, 1);
            gpio_set_level(LED_GREEN, 0);
            gpio_set_level(Left, 1);
            gpio_set_level(right, 0);
            direccion = 0;

        } else if (direccion == 2) {
            ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 0);
            ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
            vTaskDelay(pdMS_TO_TICKS(500));

            gpio_set_level(LED_RED, 0);
            gpio_set_level(LED_GREEN, 1);
            gpio_set_level(Left, 0);
            gpio_set_level(right, 1);
            direccion = 0;
        }
    }

}
