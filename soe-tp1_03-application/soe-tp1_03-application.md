# SOE TP1_03 Application

## Paso 02

### ¿Cómo usar el parámetro de Tarea?

En FreeRTOS, cada tarea se define con la firma:

void task(void *pvParameters);

El parámetro `pvParameters` es un puntero genérico (`void *`) que permite pasar información a la tarea al momento de su creación mediante `xTaskCreate`.

Uso:

xTaskCreate(task_function,
            "Task",
            stack_size,
            pvParameters,
            priority,
            &handle);

Dentro de la tarea, el parámetro se recupera mediante casting:

tipo *param = (tipo *) pvParameters;

Esto permite:

• Diferenciar múltiples instancias de una misma tarea  
• Evitar variables globales  
• Configurar comportamiento en tiempo de ejecución  


### ¿Cómo cambiar la prioridad de una Tarea ya creada?

Se utiliza la función:

vTaskPrioritySet(TaskHandle_t xTask, UBaseType_t uxNewPriority);

donde:

• `xTask`: handle de la tarea  
• `uxNewPriority`: nueva prioridad  

Ejemplo:

vTaskPrioritySet(task_handle, tskIDLE_PRIORITY + 2);

Efecto:

• El scheduler selecciona la tarea de mayor prioridad en estado Ready  
• Si la nueva prioridad es mayor, la tarea puede ejecutarse inmediatamente  
• Si es menor, puede ser desplazada por otras tareas  


## Paso 03 – Gestión de dos botones diferentes

Se modificó la tarea `task_btn` para que reciba un parámetro de tipo `task_btn_dta_t *`, permitiendo que cada instancia gestione un botón diferente.  
Se crearon dos instancias de la tarea `task_btn`:

- **Task BTN 1:**  
  - Datos asociados: `btn1`  
  - Handle: `h_task_btn_1`  

- **Task BTN 2:**  
  - Datos asociados: `btn2`  
  - Handle: `h_task_btn_2`  

Cada instancia utiliza el mismo código de la máquina de estados `task_btn_statechart`, diferenciando los botones a través del parámetro recibido.

- Ambos botones son reconocidos independientemente.  
- Cada pulsación genera los logs correspondientes (`BTN PRESSED` y `BTN HOVER`) en la consola según el estado del botón.  
- El comportamiento de rebote se gestiona correctamente gracias a los estados `FALLING` y `RISING` con la temporización `DEL_BTN_XX_MAX`.

## Paso 04 – Prioridad de `task_led`

Se modificó la prioridad de la tarea `task_led` para que tenga la máxima prioridad relativa al resto de las tareas, asegurando que se ejecute primero al inicio del scheduler:

```c
ret = xTaskCreate(task_led,
                  "Task LED",
                  (2 * configMINIMAL_STACK_SIZE),
                  NULL,
                  (tskIDLE_PRIORITY + 1ul + 1),  // Prioridad incrementada
                  &h_task_led);