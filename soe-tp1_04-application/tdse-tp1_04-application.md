# TP1 - Actividad 04: Procesamiento Periódico y Tarea IDLE

## Respuestas a las preguntas del Paso 02

### 1. ¿Cómo implementar el procesamiento periódico mediante una Tarea?
[cite_start]El procesamiento periódico en FreeRTOS se implementa mediante el uso de funciones que bloquean la tarea por un tiempo determinado, siendo `vTaskDelayUntil()` la opción recomendada para mantener una frecuencia constante[cite: 30, 200]. Por otro lado `vTaskDelay()` es una forma de bloquear tareas sin mantener la frecuencia necesariamente constante. 

Los pasos fundamentales son:
* [cite_start]**Control del tiempo:** Se debe declarar una variable para almacenar el tiempo del último "despertar" (last wake time)[cite: 200].
* [cite_start]**Inicialización:** Antes del bucle `while(1)`, se inicializa dicha variable con el valor actual del contador de ticks del sistema mediante `xTaskGetTickCount()`[cite: 200].
* **Bloqueo absoluto:** Dentro del bucle, se invoca `vTaskDelayUntil()`. [cite_start]A diferencia de un delay relativo, esta función calcula el tiempo de desbloqueo basándose en el momento en que la tarea debió ejecutarse por última vez, compensando el tiempo de ejecución del código de la tarea y evitando el "drift" o deriva temporal[cite: 30, 200].

### 2. ¿Cuándo se ejecutará la Tarea IDLE y cómo se puede utilizar?
[cite_start]La tarea IDLE es una tarea de muy baja prioridad creada automáticamente por el kernel al iniciar el scheduler[cite: 30, 200].

* [cite_start]**Condición de ejecución:** La tarea IDLE se ejecuta únicamente cuando no existen otras tareas de la aplicación en estado "Ready" (listas) que tengan una prioridad superior a 0[cite: 30, 200]. [cite_start]Es decir, cuando todo el sistema está en espera o bloqueado[cite: 200].
* **Formas de utilización:**
    * [cite_start]**Liberación de recursos:** Su función principal es limpiar y liberar la memoria de tareas que han sido eliminadas por el sistema[cite: 30, 200].
    * [cite_start]**Bajo consumo:** Es el lugar ideal para colocar al microcontrolador en modos de ahorro de energía (Sleep modes) mientras no hay procesamiento pendiente[cite: 30, 200].
    * [cite_start]**Idle Hook:** Se puede definir una función de "gancho" (`vApplicationIdleHook`) para ejecutar código de fondo, como estadísticas de uso de CPU o parpadeo de un LED de estado que indique que el sistema está encendido pero ocioso[cite: 30, 200].

## Observaciones de la Actividad 04

A partir de las pruebas realizadas con el procesamiento periódico, se asienta lo observado:

### Gestión del Tiempo de CPU y Tarea IDLE
* **Desequilibrio de Delays:** Se observó que al utilizar un `vTaskDelay` (no determinístico) con un valor muy alto en una tarea mientras la otra no posee delay alguno (manteniendo prioridades iguales), el planificador otorga aproximadamente el 99% del tiempo de procesamiento a la tarea sin bloqueo, relegando a la otra a un uso marginal del 1%.
* **Predominancia de la Tarea IDLE:** En escenarios donde ambas tareas poseen delays significativamente grandes, el sistema entra mayoritariamente en estado de espera, permitiendo que la tarea `IDLE` ocupe cerca del 99% del tiempo. Esto confirma que el tiempo ocioso es fundamental para la estabilidad y el "mantenimiento" del sistema operativo.
* **Optimización:** La búsqueda de un equilibrio en los tiempos de bloqueo permite alcanzar un uso razonable y eficiente de la CPU para cada tarea según sus necesidades.

### Impacto en la Lógica de Control (Statechart)
* **Detección del Botón:** Se determinó que si el periodo de la tarea del botón es excesivamente alto, la respuesta del sistema se degrada. Dado que la lógica del botón se basa en un *statechart* de 4 estados, un delay prolongado obliga a mantener el botón presionado durante mucho tiempo para que la transición sea detectada.
* **Pérdida de Eventos:** Si el delay supera el tiempo de presión física, el sistema puede pasar de un estado `falling` directamente a `up` nuevamente. Al "despertar" la tarea después de un delay largo, el tick de lectura detecta que el botón ya fue liberado, perdiendo la fase de confirmación de presionado.
* **Dependencia del LED:** Debido a que el comportamiento del LED es una respuesta directa a lo detectado por la tarea del botón, cualquier deficiencia en el muestreo del periférico de entrada se traduce automáticamente en una falla de respuesta en la señal de salida (LED).