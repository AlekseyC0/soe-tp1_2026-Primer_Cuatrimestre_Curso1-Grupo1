


### Paso 02: Conceptos Teóricos de FreeRTOS

En base a la experiencia depurando el proyecto, se observan los siguientes comportamientos fundamentales del RTOS:

* **¿Cómo FreeRTOS asigna tiempo de procesamiento a cada Tarea en una aplicación?** 
    FreeRTOS utiliza el temporizador del sistema (SysTick) para generar interrupciones periódicas (Ticks, configurado a 1 ms). En cada Tick, el sistema evalúa si debe realizar un cambio de contexto (Time Slicing), asignando "rebanadas de tiempo" del procesador a las tareas que lo requieran.
* **¿Cómo FreeRTOS elige qué Tarea debe ejecutarse en un momento dado?**
    El planificador (Scheduler) siempre selecciona la tarea que se encuentre en estado *Ready* (Lista) y posea la **mayor prioridad**. Si existen múltiples tareas en estado *Ready* con la misma prioridad, FreeRTOS alterna entre ellas de forma equitativa utilizando un algoritmo *Round-Robin* en cada Tick (y asignandole un procentaje del CPU a cada uno). Esto es visualmente comprobable con FreeRTOS Task viewer.
* **¿Cómo la prioridad relativa de cada Tarea afecta el comportamiento del sistema?** 
    Al ser un sistema operativo apropiativo (*preemptive*), si una tarea de alta prioridad pasa al estado *Ready*, desaloja inmediatamente a cualquier tarea de menor prioridad que esté ejecutándose (Running]. Si la tarea de alta prioridad no se bloquea explícitamente, monopolizará el CPU provocando la inanición (*starvation*) de las tareas de menor prioridad.
* **¿Cuáles son los estados en los que puede encontrarse una Tarea?** 
    * **Running (Ejecutándose):** Tiene el control actual del CPU.
    * **Ready (Lista):** En condiciones de ejecutarse, esperando que el CPU se libere.
    * **Blocked (Bloqueada):** Esperando un evento temporal (`vTaskDelay`) o de sincronización. No consume CPU.
    * **Suspended (Suspendida):** Pausada explícitamente; el planificador la ignora por completo.
* **¿Cómo implementar Tareas?**
    Se implementan como funciones en C que retornan `void` y reciben un puntero genérico `void *parameters`. Deben estructurarse dentro de un bucle infinito y nunca deben llegar a su fin mediante un `return`.
* **¿Cómo crear una o más instancias de una Tarea?** 
    Se utiliza la función de la API `xTaskCreate()`. Para crear múltiples instancias, se llama repetidas veces a esta función apuntando a la misma rutina de C, pero asignándoles nombres distintos, diferentes parámetros y manejadores (Handles) únicos para que posean su propia pila de memoria (Stack) independiente.
* **¿Cómo eliminar una Tarea?**
Se invoca la función `vTaskDelete()`, pasándole como argumento el "Handle" de la tarea que se desea destruir. Si una tarea desea eliminarse a sí misma, puede llamar a `vTaskDelete(NULL)`.

---

### Paso 03: Modificación de Prioridades Relativas

* **Acción realizada:** Se modificó en `app.c` la prioridad de `task_btn` a `(tskIDLE_PRIORITY + 2ul)`, manteniendo `task_led` en `(tskIDLE_PRIORITY + 1ul)`.
* **Comportamiento observado:** Al iniciar la depuración y observar la vista *FreeRTOS Task List*, se comprobó que `task_btn` acaparó el 100% del Run Time del procesador. Debido a que `task_btn` se ejecuta mediante un *polling* continuo (sin funciones bloqueantes), el planificador nunca le retiró el control. Como consecuencia directa, `task_led` sufrió de inanición total (*starvation*), quedando con 0% de uso de CPU y provocando que el hardware físico (LED) dejara de responder.
* **Estado final:** Se reestablecieron las prioridades relativas originales `(tskIDLE_PRIORITY + 1ul)` para ambas tareas, recuperando el reparto de CPU al 50% cada una.

---

### Paso 04: Creación y Eliminación de Instancias

* **Acción realizada:** En `app.c` se invocó tres veces consecutivas a `xTaskCreate` apuntando a `task_btn` para generar tres instancias: "Task BTN 1", "Task BTN 2" y "Task BTN 3", asignándoles Handles independientes. Adicionalmente, se importó el Handle de la tercera instancia dentro de `task_led.c` utilizando `extern`, y se incluyó la instrucción `vTaskDelete(h_task_btn_3)` en la inicialización de la tarea del LED.
* **Comportamiento observado:** Inicialmente, el sistema sufrió un desbordamiento de memoria (falta de Heap) al intentar crear las 4 tareas, lo cual se solucionó reduciendo el Stack asignado a `configMINIMAL_STACK_SIZE` para cada una. Tras esta corrección y al ejecutar el código, se visualizó en el depurador cómo "Task BTN 3" era destruida dinámicamente de la lista de tareas en tiempo real. La memoria de su Stack fue liberada de forma exitosa, demostrando la capacidad de FreeRTOS para gestionar el ciclo de vida de las tareas en tiempo de ejecución.
