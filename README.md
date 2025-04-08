# Blink LED con interrupción de botón e interrupción de temporizador

Este proyecto hace parpadear el LED LD2 de la Nucleo-STM32F446RE a una frecuencia de `c` Hz. La frecuencia es controlada por el pulsado del botón de usuario B1. Cada vez que se pulsa el botón, la frecuencia (`c`) aumenta en 1. El LED está apagado cuando `c` es igual a 0. El botón es gestionado por una rutina de servicio de interrupción (ISR). El tiempo de parpadeo es gestionado por interrupción de temporizador.

El sistema usa la interrupción del temporizador 2 (TIM2) para detectar el final del tiempo de encendido o apagado del LED. La interrupción se configura con los siguientes ajustes:

| Parámetro   | Valor           |
| ----------- | --------------- |
| Interrupción| TIM2_IRQHandler |
| Prioridad   | 2               |
| Subprioridad| 0               |

## Ejercicio: parpadeo con interrupción del TIM2

**Cree el proyecto** `blink_led_interr_button_interr`. **Se trata de una modificación del programa** `blink_led_button_interr` **donde la acción de esperar no se toma al leer mediante *polling* el valor del** `SysTick` **para medir el tiempo pasado, sino mediante una interrupción del *timer* 2** (`TIM2`).

**En todo momento usaremos la HAL de ST.**

1 . En el fichero `stm32f4_led.h` completar los `#define` que faltan:

```c
#define STM32F4_LED_TIMER TIM2
#define STM32F4_LED_TIMER_IRQn TIM2_IRQHandler
#define STM32F4_LED_TIMER_IRQ_PRIO
#define STM32F4_LED_TIMER_IRQ_SUBPRIO
```

2 . En el fichero `stm32f4_led.c` hay que crear la variable global que define la estructura de tipo `TIM_HandleTypeDef` del *timer*. Use un nombre representativo, por ejemplo `handler_timer_led`:

  ```c
  TIM_HandleTypeDef handler_timer_led;
  ```

3 . **En el mismo fichero, crear la función** `port_led_timer_setup()` **que inicializa la estructura** *timer* **y hacerla pública en el `.h`.**

   - Lo primero siempre es habilitar la **fuente de reloj** del *timer* 2:

```c
__HAL_RCC_TIM2_CLK_ENABLE();
```

  de otro modo no podrán tocar los registros del *timer*.
   - En la estructura `handler_timer_led` **inicie los campos**:
     - `handler_timer_led.Instance` con el *timer* del LED que se ha definido en `STM32F4_LED_TIMER`. 
     - `handler_timer_led.Init.Prescaler` con el valor del *prescaler* a 0 (no se sabe todavía).
     - `handler_timer_led.Init.Period` con el valor del periodo a 0 (no se sabe todavía).
     - `handler_timer_led.Init.AutoReloadPreload` con el valor a `TIM_AUTORELOAD_PRELOAD_ENABLE` (para que el contador se recargue automáticamente; si no se hace, solo se contará una vez).
   - **Inicializar el** *timer* con la función `HAL_TIM_Base_Init()` pasando la dirección de la estructura `handler_timer_led`.
   - **Activa el temporizador en modo interrupción** del *timer*: `HAL_TIM_Base_Start_IT();` pasando la dirección de la estructura `handler_timer_led`.
   - **Configurar la prioridad** de la interrupción  con la función `HAL_NVIC_SetPriority()` pasando el nombre de la IRQ del *timer*, la prioridad y la subprioridad definidas en el `.h`.
   - **Habilitar la interrupción** del *timer* en el *NVIC*: `HAL_NVIC_EnableIRQ()` pasando el nombre de la IRQ del *timer* definida en `STM32F4_LED_TIMER_IRQn`.

4 . **Crear la función** `port_led_timer_delay_ms(uint32_t delay_ms)` **que configura el tiempo de interrupción periódica del *timer* al valor** `delay_ms`, **y hacerla pública en el `.h`.**

  - **Deshabilitar** la cuenta del *timer* para que no interrumpa, para ello llamar a la función `HAL_TIM_Base_Stop_IT()` pasando la dirección de la estructura `handler_timer_led`.
  - Poner el contador `CNT` a 0 con la función `__HAL_TIM_SET_COUNTER()` pasando la dirección de la estructura `handler_timer_led` y el valor 0.
  - **Calcular el valor del periodo** (`ARR`) **en función del** *prescaler* (`PSC`).
     
    - La ecuación del periodo de interrupción del *timer* es `T_interr = f_clk / ((PSC + 1) * (ARR + 1))`.
    - La frecuencia del reloj del sistema es 16 MHz y está definida en la variable `SystemCoreClock`.
    - **Comprobar** que el valor del periodo `ARR` ni el *prescaler* `PSC` superan el valor máximo de 16 bits (65535). Puede hacerlo en un bucle `while()`.
    - Tener en cuenta que el valor de interrupción recibido es en milisegundos y hay que **convertirlo a segundos**.
    - **Trabajar con precisión `double` en todas las variables y constantes.** *castear* los valores de la frecuencia del reloj del sistema y del periodo de interrupción a `double`. *e.g.*:
      - `double f_clk = (double)SystemCoreClock`
      - `double T_interr = (double)delay_ms / 1000.0`
      - `1.0` en lugar de `1` para que el resultado sea `double`.
      - Cuando tenga los valores de `PSC` y `ARR`, los cargamos en los campos correspondientes de la estructura `handler_timer_led.Init`:
        - `handler_timer_led.Init.Prescaler` con el valor de `PSC`.
        - `handler_timer_led.Init.Period` con el valor de `ARR`.
    - **Actualizar la configuración del** *timer* con la función `HAL_TIM_Base_Init()` pasando la dirección de la estructura `handler_timer_led`.
    - **Forzar la actualización de registros** activando el bit `UG`, para ello llamar a la función `HAL_TIM_GenerateEvent()` pasando la dirección de la estructura `handler_timer_led` y el tipo de evento `TIM_EVENTSOURCE_UPDATE` (actualización). Esto hará que el *timer* recargue el valor del contador `CNT` a 0 y el valor del periodo `ARR` al nuevo valor.
    - Por último, **habilitar de nuevo la cuenta del *timer*** activando el bit `CEN` mediante la función `HAL_TIM_Base_Start_IT()` pasando la dirección de la estructura `handler_timer_led`.
  
5 . **En el fichero** `interr.c`, donde están las variables globales, **declarar la variable** `handler_timer_led` **como** `extern`. Esto le dice al compilador que la variable está definida en otro fichero y que no la defina de nuevo. La declaración debe ser:

```c
extern TIM_HandleTypeDef handler_timer_led;
```

6 . **Implementar la función** `TIM2_IRQHandler()` **que se ejecuta cuando se produce la interrupción del *timer* 2.**

  - Llamar a la función `HAL_TIM_IRQHandler()` pasando la dirección de la estructura `handler_timer_led`. Esto limpiará el bit `UIF` del registro `SR` y llamará automáticamente a la función de *Callback* `HAL_TIM_PeriodElapsedCallback()` que es la que está relacionada con la interrupción que haya sucedido; en nuestro caso, llamará a la de fin de periodo de interrupción.

7 . **Implementar la función** `HAL_TIM_PeriodElapsedCallback()` **en el mismo fichero.**
  - Lo primero es **comprobar** que la interrupción que ha sucedido es la de fin de periodo del *timer* del LED, *i.e.*:

```c
    if (handler_tim->Instance == STM32F4_LED_TIMER)
    {
      // Código de la interrupción del timer del LED
    }
```

  - Si es nuestro *timer* el que ha interrumpido, llamar a la función `port_led_toggle()` para cambiar el estado del LED **solo si el contador de pulsaciones `c` es mayor que 0.**
  - Gestionar el indicador de botón pulsado `button_pressed` adecuadamente.
  
8 . **En la `HAL_GPIO_EXTI_Callback` del botón, llamar a la función** `port_led_timer_delay_ms()` con el valor adecuado para el parpadeo del LED. 

  Como puede ver, como ya no hay que hacer espera activa, ya no hay que llamar a `port_system_delay_ms()` en la función `main()`. En su lugar, se llama a `port_led_timer_delay_ms()`, que es la que configura el *timer* para que interrumpa cada `delay_ms` milisegundos.

9 . **Por último, en** `main.c`:

  - **Quitar** del `main()` la gestión del parpadeo del LED.
  - Como las variables `c` y `button_pressed` se gestionan en las ISRs, **quitamos su declaración y en** `interr.c` **ya no hace falta declararlas como** `extern`; **quítelo**.
  - **Llamar a la función** `port_led_timer_setup()` en el `main()` para inicializar el *timer* que controla el parpadeo del LED. El bucle `while(1)` **ya no tiene que hacer nada**. El parpadeo del LED se hace en la ISR del *timer*.

## References

- **[1]**: [Fundamentos teóricos de sistemas basados en microcontrolador STM32. Sistemas Digitales II, Sistemas Electrónicos](https://oa.upm.es/88460)
- **[2]**: [Embedded Systems with ARM Cortex-M Microcontrollers in Assembly Language and C (Fourth Edition)](https://web.eece.maine.edu/~zhu/book/index.php) for explanations and examples of use of the ARM Cortex-M microcontrollers in C with CMSIS.
- **[3]**: [Programming with STM32: Getting Started with the Nucleo Board and C/C++](https://ingenio.upm.es/primo-explore/fulldisplay?docid=34UPM_ALMA51126621660004212&context=L&vid=34UPM_VU1&lang=es_ES&search_scope=TAB1_SCOPE1&adaptor=Local%20Search%20Engine&tab=tab1&query=any,contains,Programming%20with%20STM32:%20Getting%20Started%20with%20the%20Nucleo%20Board%20and%20C%2FC%2B%2B&offset=0) for examples of use of the STM32 microcontrollers with the HAL of ST.
- **[4]**: [The C Programming Language](https://ingenio.upm.es/primo-explore/fulldisplay?docid=34UPM_ALMA2151866130004212&context=L&vid=34UPM_VU1&lang=es_ES&search_scope=TAB1_SCOPE1&adaptor=Local%20Search%20Engine&isFrbr=true&tab=tab1&query=any,contains,C%20Programming%20Language)
- **[5]**: [Nucleo Boards Programming with th STM32CubeIDE](https://www.elektor.com/products/nucleo-boards-programming-with-the-stm32cubeide) for examples of use of the STM32 microcontrollers with the STM32CubeIDE.
