# Threads vs Coroutines

[English version](11-threads-vs-coroutines.en.md)

---

## 1. El modelo mental: quien maneja a quien

Un **thread** es una unidad de ejecucion que maneja el sistema operativo. Es relativamente pesado: cada uno reserva memoria para su stack (tipicamente ~1-2 MB en la JVM) y cambiar entre threads implica un context switch del SO, que cuesta.

Una **corrutina** es una unidad de trabajo que corre *sobre* threads, pero la maneja el runtime de Kotlin, no el SO. Es liviana: se pueden tener cientos de miles sin problema.

**Las corrutinas no reemplazan a los threads, corren sobre ellos.** Un dispatcher es lo que decide en que thread o pool corre cada corrutina (ver [07 - Context y Dispatchers](07-concurrencia-context.es.md)). No es "threads vs corrutinas" como enemigos; las corrutinas son una capa de abstraccion encima que permite exprimir mejor un numero pequeno de threads.

---

## 2. La diferencia clave: bloquear vs suspender

Cuando un thread espera algo (una llamada de red, un `sleep`), se **bloquea**: queda ocupado sin hacer nada, pero sigue reservando su memoria y su lugar.

Una corrutina, en cambio, **suspende**: cuando llega a un punto de espera, libera el thread para que otra corrutina lo use, y vuelve a retomar cuando su resultado esta listo. Suspender no bloquea el thread subyacente.

> Bloquear ocupa el thread sin trabajar; suspender lo libera para que trabaje en otra cosa.

---

## 3. El impacto practico (el ejemplo numerico)

Si se quieren 100.000 operaciones concurrentes con threads, se necesitarian 100.000 threads - imposible, se agota la memoria mucho antes. Con corrutinas, esas 100.000 pueden correr sobre un pool de pocos threads, porque la mayoria estan suspendidas esperando I/O y liberando su thread mientras tanto.

```kotlin
// Con threads: esto revienta la memoria (~100 GB de stacks)
repeat(100_000) { thread { Thread.sleep(1000) } }

// Con corrutinas: corre sin problema sobre pocos threads
repeat(100_000) { launch { delay(1000) } }
```

La diferencia es que `Thread.sleep` bloquea un thread real, mientras que `delay` suspende la corrutina y libera el thread.

---

## 4. Tabla comparativa

| | Thread | Corrutina |
|---|---|---|
| Lo maneja | El sistema operativo | El runtime de Kotlin |
| Costo de memoria | ~1-2 MB (stack) | Bytes (un objeto) |
| Cuantas caben | Miles | Cientos de miles |
| Al esperar | Se bloquea (ocupa el thread) | Suspende (libera el thread) |
| Context switch | Del SO, caro | Del runtime, barato |
| Cancelacion | Manual, complicada | Cooperativa, via scope |
| Relacion | Corre sobre el SO | Corre sobre threads |

---

## 5. Lo que demuestra experiencia real

**Structured concurrency**: las corrutinas viven en un scope y respetan una jerarquia (ver [06 - Scopes](06-concurrencia-scopes.es.md)). Si se cancela el scope, se cancelan todas las hijas. Con threads eso hay que manejarlo a mano y es propenso a fugas.

**La cancelacion es cooperativa**: una corrutina se cancela en los puntos de suspension, no la mata el SO de golpe (ver [08 - Cancelacion](08-concurrencia-cancelacion.es.md)). Por eso el codigo tiene que escribirse respetando la cancelacion.

**El peligro de mezclarlos**: si dentro de una corrutina se llama codigo bloqueante (una libreria vieja, JDBC sincrono), se bloquea el thread real y se pierde la ventaja. Por eso ese trabajo va en `Dispatchers.IO`, que tiene margen (hasta 64 hilos) para absorber bloqueo sin tocar los hilos que `Default` necesita para CPU.

---

## 6. La trampa del "mas rapido"

Una corrutina **no** ejecuta codigo mas rapido que un thread; el trabajo de CPU tarda lo mismo. Lo que se gana es **concurrencia y uso eficiente de recursos**: se maneja muchisima mas espera concurrente con menos memoria y menos context switches. Es "mas eficiente en concurrencia", no "mas rapido en computo" - distinguir esto es justo lo que separa una respuesta memorizada de una con criterio real.

---

## Resumen - tabla de decision rapida

| Necesito... | Opcion |
|---|---|
| Miles de tareas esperando I/O concurrentemente | Corrutinas (concurrencia barata) |
| Trabajo de CPU repartido en cores | Corrutinas con `Dispatchers.Default` (que a su vez usan threads) |
| Interactuar con una API legacy basada en threads | Threads directos, o envolver con `withContext(Dispatchers.IO)` |
| Cancelacion estructurada y jerarquica | Corrutinas (structured concurrency) |

---

## Frases para entrevista

- "Un thread lo maneja el SO; una corrutina la maneja el runtime de Kotlin y corre sobre threads."
- "Bloquear ocupa el thread sin trabajar; suspender lo libera."
- "No es threads vs corrutinas: las corrutinas son una capa encima que exprime pocos threads."
- "delay suspende, Thread.sleep bloquea - esa es la diferencia en una linea."
- "Las corrutinas no son mas rapidas en computo, son mas eficientes en concurrencia."

## Trampas tipicas de entrevista

**"?Una corrutina es mas rapida que un thread?"** - No en ejecucion de codigo: el trabajo de CPU tarda lo mismo. Se gana eficiencia de recursos para manejar mucha concurrencia de espera, no velocidad de computo.

**"?Que pasa si llamo codigo bloqueante dentro de una corrutina en Dispatchers.Default?"** - Se bloquea un hilo real del pool de Default, que es limitado (= numero de cores), y eso puede tapar el trabajo de CPU de toda la app. Ese codigo bloqueante deberia ir en Dispatchers.IO.

**"?Las corrutinas reemplazan a los threads?"** - No, corren sobre ellos. El dispatcher decide en que thread o pool corre cada corrutina.
