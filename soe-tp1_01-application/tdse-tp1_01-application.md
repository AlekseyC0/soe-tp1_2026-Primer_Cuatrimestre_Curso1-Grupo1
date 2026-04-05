# Análisis de la Secuencia de Inicio (Boot) y `SystemCoreClock`

Analizar este paso es fundamental porque nos permite ver qué ocurre "bajo el capó" del microcontrolador antes de que nuestra aplicación siquiera empiece a correr, cumpliendo con el objetivo de analizar y explicar el código fuente.

El viaje de nuestro **STM32** arranca físicamente en el archivo en código ensamblador `startup_stm32f103rbtx.s`, específicamente en la rutina llamada `Reset_Handler`. Este es el primerísimo código que se ejecuta cuando le damos energía a la placa o tocamos el botón de reset físico.

## inicialización

Cuando le damos energía a la placa o apretamos el botón de reset, el hardware (el procesador Cortex-M3) busca la dirección de una etiqueta específica llamada `Reset_Handler` y empieza a ejecutar las instrucciones en código ensamblador desde ahí.

Si mirás dentro del bloque `Reset_Handler` en ese archivo, las primeras líneas se encargan de algo aburrido pero vital: preparar la memoria RAM (copian el valor inicial de las variables y ponen en cero el resto).

```assembly
  .syntax unified
  .cpu cortex-m3
  .fpu softvfp
  .thumb

.global g_pfnVectors
.global Default_Handler

/* start address for the initialization values of the .data section.
defined in linker script */
.word _sidata
/* start address for the .data section. defined in linker script */
.word _sdata
/* end address for the .data section. defined in linker script */
.word _edata
/* start address for the .bss section. defined in linker script */
.word _sbss
/* end address for the .bss section. defined in linker script */
.word _ebss

.equ  BootRAM, 0xF108F85F
/**
 * @brief  This is the code that gets called when the processor first
 *          starts execution following a reset event. Only the absolutely
 *          necessary set is performed, after which the application
 *          supplied main() routine is called.
 * @param  None
 * @retval : None
*/


```

Pero justo después de eso, ocurre la magia. Hay dos llamadas a funciones fundamentales usando la instrucción ensambladora `bl` (Branch with Link, que significa "saltar a esta función y guardar el camino de vuelta"). La secuencia es esta:

```assembly
  .section .text.Reset_Handler
  .weak Reset_Handler
  .type Reset_Handler, %function
Reset_Handler:

/* Call the clock system initialization function.*/
    bl  SystemInit

/* Copy the data segment initializers from flash to SRAM */
  ldr r0, =_sdata
  ldr r1, =_edata
  ldr r2, =_sidata
  movs r3, #0
  b LoopCopyDataInit

CopyDataInit:
  ldr r4, [r2, r3]
  str r4, [r0, r3]
  adds r3, r3, #4

LoopCopyDataInit:
  adds r4, r0, r3
  cmp r4, r1
  bcc CopyDataInit
  
/* Zero fill the bss segment. */
  ldr r2, =_sbss
  ldr r4, =_ebss
  movs r3, #0
  b LoopFillZerobss

FillZerobss:
  str  r3, [r2]
  adds r2, r2, #4

LoopFillZerobss:
  cmp r2, r4
  bcc FillZerobss

/* Call static constructors */
    bl __libc_init_array
/* Call the application's entry point.*/
  bl main
  bx lr
.size Reset_Handler, .-Reset_Handler

```

`bl SystemInit`: Esta es la primera vez que se toca el reloj del sistema. Esta función (provista por STMicroelectronics) configura el microcontrolador de forma muy básica y segura. Enciende el oscilador interno (HSI) que funciona a 8 MHz. En este instante exacto, la variable global SystemCoreClock toma el valor de 8000000. ¡El microcontrolador ya está latiendo!


`bl main`: Una vez que el sistema tiene un reloj básico, el ensamblador nos "catapulta" hacia nuestro código en C.

¡Ahora aterrizamos en el archivo main.c!

El procesador entra en la función `main()`. Si mirás las primeras líneas de código dentro de tu `main()`, vas a notar que antes de crear tareas o llegar al famoso `while(1)`, hay llamadas a funciones de la librería `HAL` de ST (Hardware Abstraction Layer) que terminan de configurar todo el hardware a su máxima velocidad.


```c
int main(void)
{
  #if (1 == LOGGER_CONFIG_USE_SEMIHOSTING)
  initialise_monitor_handles();
  #endif

  HAL_Init();
  SystemClock_Config();
  
  MX_GPIO_Init();
  MX_USART2_UART_Init();
  MX_TIM2_Init();
  
  HAL_TIM_Base_Start_IT(&htim2);
  app_init();

  #ifdef _defaultTask_
  osThreadDef(defaultTask, StartDefaultTask, osPriorityNormal, 0, 128);
  defaultTaskHandle = osThreadCreate(osThread(defaultTask), NULL);
  #endif

  osKernelStart();

  while (1)
  {
  }
}
```

1.  **`HAL_Init()`:** Esta es la primera función que se ejecuta. Su trabajo principal es resetear todos los periféricos a un estado seguro y, de forma muy importante, **configurar el SysTick por primera vez**. En este momento, el procesador sigue corriendo con el reloj básico de 8 MHz (el que configuró `SystemInit` en el paso anterior). Por lo tanto, el SysTick se configura para interrumpir cada 1 milisegundo basándose en esos 8 MHz.
2.  **`SystemClock_Config()`:** Aquí ocurre el gran salto de velocidad. Esta función toma ese reloj básico de 8 MHz y utiliza un circuito interno llamado PLL (Phase-Locked Loop) para multiplicarlo. Para tu NUCLEO-F103RB, típicamente eleva la velocidad al máximo permitido: **72 MHz**. 

Es exactamente en este punto (dentro de `SystemClock_Config()`) donde la variable **`SystemCoreClock`** se actualiza y pasa de valer `8000000` a valer `72000000`. Además, como el procesador ahora va mucho más rápido, esta función vuelve a reconfigurar el SysTick para que siga interrumpiendo exactamente cada 1 milisegundo, pero ahora contando a la nueva velocidad de 72 MHz.

Resumiendo hasta ahora:
* **Inicio (`Reset_Handler` -> `SystemInit`):** `SystemCoreClock` = 8 MHz.
* **Entrada al `main` (`HAL_Init`):** El SysTick arranca usando los 8 MHz.
* **Aceleración (`SystemClock_Config`):** `SystemCoreClock` salta a 72 MHz y el SysTick se ajusta a esta nueva velocidad.


Aquí hay un conflicto clásico en los sistemas embebidos. Por defecto, la librería HAL de STMicroelectronics "es dueña" del SysTick y lo usa para sus funciones de retardo (como `HAL_Delay`). Pero, **FreeRTOS también necesita usar el SysTick de forma exclusiva** para su "Tick" del sistema operativo y poder hacer el cambio de tareas (Time Slicing). ¡No pueden usarlo los dos al mismo tiempo sin pisarse!

Para solucionar esto, STM32CubeIDE nos obliga a cambiar la fuente de tiempo de la HAL a otro Timer de hardware distinto, dejándole el SysTick libre a FreeRTOS. A continuacion se desarrollo con detalle en codigo de qué ocurre:


## Evolución de SystemCoreClock y SysTick

El código nos muestra claramente las tres etapas de inicialización:

**Etapa 1: El Arranque Básico (`HAL_Init()`)**
Al entrar a `int main(void)`, la primera gran función que se llama es `HAL_Init()`.
* **¿Qué pasa con SysTick?** Como mencioné antes, en este momento el procesador corre a la velocidad "por defecto" (típicamente 8 MHz desde el oscilador interno). `HAL_Init()` configura el SysTick para generar una interrupción cada 1 milisegundo basándose en esos 8 MHz iniciales.

```c
HAL_StatusTypeDef HAL_Init(void)
{
  /* Configure Flash prefetch */
#if (PREFETCH_ENABLE != 0)
#if defined(STM32F101x6) || defined(STM32F101xB) || defined(STM32F101xE) || defined(STM32F101xG) || \
    defined(STM32F102x6) || defined(STM32F102xB) || \
    defined(STM32F103x6) || defined(STM32F103xB) || defined(STM32F103xE) || defined(STM32F103xG) || \
    defined(STM32F105xC) || defined(STM32F107xC)

  /* Prefetch buffer is not available on value line devices */
  __HAL_FLASH_PREFETCH_BUFFER_ENABLE();
#endif
#endif /* PREFETCH_ENABLE */

  /* Set Interrupt Group Priority */
  HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4);

  /* Use systick as time base source and configure 1ms tick (default clock after Reset is HSI) */
  HAL_InitTick(TICK_INT_PRIORITY);

  /* Init the low level hardware */
  HAL_MspInit();

  /* Return function status */
  return HAL_OK;
}
```

**Etapa 2: Aceleración (`SystemClock_Config()`)**
Inmediatamente después, se llama a `SystemClock_Config()`. Si mirás el código de esta función, hace dos cosas clave:
1.  **Configura el multiplicador PLL:** Activa el PLL (`RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;`) para multiplicar el reloj interno.
2.  **Aplica la nueva velocidad:** Asigna este nuevo reloj ultrarrápido (el PLL) como la fuente principal del sistema (`RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;`).
* **¿Qué pasa con `SystemCoreClock` y SysTick?** Es en la función `HAL_RCC_ClockConfig()` (al final de `SystemClock_Config`) donde la variable `SystemCoreClock` se actualiza a su valor final (probablemente 72 MHz en tu placa). Además, esta misma función **recalcula y reconfigura automáticamente el SysTick** para que siga contando exactamente 1 milisegundo, pero ahora a la nueva velocidad de 72 MHz.

```c
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
```

### El Conflicto de Tiempos: TIM4 vs SysTick

En un proyecto estándar sin RTOS, la librería HAL usa el **SysTick** para sus propios retardos (como `HAL_Delay()`). Pero cuando agregamos FreeRTOS, el kernel necesita tomar el control *exclusivo* del SysTick para gestionar el cambio de tareas (Time Slicing).

Si ambos intentan usar el SysTick para cosas distintas, el sistema se rompe. La solución es separar los trabajos.


```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
  /* USER CODE BEGIN Callback 0 */

  /* USER CODE END Callback 0 */
  if (htim->Instance == TIM4) {
    HAL_IncTick();
  }
  /* USER CODE BEGIN Callback 1 */
	if (htim->Instance == TIM2)
	{
		ulHighFrequencyTimerTicks++;
	}

  /* USER CODE END Callback 1 */
}
```


* **¿Cómo interactúa?** El STM32CubeIDE configuró el **Timer 4 (TIM4)** de hardware para que genere una interrupción cada 1 milisegundo. Cuando esto ocurre, el microcontrolador entra a esta función (el *Callback*) y llama a `HAL_IncTick()`.
* **¿Para qué interactúa?** La función `HAL_IncTick()` incrementa la variable global `uwTick` de la librería HAL. Esto significa que **TIM4 se convirtió en el nuevo "corazón" exclusivo de la librería HAL**. 
* **El resultado:** Al delegarle el trabajo de la HAL al TIM4, el **SysTick queda 100% liberado** para que FreeRTOS lo use para gobernar las tareas sin interferencias.

## ¿Qué pasaría con las funciones de la HAL (como `HAL_Delay`) si en el entorno de STM32CubeIDE nos olvidamos de hacer este cambio y le dejamos el SysTick tanto a FreeRTOS como a la HAL?

La función `HAL_Delay(1000)` internamente es un bucle while que se queda esperando a que esa variable global uwTick aumente en 1000 unidades.

Si dejamos el SysTick para ambos, cuando FreeRTOS arranca su planificador, "secuestra" la interrupción del SysTick para su propio uso exclusivo y deja de llamar a `HAL_IncTick()`. Como resultado, la variable `uwTick` se congela para siempre. Si en algún lugar de tu código (o adentro de alguna otra función de inicialización de la HAL) se llama a `HAL_Delay()`, el microcontrolador se va a quedar atrapado en ese while infinito esperando un tiempo que nunca va a pasar. ¡El sistema se cuelga por completo!

## ¿Qué ocurre dentro de `stm32f1xx_it.c`, mientras pasa esto?

Este archivo es el **hogar de las Rutinas de Servicio de Interrupción (ISR)**. Cuando ocurre un evento físico en el hardware del microcontrolador, el procesador frena lo que está haciendo (pausa las tareas de FreeRTOS) y "salta" a ejecutar una de las funciones que están acá adentro.

Anteriormente vimos que en `main.c` estaba la función que sumaba los ticks (`HAL_TIM_PeriodElapsedCallback`), pero faltaba saber cómo llegaba el procesador hasta ahí.

```c
void TIM4_IRQHandler(void)
{
  HAL_TIM_IRQHandler(&htim4);
}
```
Cada vez que pasan exactamente 1 milisegundo, el hardware del Timer 4 genera una interrupción y obliga al procesador a saltar a esta función `TIM4_IRQHandler()`. Esta función luego llama a `HAL_TIM_IRQHandler()`, la cual "limpia" la bandera de la interrupción y es la encargada de llamar a nuestra conocida `HAL_TIM_PeriodElapsedCallback()` en `main.c` para hacer que la variable de tiempo de la HAL avance.


A veces lo más importante de un código es lo que *no* está. Si alguna vez programaste un STM32 sin RTOS, habrás notado que en este archivo siempre hay una función llamada `SysTick_Handler(void)`. 

**¡Pero en este archivo no está!**
¿Por qué? Porque como charlamos antes, FreeRTOS necesita el SysTick de manera exclusiva. Por lo tanto, cuando STM32CubeIDE genera el código con FreeRTOS habilitado, borra el `SysTick_Handler` de este archivo y lo traslada a los archivos internos del sistema operativo (específicamente en el *port* de FreeRTOS o mapeado en `FreeRTOSConfig.h`). Esto confirma a la perfección que le entregamos el control total del SysTick al kernel de FreeRTOS.


Además del TIM4, este archivo está manejando dos interrupciones más:
* **`TIM2_IRQHandler`**: Responde al Timer 2. Si recordás el `main.c`, este timer se estaba usando para contar las estadísticas de tiempo de ejecución de las tareas (Run Time Stats).
* **`EXTI15_10_IRQHandler`**: Esta es la interrupción del botón físico de tu placa NUCLEO (conectado al pin `B1_Pin`). Cuando lo presiones, el hardware saltará acá.


## ¿Y que ocurre con `FreeRTOSConfig.h`?

El `FreeRTOSConfig.h` actúa como el **puente exacto entre el hardware de tu STM32 y el kernel de FreeRTOS**

```c
#ifndef FREERTOS_CONFIG_H
#define FREERTOS_CONFIG_H

#include <stdint.h>
extern uint32_t SystemCoreClock;

/* 1. Configuración principal del planificador (Scheduler) */
#define configUSE_PREEMPTION                     1
#define configCPU_CLOCK_HZ                       ( SystemCoreClock )
#define configTICK_RATE_HZ                       ((TickType_t)1000)
#define configMAX_PRIORITIES                     ( 7 )

/* 2. Configuración de Memoria */
#define configMINIMAL_STACK_SIZE                 ((uint16_t)128)
#define configTOTAL_HEAP_SIZE                    ((size_t)3072)

/* 3. Inclusión de funciones de la API (Ahorro de memoria Flash) */
#define INCLUDE_vTaskPrioritySet            1
#define INCLUDE_vTaskDelete                 1
#define INCLUDE_vTaskDelay                  1

/* 4. Manejo de Errores */
#define configASSERT( x ) if ((x) == 0) {taskDISABLE_INTERRUPTS(); for( ;; );}

/* 5. Mapeo de Interrupciones del hardware (Cortex-M) al RTOS */
#define vPortSVCHandler    SVC_Handler
#define xPortPendSVHandler PendSV_Handler
#define xPortSysTickHandler SysTick_Handler

#endif /* FREERTOS_CONFIG_H */
```

1. La conexión de la velocidad (`SystemCoreClock`)**
Como vimos antes, el `main.c` arranca a 8 MHz y luego acelera a 72 MHz, guardando ese valor en `SystemCoreClock`. ¿Pero cómo se entera FreeRTOS de esa velocidad para hacer sus cálculos de tiempo? A través de estas dos líneas en este archivo:
* `extern uint32_t SystemCoreClock;` (Importa la variable global desde el entorno de C).
* `#define configCPU_CLOCK_HZ ( SystemCoreClock )` (Le inyecta esa velocidad al "motor" matemático de FreeRTOS).

2. El ritmo del sistema (`configTICK_RATE_HZ`)**
Casi al principio vas a ver la línea `#define configTICK_RATE_HZ ((TickType_t)1000)`. Esto le dice explícitamente al kernel que el Tick del sistema debe ocurrir 1000 veces por segundo (cada 1 milisegundo). Con este dato y sabiendo la velocidad del CPU (el punto anterior), FreeRTOS sabe exactamente cómo configurar el hardware del SysTick.

3. El "secuestro" del SysTick: 
Casi al final del archivo vas a encontrar esta línea brillante:
`#define xPortSysTickHandler SysTick_Handler`. Esta macro en C es la que hace que, cuando el hardware dispara la interrupción del SysTick, la llamada se redirija **directamente hacia la función interna de FreeRTOS** (`xPortSysTickHandler`).Es la prueba en código de cómo FreeRTOS "secuestra" el SysTick para su uso exclusivo.

## ¿Y como se ve involucrado `freertos.c`?

Este archivo actúa como el "pegamento" final entre la configuración de tu proyecto y el núcleo interno de FreeRTOS.

Se involucra directamente en el comportamiento del programa justo antes de llegar al loop principal (`while(1)`) de esta manera:

### Análisis del archivo de integración: `freertos.c`

El archivo `freertos.c` autogenerado por STM32CubeIDE contiene la implementación de ciertas funciones (callbacks o "hooks") que el kernel de FreeRTOS requiere dependiendo de cómo hayamos configurado el archivo `FreeRTOSConfig.h`.

Al limpiar los comentarios del entorno, el código se reduce a lo siguiente:

```c
#include "FreeRTOS.h"
#include "task.h"
#include "main.h"

/* Prototipos de funciones */
void vApplicationGetIdleTaskMemory( StaticTask_t **ppxIdleTaskTCBBuffer, StackType_t **ppxIdleTaskStackBuffer, uint32_t *pulIdleTaskStackSize );
void configureTimerForRunTimeStats(void);
unsigned long getRunTimeCounterValue(void);

/* Funciones requeridas cuando configGENERATE_RUN_TIME_STATS = 1 */
__weak void configureTimerForRunTimeStats(void)
{
}

__weak unsigned long getRunTimeCounterValue(void)
{
  return 0;
}

/* Memoria requerida cuando configSUPPORT_STATIC_ALLOCATION = 1 */
static StaticTask_t xIdleTaskTCBBuffer;
static StackType_t xIdleStack[configMINIMAL_STACK_SIZE];

void vApplicationGetIdleTaskMemory( StaticTask_t **ppxIdleTaskTCBBuffer, StackType_t **ppxIdleTaskStackBuffer, uint32_t *pulIdleTaskStackSize )
{
  *ppxIdleTaskTCBBuffer = &xIdleTaskTCBBuffer;
  *ppxIdleTaskStackBuffer = &xIdleStack[0];
  *pulIdleTaskStackSize = configMINIMAL_STACK_SIZE;
}
```

1. Le da un hogar a la Tarea IDLE (`vApplicationGetIdleTaskMemory`):**
    Como charlamos al principio, cuando en el `main.c` se llama a la función para iniciar el planificador (`osKernelStart()`), lo primero que hace FreeRTOS antes de tomar el control del procesador es crear automáticamente la tarea especial llamada **IDLE**.
    Como en tu archivo `FreeRTOSConfig.h` está habilitada la opción de usar memoria "estática" (`configSUPPORT_STATIC_ALLOCATION 1`), el sistema operativo se niega a arrancar a menos que vos mismo le des un pedazo de memoria RAM dedicado para esa tarea. 
    Este archivo hace exactamente eso: declara las variables globales `xIdleTaskTCBBuffer` (para el bloque de control) y `xIdleStack` (para la pila de la tarea), y se las entrega a FreeRTOS a través de esa función. Sin este bloque de código, tu programa compilaría, pero el sistema operativo nunca arrancaría.

2. Prepara el terreno para las estadísticas de tiempo (Run Time Stats):**
    Fijate que al principio del archivo hay dos funciones llamadas `configureTimerForRunTimeStats` y `getRunTimeCounterValue` que tienen la palabra `__weak` adelante.
    FreeRTOS necesita una forma de medir cuánto tiempo de CPU consume cada tarea (estadísticas). Al ponerles `__weak`, STM32CubeIDE está diciendo: *"Voy a crear estas funciones vacías por defecto para que FreeRTOS no tire error de compilación"*. 
    Pero si hacés memoria, ¡nosotros vimos estas exactas mismas funciones, sin el `__weak`, escritas en tu `main.c`! En el `main.c` las configuramos para que usen el Temporizador 2 (TIM2). De esta forma, este archivo provee la estructura base, pero deja que tu código principal tome el control real del hardware.



# Resumen Parcial
Desde el `Reset_Handler`, pasamos por la configuración del reloj a 72 MHz en el `main.c`, luego redirigimos el reloj de la HAL al TIM4 en `stm32f1xx_it.c`, configuramos el SysTick para FreeRTOS en `FreeRTOSConfig.h`, y finalmente, este archivo `freertos.c` le entrega la memoria esencial para la tarea IDLE. Justo después de todo esto, el planificador arranca y el `while(1)` del `main.c` se vuelve inalcanzable.

# Desarrollo de la carpeta `app`

## ¿Como funciona `app.c`?

### Análisis de la inicialización de la aplicación (`app_init`)

Una vez que el hardware básico y el reloj del sistema han sido configurados en el `main()`, la ejecución pasa a la función `app_init()`. Al limpiar este archivo de comentarios, podemos observar claramente cómo se declaran las variables globales de monitoreo, los *Handles* de las tareas y, lo más importante, cómo se le indica al RTOS qué tareas debe administrar mediante la función `xTaskCreate`.

```c
#include "main.h"
#include "cmsis_os.h"
#include "logger.h"
#include "dwt.h"
#include "board.h"
#include "task_btn.h"
#include "task_led.h"

#define G_APP_TICK_CNT_INI              0ul
#define G_TASK_IDLE_CNT_INI             0ul
#define G_APP_STACK_OVERFLOW_CNT_INI    0ul

uint32_t g_app_tick_cnt;
uint32_t g_task_idle_cnt;
uint32_t g_app_stack_overflow_cnt;

TaskHandle_t h_task_btn;
TaskHandle_t h_task_led;

void app_init(void)
{
    g_app_tick_cnt = G_APP_TICK_CNT_INI;
    g_task_idle_cnt = G_TASK_IDLE_CNT_INI;
    g_app_stack_overflow_cnt = G_APP_STACK_OVERFLOW_CNT_INI;

    LOGGER_INFO(" ");
    LOGGER_INFO("%s is running - Tick [mS] = %3d", GET_NAME(app_init), (int)xTaskGetTickCount());
    LOGGER_INFO(" RTOS - Event-Triggered Systems (ETS)");
    LOGGER_INFO(" soe-tp0_03-application: Demo Code");

    BaseType_t ret;

    ret = xTaskCreate(task_btn,
                      "Task BTN",
                      (2 * configMINIMAL_STACK_SIZE),
                      NULL,
                      (tskIDLE_PRIORITY + 1ul),
                      &h_task_btn);

    configASSERT(pdPASS == ret);

    ret = xTaskCreate(task_led,
                      "Task LED",
```


Este archivo es fundamental porque funciona como el "director de orquesta" de tu aplicación. Si recordás el `main.c` que vimos antes, justo antes de encender el planificador de FreeRTOS, había una llamada a `app_init()`. Bueno, ¡ese código vive exactamente acá!


1. Inicialización y Diagnóstico:
Las primeras líneas dentro de `app_init()` inicializan contadores globales y utilizan una función `LOGGER_INFO` para imprimir mensajes por puerto serie. Esto es valiosísimo para depuración (debugging), ya que te permite ver en la consola en qué momento exacto de los *Ticks* del sistema arranca la aplicación (usando `xTaskGetTickCount()`).

2. El nacimiento de las Tareas (La parte central):
Aquí es donde aplicamos la teoría que charlamos al principio. El código utiliza la API nativa de FreeRTOS para dar vida a tus dos hilos de ejecución:

* **`task_btn` (La tarea del botón):**
    * Llama a `xTaskCreate` pasándole la función `task_btn` que contiene el código a ejecutar.
    * **Memoria (Stack):** Le asigna `2 * configMINIMAL_STACK_SIZE`. Si `configMINIMAL_STACK_SIZE` es 128 (como vimos en tu `FreeRTOSConfig.h`), le está asignando 256 palabras. Al ser un procesador de 32 bits (4 bytes por palabra), ¡le está reservando **1024 bytes** de memoria RAM solo para esta tarea!
    * **Parámetro:** Le pasa `NULL`. Por ahora la tarea no recibe información externa al iniciar.
    * **Prioridad:** Le asigna `tskIDLE_PRIORITY + 1ul`. Dado que la tarea IDLE siempre tiene prioridad 0, esta tarea **nace con prioridad 1**.
    * **Handle:** Guarda el control de la tarea en la variable `h_task_btn`.

* **`task_led` (La tarea del LED):**
    * Se crea de manera idéntica a la anterior. Lo más importante que tenés que notar acá es que **le asigna exactamente la misma prioridad (Prioridad 1)** que a la tarea del botón.

### Consecuencia en el Planificador (El Time Slicing)
Dado que ambas tareas tienen prioridad 1, ¿cómo elige FreeRTOS cuál ejecutar?. El sistema operativo aplicará **Round-Robin con Time Slicing**. Cada vez que ocurra la interrupción del SysTick (cada 1 milisegundo), FreeRTOS frenará a la tarea actual y le cederá el procesador a la otra, garantizando que ambas avancen compartiendo el tiempo equitativamente.

### Seguridad y Memoria
El código utiliza `configASSERT(pdPASS == ret)` justo después de crear cada tarea. Esta es una excelente práctica: si el microcontrolador se quedara sin memoria RAM para crear la tarea, `xTaskCreate` devolvería un error y el `configASSERT` "congelaría" el sistema ahí mismo para que te des cuenta del fallo, en lugar de arrancar un sistema inestable. Finalmente, `xPortGetFreeHeapSize()` calcula cuánta memoria RAM sobró después de crear todo.

## ¿Como funciona la tarea `task_btn_c`?

El archivo `task_btn.c` contiene la lógica para leer el estado del botón de usuario de la placa NUCLEO. Al limpiar el código de comentarios, podemos observar claramente su estructura, dividida en la declaración de la tarea de FreeRTOS y la máquina de estados que evalúa el botón.



```c
#include "main.h"
#include "cmsis_os.h"
#include "logger.h"
#include "dwt.h"
#include "board.h"
#include "app.h"
#include "task_btn_attribute.h"
#include "task_led_attribute.h"
#include "task_led_interface.h"

#define DEL_BTN_XX_MIN      0ul
#define DEL_BTN_XX_MED      25ul
#define DEL_BTN_XX_MAX      50ul

task_btn_dta_t task_btn_dta = {
        EV_BTN_XX_UP, ST_BTN_XX_UP, DEL_BTN_XX_MIN,
        B1_GPIO_Port, B1_Pin
};

void task_btn_statechart();

void task_btn(void *parameters)
{
    LOGGER_INFO(" ");
    LOGGER_INFO("%s is running - Tick [mS] = %3d", pcTaskGetName(NULL), (int)xTaskGetTickCount());

    for (;;)
    {
        task_btn_statechart();
    }
}

void task_btn_statechart(void)
{
    if (BTN_PRESSED == HAL_GPIO_ReadPin(task_btn_dta.gpio_port, task_btn_dta.pin))
    {
        task_btn_dta.event = EV_BTN_XX_DOWN;
    }
    else
    {
        task_btn_dta.event = EV_BTN_XX_UP;
    }

    switch (task_btn_dta.state)
    {
        case ST_BTN_XX_UP:
            if (EV_BTN_XX_DOWN == task_btn_dta.event)
            {
                task_btn_dta.tick = xTaskGetTickCount();
                task_btn_dta.state = ST_BTN_XX_FALLING;
            }
            break;

        case ST_BTN_XX_FALLING:
            if (DEL_BTN_XX_MAX <= (xTaskGetTickCount() - task_btn_dta.tick))
            {
                if (EV_BTN_XX_DOWN == task_btn_dta.event)
                {
                    LOGGER_INFO(" %s - BTN PRESSED", pcTaskGetName(NULL));
                    put_event_task_led(EV_LED_XX_BLINK);
                    task_btn_dta.state = ST_BTN_XX_DOWN;
                }
                else
                {
                    task_btn_dta.state = ST_BTN_XX_UP;
                }
            }
            break;

        case ST_BTN_XX_DOWN:
            if (EV_BTN_XX_UP == task_btn_dta.event)
            {
                task_btn_dta.tick = xTaskGetTickCount();
                task_btn_dta.state = ST_BTN_XX_RISING;
            }
            break;

        case ST_BTN_XX_RISING:
            if (DEL_BTN_XX_MAX <= (xTaskGetTickCount() - task_btn_dta.tick))
            {
                if (EV_BTN_XX_UP == task_btn_dta.event)
                {
                    LOGGER_INFO(" %s - BTN HOVER", pcTaskGetName(NULL));
                    put_event_task_led(EV_LED_XX_OFF);
                    task_btn_dta.state = ST_BTN_XX_UP;
                }
                else
                {
                    task_btn_dta.state = ST_BTN_XX_DOWN;
                }
            }
            break;

        default:
            task_btn_dta.tick  = xTaskGetTickCount();
            task_btn_dta.state = ST_BTN_XX_UP;
            task_btn_dta.event = EV_BTN_XX_UP;
            break;
    }
}
```


### 1. La Tarea Principal (`task_btn`)
Si mirás la función `task_btn(void *parameters)`, vas a ver el clásico bucle infinito `for (;;)` que debe tener toda tarea de FreeRTOS. Adentro de ese bucle, hace una sola cosa: llama a `task_btn_statechart()`.

**El detalle crítico (La Trampa):** ¿Notaste qué es lo que *falta* dentro de ese bucle infinito? **¡No hay ninguna función de retardo bloqueante!** No hay ningún `vTaskDelay()` ni `osDelay()`. 

Esto significa que esta tarea **nunca pasa al estado Blocked**. Se la pasa ejecutando el bucle a máxima velocidad de la CPU, leyendo el pin constantemente. A este tipo de tareas se las llama de "Procesamiento Continuo" o *Polling*. 
Dado que en `app.c` vimos que tiene Prioridad 1 (igual que el LED), FreeRTOS usa el **Time Slicing** (Round-Robin) para turnar el procesador entre el botón y el LED. Si esta tarea tuviera más prioridad que el LED, el LED jamás se encendería.

### 2. La Máquina de Estados (`task_btn_statechart`)
En lugar de frenar el procesador con un `HAL_Delay` para evitar los rebotes mecánicos del botón (lo cual rompería el sistema operativo), el código implementa una máquina de estados finitos (FSM) no bloqueante:

1.  **Lectura física:** Usa la HAL (`HAL_GPIO_ReadPin`) para ver si el botón está presionado o suelto.
2.  **Debouncing (Antirrebote) Inteligente:** Usa las variables `DEL_BTN_XX_MAX` (50 milisegundos) y `xTaskGetTickCount()`. 
    * Cuando detecta que el botón se presionó (pasa al estado `ST_BTN_XX_FALLING`), anota en qué Tick (milisegundo) ocurrió.
    * En las siguientes pasadas por el bucle, calcula la diferencia de tiempo: `xTaskGetTickCount() - task_btn_dta.tick`.
    * Solo si pasaron 50 milisegundos (50 ticks) y el botón *sigue* presionado, asume que es una pulsación real y no ruido eléctrico.

### Comunicación con la Tarea LED
Una vez que la máquina de estados confirma que el botón se presionó de forma válida (entra al `if` dentro de `ST_BTN_XX_FALLING`), hace dos cosas:
1.  Imprime por consola: `LOGGER_INFO(" %s - BTN PRESSED", pcTaskGetName(NULL));`
2.  **Llama a la interfaz:** Ejecuta `put_event_task_led(EV_LED_XX_BLINK);`. Esta es la función que "avisa" a la otra tarea (la del LED) que tiene que empezar a parpadear. Cuando se suelta el botón de forma válida, envía `EV_LED_XX_OFF`.

## ¿Como funciona `task_led.c` y `task_led_interface.c`?

### Análisis de la Tarea del LED (`task_led.c`) y Sincronización

El archivo `task_led.c` contiene la lógica encargada de controlar el LED de la placa de desarrollo. Al igual que la tarea del botón, está implementada mediante una **Máquina de Estados (Statechart)**. 

Al limpiar el código, podemos observar claramente su estructura:

```c
#include "main.h"
#include "cmsis_os.h"
#include "logger.h"
#include "dwt.h"
#include "board.h"
#include "app.h"
#include "task_led_attribute.h"

#define DEL_LED_XX_MIN      0ul
#define DEL_LED_XX_MED      250ul
#define DEL_LED_XX_MAX      500ul

task_led_dta_t task_led_dta = {
        false, EV_LED_XX_OFF, ST_LED_XX_OFF, DEL_LED_XX_MIN,
        LD2_GPIO_Port, LD2_Pin
};

void task_led_statechart();

void task_led(void *parameters)
{
    LOGGER_INFO(" ");
    LOGGER_INFO("%s is running - Tick [mS] = %3d", pcTaskGetName(NULL), (int)xTaskGetTickCount());

    HAL_GPIO_WritePin(task_led_dta.gpio_port, task_led_dta.pin, LED_OFF);

    for (;;)
    {
        task_led_statechart();
    }
}

void task_led_statechart(void)
{
    switch (task_led_dta.state)
    {
        case ST_LED_XX_OFF:
            if ((true == task_led_dta.flag) && (EV_LED_XX_BLINK == task_led_dta.event))
            {
                LOGGER_INFO(" %s - LED BLINK", pcTaskGetName(NULL));
                task_led_dta.flag = false;
                task_led_dta.tick = xTaskGetTickCount();
                task_led_dta.state = ST_LED_XX_BLINK;
                HAL_GPIO_WritePin(task_led_dta.gpio_port, task_led_dta.pin, LED_ON);
            }
            break;

        case ST_LED_XX_BLINK:
            if ((true == task_led_dta.flag) && (EV_LED_XX_OFF == task_led_dta.event))
            {
                LOGGER_INFO(" %s - LED OFF", pcTaskGetName(NULL));
                task_led_dta.flag = false;
                task_led_dta.state = ST_LED_XX_OFF;
                HAL_GPIO_WritePin(task_led_dta.gpio_port, task_led_dta.pin, LED_OFF);
            }
            else
            {
                if (DEL_LED_XX_MAX <= (xTaskGetTickCount() - task_led_dta.tick))
                {
                    task_led_dta.tick = xTaskGetTickCount();
                    HAL_GPIO_TogglePin(task_led_dta.gpio_port, task_led_dta.pin);
                }
            }
            break;

        default:
            task_led_dta.flag = false;
            task_led_dta.event = EV_LED_XX_OFF;
            task_led_dta.state = ST_LED_XX_OFF;
            task_led_dta.tick  = xTaskGetTickCount();
            HAL_GPIO_WritePin(task_led_dta.gpio_port, task_led_dta.pin, LED_OFF);
            break;
    }
}
```


Vamos a desglosar este código para entender cómo responde a las órdenes que le manda el botón.

### La Tarea Principal (`task_led`)
Al igual que vimos en la tarea del botón, `task_led` implementa su bucle infinito `for (;;)` donde únicamente llama a `task_led_statechart()`. 

**La lección clave que se repite:** Tampoco hay funciones bloqueantes (como `vTaskDelay`) acá. Esto confirma nuestra teoría de `app.c`: como ambas tareas tienen la misma prioridad (Prioridad 1) y ninguna se bloquea o suspende, **FreeRTOS está repartiendo el tiempo de la CPU constantemente (Time Slicing)**, dándole un milisegundo a `task_btn` para que lea el pin, y un milisegundo a `task_led` para que ejecute su máquina de estados.

### La Máquina de Estados (`task_led_statechart`)
Esta máquina de estados es la que le da vida al LED en base a los eventos que recibe:

* **Estado `ST_LED_XX_OFF` (Apagado):** La tarea evalúa continuamente si la variable `task_led_dta.flag` se puso en `true` y si el evento recibido es `EV_LED_XX_BLINK`. Si esto ocurre (lo cual es gatillado por la tarea del botón cuando lo presionás), enciende el LED, anota el *Tick* actual (la hora del sistema) y salta al estado de parpadeo.
* **Estado `ST_LED_XX_BLINK` (Parpadeo no bloqueante):**
    Acá hace dos cosas simultáneas:
    1.  Revisa constantemente si le llegó la orden de apagarse (`EV_LED_XX_OFF`). Si llega, apaga el LED y vuelve al estado OFF.
    2.  Si no llegó la orden de apagarse, hace una resta de tiempos: `xTaskGetTickCount() - task_led_dta.tick`. Si la diferencia es mayor o igual a `DEL_LED_XX_MAX` (que está definido en 500 Ticks/ms), cambia el estado del pin del LED (lo apaga si estaba prendido, o lo prende si estaba apagado) usando `HAL_GPIO_TogglePin`.



### El funcionamiento de la Interfaz `task_btn` -> `task_led`

Este archivo **`task_led_interface.c`** es pequeñito pero cumple una función arquitectónica muy importante en tu proyecto: **encapsulamiento**.


```c
#include "main.h"
#include "cmsis_os.h"
#include "logger.h"
#include "dwt.h"
#include "board.h"
#include "app.h"
#include "task_led_attribute.h"

void put_event_task_led(task_led_ev_t event)
{
    task_led_dta.event = event;
    task_led_dta.flag = true;
}
```

En lugar de que `task_btn` modifique directamente las variables de `task_led` (lo cual sería una mala práctica de programación en C, conocida como "código espagueti"), se utiliza esta función pública `put_event_task_led(task_led_ev_t event)`.

Cuando el botón detecta una pulsación, llama a esta función y le pasa el evento (por ejemplo, `EV_LED_XX_BLINK`). La función hace exactamente dos cosas muy simples pero vitales:
1.  Guarda el evento en la estructura compartida: `task_led_dta.event = event;`
2.  Levanta la bandera para avisar que hay un dato nuevo: `task_led_dta.flag = true;`

Es súper importante notar que esta comunicación se está haciendo a través de **memoria compartida** (variables globales). Aunque funciona perfectamente en este proyecto porque las tareas usan *Time Slicing* y no se interrumpen destructivamente, en un sistema RTOS más avanzado o con interrupciones, acceder a variables globales compartidas sin protección (como un Mutex) puede ser peligroso. En futuros trabajos prácticos, seguramente reemplazarán esta interfaz por una **Cola de Mensajes (Queue)** nativa de FreeRTOS, ¡que es la forma 100% segura de pasar eventos!


## ¿Que hace `freertos.c` en la carpeta app/scr?

Este archivo es el `freertos.c` que está ubicado dentro de la carpeta `app/src` y es distinto al archivo autogenerado de configuración que vimos antes. 

### Análisis de las Funciones "Hook" (Callbacks del Sistema)

Para finalizar el análisis del código fuente, revisaremos las implementaciones de los "Hooks" de FreeRTOS. Estas son funciones que el Kernel llama automáticamente bajo ciertas condiciones, permitiéndonos inyectar código de usuario en eventos clave del sistema operativo.

Al limpiar el código de los comentarios de licencia y encabezados, obtenemos:

```c
#include "main.h"
#include "cmsis_os.h"
#include "logger.h"
#include "dwt.h"
#include "board.h"
#include "app.h"

void vApplicationIdleHook(void)
{
    /* Incrementa el contador de ejecuciones de la tarea Idle */
    g_task_idle_cnt++;
}

void vApplicationTickHook(void)
{
    /* Incrementa el contador de Ticks de la aplicación */
    g_app_tick_cnt++;
}

void vApplicationStackOverflowHook(xTaskHandle xTask, signed char *pcTaskName)
{
    /* Si la pila de una tarea se desborda, el sistema se detiene aquí */
    taskENTER_CRITICAL();
    configASSERT( 0 );   
    taskEXIT_CRITICAL();

    g_app_stack_overflow_cnt++;
}
```

En este archivo se implementan lo que en FreeRTOS se conocen como **Hooks (Ganchos)** o funciones *Callback*. Son funciones especiales que nosotros escribimos, pero que el sistema operativo llama automáticamente cuando ocurren ciertos eventos importantes a nivel del kernel.

Este archivo se involucra brindando **estadísticas, telemetría y seguridad** al sistema. Vamos a analizar las tres funciones que contiene:

### 1. `vApplicationTickHook` (El latido del sistema)
Esta función se ejecuta de forma automática **cada vez que ocurre una interrupción del SysTick** (es decir, cada 1 milisegundo en tu configuración). 
* **Funcionamiento:** Al ejecutarse, incrementa la variable global `g_app_tick_cnt`.
* **Detalle clave:** Como dice el comentario en tu código, esta función se ejecuta dentro de una interrupción (ISR). Por lo tanto, debe ser extremadamente rápida y no debe usar funciones bloqueantes normales de FreeRTOS.

### 2. `vApplicationStackOverflowHook`
Cuando en `app.c` creamos `task_btn` y `task_led`, le asignamos a cada una un tamaño de Stack (pila de memoria). Si por algún error de programación una de esas tareas se excede y empieza a escribir fuera de su memoria asignada (Stack Overflow), FreeRTOS lo detecta y llama inmediatamente a esta función.
* **Funcionamiento:** Entra en una sección crítica (`taskENTER_CRITICAL()`) y ejecuta `configASSERT( 0 );`. Esto "cuelga" el microcontrolador a propósito en ese punto exacto.
* **¿Para qué sirve?** Es una herramienta invaluable de depuración. Es preferible que el sistema se detenga de forma segura y controlada para que el programador vea el error, en lugar de que siga corriendo corrompiendo la memoria y provocando comportamientos impredecibles.

### 3. `vApplicationIdleHook` 
Esta función es llamada repetidamente por la **Tarea IDLE** de FreeRTOS. Recordemos que la tarea IDLE es esa tarea de prioridad 0 que el sistema operativo ejecuta *solo* cuando todas las demás tareas están bloqueadas o suspendidas.
* **Funcionamiento:** Incrementa el contador `g_task_idle_cnt`.
* Si conectamos este archivo con lo que vimos en `task_btn.c` y `task_led.c`, recordá que **tus tareas no tienen retardos bloqueantes**, sino que están constantemente haciendo *polling* al máximo de velocidad usando *Time Slicing*. Como tus dos tareas siempre están en estado *Ready* o *Running*, y tienen Prioridad 1, **¡la Tarea IDLE (Prioridad 0) jamás tendrá la oportunidad de ejecutarse!**
* **Conclusión:** En esta versión específica la variable `g_task_idle_cnt` siempre valdrá cero, porque el procesador está trabajando al 100% de su capacidad sin descansar nunca.

---
