# Pintos: Scheduler de Prioridades Multi Level FeedBack Queue

## Notas importantes
Revisar a detalle la descripción de la prácticas pues allí vienen varias consideraciones importantes de cómo implementar la solución.

## Nice
* Se debe de agregar al PCB, i.e `struct thread`.
* Su valor debe de estar siempre entre `-20 <= nice <= 20`.
* El thread `initial_thread` debe de tener un valor inicial de `nice` de `0` (recomendamos asignar este valor al final de `thread_init`), el resto de los hilos lo heredan del hilo padre (recomendamos asignar este valor en `thread_create`, en este caso el thread padre es el `thread_current ()`). 

### Priority
* La prioridad de todos los hilos se actualiza cada 4 ticks.
* Si el valor calculado se sale del rango `[PRI_MIN, PRI_MAX]` hay que acotarlo para que quede en rango, esto es particularmente importante cuando ya implementamos la `ready_list` como un multi level queue.

### Recent CPU
* Se debe de agregar al PCB, i.e `struct thread`.
* Para el thread_current se actualiza un vez cada tick (+1), excepto si es `idle_thread`.
* Se actualiza para todos los hilos una vez cada segundo.
* El thread `initial_thread` debe de tener un valor inicial de `recent_cpu` de `0` (recomendamos asignar este valor al final de `thread_init`), el resto de los hilos lo heredan del hilo padre (recomendamos asignar este valor en `thread_create`, en este caso el thread padre es el `thread_current ()`). 
  
### Load Average
* Es una valor global del módulo de threads (`threads.c`)
* Su valor iniciar debe de ser 0 en `thread_init ()`.
* Lo tenemos que actualizar una vez cada segundo (`timer_ticks () % TIMER_FREQ == 0.`).

### Recálculos
* Un segundo transcurre cada vez que `timer_ticks () % TIMER_FREQ == 0.`
  - Nota: `TIMER_FREQ` está definido en `timer.h`
* Los recálculos de todos los valores los podemos realizar en `thread_tick` en `thread.c`, pues esta es una
subrutina del interrupt handler del timer (`timer_interrupt ()` en `timer.c`).
* Estos recálculos se tienen que llevar acabo cuando está prendida la bandera `thread_mlfqs`.
* La lista `all_list` de `thread.c` contiene a todos los hilos independientemente de su estado (RUNNING, BLOCKED, etc..). La mejor manera de iterarlar es mediante la función `thread_foreach (func, aux)`, que ejecuta la función `func(thread, aux)` para cada `thread` dentro de `all_list`.

### Números de Punto Fijo (fixpoint_t)
* El procesador no tiene habilitado el procesamiento números flotantes en modo kernel por temas de performance, por esa razón debemos de implementar un formato de números racionales simple pero efectivo, aquí es donde metemos los números de punto fijo (`fixpoint`).
* Revisar la descripción de los números de punto fijo primero en el PDF y luego en el archivo `fixpoint.(c|h)`
* Introducimos el tipo `fixpoint_t` como un alias de `int32_t` para hacer el código más legible.
* A continuación algunos ejemplos de como operar números de punto fijo con números entero o con otros números de punto fijo:
```c
fixpoint_t x;
int32_t a;

fixpoint_t a_times_x = x * a;
fixpoint_t x_div_a = x / a;
fixpoint_t x_plus_a = x + int_to_fixpoint(a);

fixpoint_t y;

fixpoint_t y_times_x = fixpoint_mult (x, y);
fixpoint_t x_div_y = fixpoint_div (x, y);
fixpoint_t x_plus_y = x + y;

// 59/60 to fix point
fixpoint_t rational_num = rational_to_fixpoint (59, 60);
```

### Cambios en mlfqs-calcultions.c
Se sugiera implementar en este archivo las fórmulas para calcular `load_avg`, `recent_cpu` y `priority`.

### Cambios en thread.h
```c
// thread.h
#define NICE_MAX 20
#define NICE_DEFAULT 0
#define NICE_MIN -20

#define RECENT_CPU_DEFAULT 0

struct thread {
  ..
  int nice;
  fixpoint_t recent_cpu;
  ..
}
```

### Cambios en thread.c

```c
// thread.c
// global en este módulo
// inicializar en 0 en thread_init ()
fixpoint_t load_avg;

...


// Nota: corre con las interrupciones apagadas pues es una subrutina de un interrupt handler
void thread_tick (void) {
  // ... actualiza ticks

  if (thread_mlfqs) {
    // calcular el número de hilos activos (i.e. en estado running o ready)
    // aquí recalculamos el load_avg cada segundo
    // aquí incrementamos en 1 en valor de recent_cpu para el thread_current si este es distinto de idle_thread
    // aquí recalculamos el recent_cpu para todos los hilos cada segundo
    // aquí recalculamos la prioridad para todos los hilos cada 4 ticks
    if (han_pasado_4_tick) {
      // actualizamos la prioridad de todos los hilos con la función `thread_foreach`
      thread_foreach (update_thread_priority, NULL);
      // NOTA: hay que reordenar otra vez la ready_list pues las prioridades de los hilos en estado ready cambiaron.
    }
  }

  // ... thread preemption
}

static void update_thread_priority(struct thread* thread, void* aux UNSED) {
  // aquí realizar el cálculo
  // nota: mlfqs_calculate_priority está definida en `mlfqs-calculations.(h|c)`
  thread->priority = mlfqs_calculate_priority(thread->recent_cpu, thread->nice);
}
```

## Multi Level Feedback Queue
Sugerimos realizar este cambio solamente hasta que ya se terminó de resolver el 
problema de la hambruma (__starvation__) con todo lo anterior.
```c
// original
struct list ready_list;

// nueva - una lista RR por cada prioridad pues son acotada [0, 63]
struct list ready_list[PRI_MAX+1];  

// Agregar un hilo a la ready list
struct thread* t = thread_current ();
list_push_front(&ready_list[t->priority], &t->elem);

// para encontrar el hilo de más alta prioridad de la ready_list
// lo que hacemos es iterar el arreglo de la entrada 63 a la 0 y hacemos 
// list_pop_front de la primera lista no vacía que nos encontremos
```
