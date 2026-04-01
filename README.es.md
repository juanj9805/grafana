# Stack de Monitoreo con Grafana

Stack de monitoreo basado en Docker usando Prometheus, Grafana y Node Exporter.

## Arquitectura

```
Node Exporter → Prometheus → Grafana
  (recolecta)   (almacena)  (visualiza)
```

| Servicio      | Puerto | Rol                                      |
|---------------|--------|------------------------------------------|
| Grafana       | 3000   | Visualización y dashboards               |
| Prometheus    | 9090   | Almacenamiento y consulta de métricas    |
| Node Exporter | 9100   | Expone métricas del sistema host         |

---

## Requisitos

- Docker y Docker Compose instalados en tu servidor
- Puertos 3000, 9090 y 9100 abiertos en el firewall

---

## Paso 1 — Levantar el stack

```bash
cd monitoring
docker compose up -d
```

Verificar que todos los contenedores están corriendo:

```bash
docker ps
```

Deberías ver `grafana`, `prometheus` y `node-exporter` con estado `Up`.

---

## Paso 2 — Verificar los targets de Prometheus

Abre en tu navegador:

```
http://<ip-de-tu-servidor>:9090/targets
```

Tanto `prometheus` como `node-exporter` deben mostrar estado **UP**.

El job `docker` puede aparecer como DOWN — esto es esperado a menos que las métricas de Docker estén habilitadas explícitamente en el host.

---

## Paso 3 — Conectar Grafana a Prometheus

1. Abre Grafana: `http://<ip-de-tu-servidor>:3000`
2. Inicia sesión con `admin` / `admin` — se te pedirá cambiar la contraseña
3. Ve a **Connections → Data sources → Add data source**
4. Selecciona **Prometheus**
5. Establece la URL:
   ```
   http://prometheus:9090
   ```
6. Haz clic en **Save & test**

Resultado esperado: `Successfully queried the Prometheus API.`

> Grafana y Prometheus se comunican usando el nombre del contenedor `prometheus` porque ambos están en la misma red Docker (`monitoring`).

---

## Paso 4 — Importar un dashboard pre-construido

1. Ve a **Dashboards → New → Import**
2. Ingresa el ID del dashboard:
   ```
   1860
   ```
3. Haz clic en **Load**
4. En el campo Prometheus, selecciona tu fuente de datos Prometheus
5. Haz clic en **Import**

Esto importa el dashboard **Node Exporter Full** — un dashboard de la comunidad con paneles para CPU, memoria, disco, red y más.

---

## Paso 5 — Explorar tus métricas

Una vez cargado el dashboard puedes ver:

- Uso de CPU por núcleo
- Uso de memoria y swap
- Throughput de lectura/escritura de disco
- Tráfico de red entrante/saliente
- Carga promedio del sistema

También puedes consultar métricas manualmente en:

```
http://<ip-de-tu-servidor>:9090
```

Ejemplos de consultas PromQL:

```promql
# Porcentaje de uso de CPU
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memoria disponible en GB
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024

# Porcentaje de uso de disco
100 - ((node_filesystem_avail_bytes * 100) / node_filesystem_size_bytes)
```

---

## Detener el stack

```bash
cd monitoring
docker compose down
```

Para también eliminar los datos almacenados de Grafana:

```bash
docker compose down -v
```
