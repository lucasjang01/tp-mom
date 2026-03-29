# Trabajo Práctico - Middlewares Orientados a Mensajes

Los middlewares orientados a mensajes (MOMs) son un recurso importante para el control de la complejidad en los sistemas distribuídos, puesto que permiten a las distintas partes del sistema comunicarse abstrayéndose de problemas como los cambios de ubicación, fallos, performance y escalabilidad.

En este repositorio se proveen conjuntos de pruebas para los dos formas más comunes de organización de la comunicación sobre colas, que en RabbitMQ se denominan Work Queues y Exchanges.

Se recomienda familiarizarse con estos conceptos leyendo la documentación de RabbitMQ y siguiendo los [tutoriales introductorios](https://www.rabbitmq.com/tutorials).

## Condiciones de Entrega

El código de este repositorio se agrupa en dos carpetas, una para Python y otra para Golang. Los estudiantes deberán elegir **sólo uno** de estos lenguajes y completar la implementación de las interfaces de middleware provistas con el objetivo de pasar las pruebas asociadas.

Al momento de la evaluación y ejecución de las pruebas se **descartarán** los cambios realizados a todos los archivos, a excepción de:

**Python:** `/python/src/common/middleware/middleware_rabbitmq.py` 

**Golang:** `/golang/internal/factory/*/*.go` 

## Ejecución

`make up` : Inicia contenedores de RabbitMQ  y de pruebas de integración. Comienza a seguir los logs de las pruebas.

`make down`:   Detiene los contenedores de pruebas y destruye los recursos asociados.

`make logs`: Sigue los logs de todos los contenedores en un solo flujo de salida.

`make local`: Ejecuta las pruebas de integración desde el Host, facilitando el desarrollo. Se explica con mayor detalle dentro de su sección.

## Pruebas locales desde el Host

Habiendo iniciado el contenedor de RabbitMQ o configurado una instancia local del mismo pueden ejecutarse las pruebas sin necesidad de detener y reiniciar los contenedores ejecutando `make local`, siempre que se cumplan los siguientes requisitos.

### Python
Instalar una versión de Python superior a `3.14`. Se recomienda emplear un gestor de versiones, como ser `pyenv`.
Instalar los dependencias de la suite de pruebas:
`pip install -r python/src/tests/requirements.txt`

### Golang
Instalar una versión de Golang superior a `1.24`.
Instalar los dependencias de la suite de pruebas:
`go mod download`

---

## Resolución (Python)

- **Work Queue**: múltiples productores y consumidores comparten una cola nombrada. Cada mensaje es procesado por exactamente un consumidor. Es útil para distribuir trabajo entre workers.
- **Direct Exchange**: los mensajes se enrutan por routing key. Cada consumidor se suscribe a una o más keys y recibe una copia propia de cada mensaje que le corresponde, habilitando patrones de broadcast.

### Decisiones de diseño

**Clase base `_RabbitMQBase`**

Los dos patrones Work Queue y Direct Exchange comparten la lógica de conexión, el loop de consumo y el cierre de recursos. Para no repetir código, esta lógica se encapsuló en una clase base interna, de la que heredan las dos implementaciones.

**Patrón Work Queue**

Se declara una cola durable y se configura `prefetch_count=1` para garantizar distribución justa entre consumidores competidores: RabbitMQ no envía un segundo mensaje a un consumidor hasta que éste haga `ack` del primero. El envío usa el exchange default (`''`), que enruta directamente por nombre de cola.

**Patrón Direct Exchange**

Se declara un exchange de tipo `direct`. Cada instancia de consumidor crea su propia cola exclusiva con nombre generado por el servidor, que se elimina automaticamente al desconectarse. Luego bindea esa cola al exchange por cada routing key correspondiente. Esto garantiza que si dos consumidores se suscriben a la misma key, cada uno recibe su propia copia del mensaje (fan-out), a diferencia de una work queue donde se repartirian.

**Adaptación del callback**

La interfaz del middleware expone `(message, ack, nack)` pero pika internamente llama a los callbacks con `(ch, method, properties, body)`. El método `_wrap_callback` hace de traductor entre ambas firmas.

**`stop_consuming` desde adentro del callback**

Los tests llaman a `stop_consuming()` desde adentro del callback de consumo. Esto es seguro en pika porque `BlockingConnection` es single-threaded: `stop_consuming()` solo setea un flag interno que pika chequea después de que el callback retorna, saliendo del loop limpiamente.

**Manejo de errores**

Cada operacion mapea las excepciones de pika a las excepciones definidas en la interfaz: errores de conexión levantan `MessageMiddlewareDisconnectedError`, errores de mensajería levantan `MessageMiddlewareMessageError`, y errores al cerrar levantan `MessageMiddlewareCloseError`. El cierre de canal y conexion se realiza en bloques independientes para que un fallo en uno no impida cerrar el otro.
