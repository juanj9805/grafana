# Cómo Funciona el Stack de Monitoreo

Este documento explica los conceptos detrás del stack — no solo cómo ejecutarlo, sino por qué está construido así y qué hace cada pieza realmente.

---

## El Problema que Resuelve

Cuando corres un servidor, pasan cosas que no puedes ver en tiempo real:
- El CPU se dispara cuando aumenta el tráfico
- La memoria se llena lentamente hasta que la app se cae
- El disco se agota a las 3am
- Un servicio se cae y nadie se da cuenta por horas

Necesitas un sistema que **vigile continuamente tu servidor**, almacene esos datos en el tiempo, y te permita **visualizarlos y generar alertas**. Eso es exactamente lo que hace este stack.

---

## Qué es Docker (y por qué usarlo aquí)

### La idea central

Normalmente, para correr Prometheus en un servidor tendrías que:
1. Descargar el binario
2. Instalar sus dependencias
3. Configurarlo para que inicie al arrancar
4. Esperar que no conflictúe con algo más ya instalado

Docker lo resuelve diferente. Un **contenedor** es un proceso aislado que empaca la aplicación y todo lo que necesita para correr — dependencias, configuración, runtime — en una sola unidad. Corre en tu servidor pero está aislado de todo lo demás.

Piénsalo así: tu servidor es un edificio, y cada contenedor es un apartamento. Cada apartamento tiene su propia cocina, baño y servicios. Comparten la misma infraestructura del edificio (el kernel de Linux) pero no se interfieren entre sí.

### Imagen vs contenedor

- **Imagen**: el plano (solo lectura). Como una clase en programación.
- **Contenedor**: una instancia corriendo de esa imagen. Como un objeto instanciado de una clase.

Cuando ejecutas `docker compose up`, Docker descarga las imágenes (si no están en caché) y arranca contenedores a partir de ellas.

### Docker Compose

Correr contenedores uno por uno con `docker run` se vuelve tedioso cuando tienes múltiples servicios. **Docker Compose** te permite definir todos tus servicios en un solo archivo `docker-compose.yml` y arrancarlos/detenerlos juntos con un comando.

Tu `docker-compose.yml` define tres servicios: Prometheus, Grafana y Node Exporter. Cada uno se convierte en un contenedor cuando ejecutas `docker compose up`.

### Redes de Docker

Por defecto, los contenedores están aislados — no pueden hablar entre sí. Cuando defines una red en `docker-compose.yml`, Docker crea una red virtual privada y conecta los contenedores especificados a ella.

En tu setup, los tres contenedores están conectados a una red llamada `monitoring`. Dentro de esta red, los contenedores pueden alcanzarse entre sí usando su **nombre de contenedor** como hostname. Por eso en `prometheus.yml` escribes `node-exporter:9100` en lugar de una IP — Docker resuelve ese nombre al contenedor correcto automáticamente.

Es el mismo concepto que DNS, pero limitado a tu red Docker privada.

### Volúmenes de Docker

Los contenedores son efímeros por defecto — si detienes y eliminas un contenedor, todos los datos dentro se pierden. Un **volumen** es un área de almacenamiento persistente que vive fuera del contenedor y sobrevive los reinicios.

En tu setup, Grafana usa un volumen llamado `grafana-storage` para persistir tus dashboards y configuración. Prometheus no tiene un volumen configurado, lo que significa que si lo reinicias, las métricas históricas se pierden — algo a mejorar más adelante.

---

## Los Tres Servicios

### Node Exporter — el sensor

Node Exporter es un programa que lee métricas a nivel de sistema desde el kernel de Linux y las expone por HTTP.

Tu kernel de Linux registra todo lo que pasa en la máquina: cuánto CPU usa cada proceso, cuánta memoria hay libre, cuántos bytes se escribieron al disco, cuántos paquetes pasaron por la interfaz de red. Estos datos viven en archivos virtuales especiales bajo `/proc` y `/sys`.

Node Exporter lee esos archivos y sirve los datos en:
```
http://node-exporter:9100/metrics
```

Si abres esa URL verás miles de líneas crudas como:
```
node_cpu_seconds_total{cpu="0",mode="idle"} 12345.67
node_memory_MemAvailable_bytes 2147483648
node_filesystem_avail_bytes{mountpoint="/"} 53687091200
```

Este formato se llama **formato de exposición de Prometheus** — un formato de texto plano que Prometheus sabe leer.

Node Exporter no almacena nada. Solo lee y expone. Es un sensor, no una base de datos.

Nota en `docker-compose.yml` que Node Exporter monta `/proc`, `/sys` y `/` del host dentro del contenedor:

```yaml
volumes:
  - /proc:/host/proc:ro
  - /sys:/host/sys:ro
  - /:/rootfs:ro
```

Esto es necesario porque Node Exporter corre dentro de un contenedor, que está aislado del host. Sin estos montajes, solo vería las métricas (vacías) del propio contenedor, no las de tu servidor real. El `:ro` significa solo lectura — el contenedor puede leer estas rutas pero no modificarlas. Esto es un límite de seguridad.

---

### Prometheus — la base de datos

Prometheus es una **base de datos de series de tiempo**. Una base de datos de series de tiempo almacena valores indexados por tiempo — cada punto de dato tiene un valor y un timestamp.

Esto es diferente a una base de datos relacional como PostgreSQL. En vez de filas y columnas representando entidades, tienes **métricas** que cambian en el tiempo. Por ejemplo:

```
node_memory_MemAvailable_bytes a las 12:00:00 → 2.1 GB
node_memory_MemAvailable_bytes a las 12:00:05 → 2.0 GB
node_memory_MemAvailable_bytes a las 12:00:10 → 1.9 GB
```

Eso es una serie de tiempo — la misma métrica rastreada a través del tiempo.

#### Cómo Prometheus recolecta datos — el modelo pull

La mayoría de sistemas envían (push) datos a un recolector central. Prometheus hace lo opuesto: **jala** (scrapes) datos de los targets según un horario.

Cada 5 segundos (configurado en `prometheus.yml` con `scrape_interval: 5s`), Prometheus hace una petición HTTP GET al endpoint `/metrics` de cada target, parsea la respuesta y almacena los valores con un timestamp.

Este modelo pull tiene ventajas:
- Puedes saber exactamente cuándo un target se cayó (dejó de responder)
- Los targets no necesitan saber dónde está Prometheus — solo exponen un endpoint
- Fácil de probar manualmente (puedes hacer curl a cualquier target directamente)

#### prometheus.yml

Este archivo le dice a Prometheus qué hacer scrape:

```yaml
scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

`node-exporter:9100` resuelve al contenedor de Node Exporter en la red Docker. Prometheus consulta `http://node-exporter:9100/metrics` cada 5 segundos.

#### PromQL

Prometheus viene con su propio lenguaje de consulta llamado **PromQL**. Lo usas para hacerle preguntas a tus métricas. Por ejemplo:

```promql
node_memory_MemAvailable_bytes
```
Retorna la memoria disponible actual.

```promql
rate(node_cpu_seconds_total{mode="idle"}[5m])
```
Retorna la tasa por segundo de tiempo idle de CPU durante los últimos 5 minutos — de esto puedes calcular el porcentaje de uso de CPU.

PromQL está diseñado para datos de series de tiempo. No es SQL, pero el modelo mental es similar: estás filtrando y transformando datos, solo que a través del tiempo en vez de filas.

---

### Grafana — la capa de visualización

Grafana es una herramienta de dashboards. No recolecta ni almacena ninguna métrica — se conecta a fuentes de datos (como tu Prometheus) y las consulta para mostrar gráficas, gauges, tablas y alertas.

Cuando agregaste Prometheus como fuente de datos con URL `http://prometheus:9090`, le dijiste a Grafana: "cuando necesites datos, pregúntale a Prometheus en esta dirección." Nuevamente, resolución de nombre de contenedor via la red Docker.

Cuando cargas un dashboard, cada panel ejecuta una consulta PromQL contra Prometheus y renderiza el resultado como una gráfica. Los datos siempre son en vivo — Grafana consulta Prometheus en cada actualización.

#### Por qué importar el dashboard ID 1860

El dashboard 1860 es **Node Exporter Full**, mantenido por la comunidad. Alguien ya escribió todas las consultas PromQL para métricas comunes del sistema y las organizó en un layout limpio. Obtienes un dashboard de calidad de producción en segundos en vez de construirlo desde cero.

Este es un patrón que verás en todo el ecosistema de Prometheus — un formato estándar de métricas significa que los dashboards son reutilizables en cualquier infraestructura.

---

## Cómo se Conectan los Tres Servicios

```
┌─────────────────────────────────────────────────┐
│           Red Docker "monitoring"               │
│                                                 │
│  ┌──────────────┐    scrapes     ┌────────────┐ │
│  │ Node Exporter│ ◄────────────  │ Prometheus │ │
│  │   :9100      │  cada 5s       │   :9090    │ │
│  └──────────────┘                └────────────┘ │
│                                        ▲        │
│                                        │consulta │
│                                   ┌────────┐    │
│                                   │Grafana │    │
│                                   │  :3000 │    │
│                                   └────────┘    │
└─────────────────────────────────────────────────┘
          ▲                              ▲
          │                             │
     (opcional)                   tú, en el navegador
   métricas crudas
```

El flujo para cada panel que ves en Grafana:

1. Abres un dashboard en tu navegador
2. Grafana envía una consulta PromQL a Prometheus (`http://prometheus:9090`)
3. Prometheus busca los datos de series de tiempo almacenados
4. Grafana recibe los puntos de datos y renderiza la gráfica
5. Cada 30 segundos (por defecto), Grafana refresca y repite

Mientras tanto, en segundo plano:

1. Cada 5 segundos, Prometheus hace scrape a `http://node-exporter:9100/metrics`
2. Node Exporter lee `/proc` y `/sys` del host y retorna las métricas
3. Prometheus almacena los valores con timestamps

---

## Por Qué Esta Arquitectura

Puede que te preguntes: ¿por qué tres servicios separados? ¿Por qué no un programa que haga todo?

Este es el principio de **separación de responsabilidades** aplicado a infraestructura:

- Node Exporter solo sabe leer métricas del sistema. Hace una cosa bien.
- Prometheus solo sabe hacer scrape, almacenar y consultar datos de series de tiempo. Hace una cosa bien.
- Grafana solo sabe visualizar datos de diversas fuentes. Hace una cosa bien.

Porque están desacoplados:
- Puedes reemplazar Grafana con otra herramienta de visualización sin tocar Prometheus
- Puedes agregar otros exporters (bases de datos, servidores web, tu propia app) sin cambiar nada más
- Puedes correr múltiples instancias de Prometheus haciendo scrape a los mismos exporters
- Cada servicio puede escalarse, actualizarse o reiniciarse de forma independiente

Este es el mismo principio detrás de los microservicios — componentes pequeños y enfocados conectados por interfaces bien definidas (en este caso, endpoints HTTP).

---

## Qué Explorar a Continuación

Ahora que entiendes cómo funciona, estos son los siguientes pasos naturales:

1. **Escribe tus propias consultas PromQL** — ve a `http://<tu-ip>:9090` y explora las métricas que expone Node Exporter. Intenta responder: ¿cuál es mi uso actual de CPU? ¿Cuánto espacio de disco queda?

2. **Construye un panel desde cero en Grafana** — agrega un nuevo panel a un dashboard, escribe una consulta PromQL y elige un tipo de visualización. Aquí es donde realmente aprenderás PromQL.

3. **Agrega alertas** — configura reglas de alertas en Prometheus que se disparen cuando el CPU supere el 80% o el disco esté por debajo del 10%. Así es como funcionan los sistemas de on-call en producción.

4. **Instrumenta tu propia aplicación** — usa una librería cliente de Prometheus (disponible para Node.js, Python, Go, etc.) para exponer métricas personalizadas desde una app que construyas. Este es el caso de uso real en producción.
