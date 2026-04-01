# Docker Guide - TMR2026 Point-LIO + Unitree L1

Documentación completa para construir, ejecutar y usar el contenedor Docker con Point-LIO y Unitree L1 LIDAR en ROS Noetic.

---

## 📋 Tabla de Contenidos

1. [Stack Tecnológico](#stack-tecnológico)
2. [Construcción y Ejecución](#construcción-y-ejecución)
3. [Comandos ROS](#comandos-ros)
4. [Limitación de Recursos](#limitación-de-recursos)
5. [Docker Compose (Recomendado)](#docker-compose-recomendado)

---

## Stack Tecnológico

El Docker contiene lo siguiente:

- **Base**: `arm64v8/ros:noetic-robot` (ROS 1)
- **SDK**: Unitree L1 (`unilidar_sdk`)
- **SLAM**: Point-LIO para Unitree L1 (`point_lio_unilidar`)
- **Librerías**: GTSAM 4.0.3, PCL, OpenCV, etc.

### ⚠️ Importante: Solo ROS 1 (Noetic)

Se eliminan automáticamente los componentes de ROS 2 del SDK de Unitree:
```dockerfile
RUN rm -rf $WORKSPACE/src/unilidar_sdk/unitree_lidar_ros/src/unitree_lidar_ros2
```

---

## Construcción y Ejecución

### 1️⃣ Construir la Imagen (Build)

```bash
# Desde la raíz del proyecto TMR2026
cd /home/uav/TMR2026

# Construir la imagen
docker build -t tmr2026:point-lio-l1 .
```

**Esto toma tiempo** (10-20 minutos):
- Descarga base de ROS Noetic
- Compila GTSAM
- Descarga unilidar_sdk y point_lio_unilidar
- Compila todo con catkin

**Tip**: Si está en laptop, cambiar a "performance" mode en batería.

---

### 2️⃣ Ejecutar el Contenedor (Run)

#### Opción A: Ejecución Básica

```bash
docker run -it \
  --privileged \
  --network host \
  -v /dev:/dev \
  tmr2026:point-lio-l1
```

#### Opción B: Con Limitación de RAM (Recomendado)

```bash
docker run -it \
  --privileged \
  --network host \
  -v /dev:/dev \
  -m 3g \
  --memory-swap 3g \
  --name my_tmr \
  tmr2026:point-lio-l1
```

**Parámetros explicados**:

| Parámetro | Función |
|-----------|---------|
| `-it` | Terminal interactiva (i=interactive, t=tty) |
| `--privileged` | Acceso completo a hardware (LIDAR, USB, etc) |
| `--network host` | Comparte red del host (mejor latencia, necesario para LIDAR) |
| `-v /dev:/dev` | Mapea dispositivos del host (`/dev/ttyUSB0`, LIDAR, etc) |
| `-m 3g` | Limita RAM a 3 GB |
| `--memory-swap 3g` | Sin swap (usa solo RAM pura) |
| `--name my_tmr` | Nombre del contenedor (evita crear instancias duplicadas) |

---

### 3️⃣ Entrar en Contenedor Existente (Exec)

**Problema**: `docker run` crea un contenedor NUEVO cada vez.

**Solución**: Usar `docker exec` para entrar en el MISMO contenedor:

```bash
# Terminal 1 (corre el contenedor)
docker run -it --name my_tmr --privileged --network host -v /dev:/dev -m 3g --memory-swap 3g tmr2026:point-lio-l1

# Terminal 2, 3, 4... (entran en el MISMO)
docker exec -it my_tmr bash
```

---

### 4️⃣ Gestionar Contenedores

```bash
# Ver contenedores activos
docker ps

# Ver todos (incluyendo detenidos)
docker ps -a

# Eliminar contenedores no usados
docker rm nombre_contenedor

# Eliminar imagen
docker rmi tmr2026:point-lio-l1

# Ver espacios usados
docker system df
```

---

### 5️⃣ Verificar Límites de RAM

Dentro del contenedor:

```bash
# Ver RAM disponible
free -h

# Monitorear uso en vivo
watch -n 1 free -h
```

---

## Comandos ROS

### Dentro del Contenedor

Primero, sourcer el workspace:

```bash
source /opt/ros/noetic/setup.bash
source /root/catkin_ws/devel/setup.bash
```

(Ya viene automatizado en `.bashrc` del Dockerfile)

---

### 1. Unitree L1 SDK - Sin Visualización

```bash
roslaunch unitree_lidar_ros run_without_rviz.launch
```

**Qué hace**:
- Inicia el driver del LIDAR
- Publica puntos en `/points_topic`
- **Sin RViz** (solo modo servidor)

---

### 2. Point-LIO Mapping - Mapping/SLAM

```bash
roslaunch point_lio_unilidar mapping_unilidar_l1.launch
```

**Qué hace**:
- Corre Point-LIO con Unitree L1
- **Mapea** en tiempo real
- Publica mapa y posición del robot
- Requiere que el LIDAR esté corriendo

**Órdenes típicas en otra terminal**:

```bash
# Ver topics disponibles
rostopic list

# Ver datos del LIDAR
rostopic echo /points_topic

# Ver posición estimada
rostopic echo /odometry
```

---

### 3. Iniciar ROS Master

```bash
roscore
```

(Normalmente Point-LIO lo inicia automáticamente)

---

### 4. Visualizar con RViz

```bash
rosrun rviz rviz
```

O si quieres la config de Point-LIO pre-configurada:

```bash
rosrun rviz rviz -d /root/catkin_ws/src/point_lio_unilidar/config/rviz_config.rviz
```

---

## Limitación de Recursos

### RAM

```bash
# 2 GB RAM
-m 2g --memory-swap 2g

# 3 GB RAM (Recomendado para Jetson)
-m 3g --memory-swap 3g

# 4 GB RAM
-m 4g --memory-swap 4g

# Con swap (4GB RAM + 2GB disco)
-m 4g --memory-swap 6g
```

### CPU

```bash
# Limitar a 2 cores
--cpus="2"

# Limitar a 50% de un core
--cpus="0.5"

# Limitar a cores específicos (0 y 1)
--cpuset-cpus="0,1"
```

### Ejemplo Completo (RAM + CPU)

```bash
docker run -it \
  --privileged \
  --network host \
  -v /dev:/dev \
  -m 3g \
  --memory-swap 3g \
  --cpus="3" \
  --name my_tmr \
  tmr2026:point-lio-l1
```

---

## Docker Compose (Recomendado)

### ¿Por qué Docker Compose?

| Opción | Ventajas | Desventajas |
|--------|----------|------------|
| **docker run** | Simple para una vez | Repite parámetros, fácil olvidar flags |
| **docker-compose** ✅ | Guardar config, reproducible, fácil de compartir | Un archivo más |

---

### Crear docker-compose.yml

En la raíz del proyecto (`/home/uav/TMR2026/`), crea:

```yaml
version: '3.8'

services:
  point_lio:
    build:
      context: .
      dockerfile: Dockerfile
    image: tmr2026:point-lio-l1
    container_name: tmr2026_pointlio

    environment:
      - DISPLAY=${DISPLAY:-:0}

    volumes:
      - /dev:/dev                          # Hardware
      - /tmp/.X11-unix:/tmp/.X11-unix:rw  # Gráficos (si quieres RViz)

    ports:
      - "8765:8765"

    tty: true
    stdin_open: true
    privileged: true
    shm_size: 10g

    # Limitación de recursos
    deploy:
      resources:
        limits:
          memory: 3g
          cpus: '3'
```

---

### Ejecutar con Docker Compose

```bash
# 1. Construir la imagen (primera vez)
docker-compose build

# 2. Iniciar el contenedor
docker-compose up -it

# 3. En otra terminal, entrar (si necesitas otra bash)
docker-compose exec point_lio bash

# 4. Detener
docker-compose down

# 5. Eliminar todo (imagen + contenedor + volúmenes)
docker-compose down -v
```

---

### Ventajas de docker-compose.yml

✅ **Todo guardado** en un archivo
✅ **Reproducible**: mismo comando siempre
✅ **Fácil compartir**: otros clonen y ejecuten igual
✅ **Escalable**: agregar servicios (DB, sensor bridge, etc)
✅ **Menos errores**: no olvidas flags

---

### Ejemplo: Agregar RViz a docker-compose.yml

```yaml
version: '3.8'

services:
  point_lio:
    build:
      context: .
      dockerfile: Dockerfile
    image: tmr2026:point-lio-l1
    container_name: tmr2026_pointlio

    environment:
      - DISPLAY=${DISPLAY:-:0}

    volumes:
      - /dev:/dev
      - /tmp/.X11-unix:/tmp/.X11-unix:rw
      - ./ros_bags:/root/bags                    # Para guardar bags

    tty: true
    stdin_open: true
    privileged: true
    shm_size: 10g

    deploy:
      resources:
        limits:
          memory: 3g
          cpus: '3'

    # Opcionalmente: red específica para LIDAR
    networks:
      - default

networks:
  default:
    driver: bridge
```

Luego:

```bash
# Construir y ejecutar
docker-compose up

# En otra terminal
docker-compose exec point_lio bash

# Dentro del contenedor
roslaunch point_lio_unilidar mapping_unilidar_l1.launch
```

---

## Workflow Completo (Recomendado)

### Primera vez (Setup)

```bash
cd /home/uav/TMR2026

# 1. Construir imagen
docker-compose build

# 2. Iniciar contenedor
docker-compose up -it
```

### Usar el contenedor (desarrollo)

Terminal 1 (contenedor principal):
```bash
docker-compose up
```

Terminal 2, 3, ... (bash adicionales):
```bash
docker-compose exec point_lio bash

# Dentro:
source /opt/ros/noetic/setup.bash
source /root/catkin_ws/devel/setup.bash

# Lanzar Point-LIO + Mapping
roslaunch point_lio_unilidar mapping_unilidar_l1.launch
```

Terminal 4 (si quieres RViz):
```bash
docker-compose exec point_lio bash

# Dentro:
rosrun rviz rviz
```

### Detener

```bash
# Ctrl+C en la terminal de docker-compose up
# O en otra:
docker-compose down
```

---

## Troubleshooting

### ❌ Error: "Unable to locate package unitree_lidar_ros2"

**Solución**: Ya está arreglado en el Dockerfile (línea 57-58 lo elimina).

### ❌ Error: "LIDAR no se detecta"

```bash
# Revisar hardware
ls -la /dev/ttyUSB*

# Revisar permisos
docker run --privileged  # Asegurar que está

# Ver topics
rostopic list
```

### ❌ Ram llena

```bash
# Dentro del contenedor
free -h
top

# Limitar en docker-compose.yml:
deploy:
  resources:
    limits:
      memory: 2g  # Reducir a 2GB
```

### ❌ Contenedor lento

```bash
# Limitar CPU si es necesario
--cpus="2"

# Ver stats
docker stats tmr2026_pointlio
```

---

## Referencias Rápidas

### Build

```bash
docker-compose build
# O con reconstrucción forzada
docker-compose build --no-cache
```

### Run

```bash
docker-compose up -it
```

### Exec

```bash
docker-compose exec point_lio bash
docker-compose exec point_lio roslaunch point_lio_unilidar mapping_unilidar_l1.launch
```

### Clean

```bash
docker-compose down             # Detener
docker-compose down -v          # Detener + eliminar volúmenes
docker system prune             # Limpiar imágenes/contenedores no usados
```

---

## Stack Visual

```
┌─────────────────────────────────────────────┐
│         Host (tu laptop/Jetson)             │
├─────────────────────────────────────────────┤
│                                             │
│  ┌───────────────────────────────────────┐  │
│  │      Docker Container                 │  │
│  │   (3GB RAM, 3 CPUs, ROS Noetic)      │  │
│  │                                       │  │
│  │  - Unitree L1 SDK                    │  │
│  │  - Point-LIO SLAM                    │  │
│  │  - GTSAM, PCL, OpenCV                │  │
│  │                                       │  │
│  │  Procesos:                            │  │
│  │  → roscore                            │  │
│  │  → unitree_lidar_driver               │  │
│  │  → point_lio_node                     │  │
│  │  → (RViz opcional)                    │  │
│  │                                       │  │
│  └───────────────────────────────────────┘  │
│                   ↑                         │
│         Volúmenes montados:                │
│         - /dev → hardware                  │
│         - /tmp/.X11-unix → display         │
│                                             │
└─────────────────────────────────────────────┘
       ↓
   LIDAR Unitree L1
   (USB Serial: /dev/ttyUSB0)
```

---

## Checklist de Configuración

- [ ] Dockerfile limpio (solo ROS Noetic, sin ROS2)
- [ ] docker-compose.yml creado
- [ ] Imagen construida (`docker-compose build`)
- [ ] Contenedor ejecutándose (`docker-compose up`)
- [ ] RAM limitada a 3GB
- [ ] LIDAR detectado (`/dev/ttyUSB*`)
- [ ] rostopic list muestra topics
- [ ] Point-LIO lanzado correctamente
- [ ] RViz (opcional) visualiza el mapa

---

## Comandos Más Usados

```bash
# Construir
docker-compose build

# Ejecutar
docker-compose up -it

# Entrar en contenedor existente
docker-compose exec point_lio bash

# Dentro del contenedor: Unitree L1 sin viz
roslaunch unitree_lidar_ros run_without_rviz.launch

# Dentro del contenedor: Point-LIO Mapping
roslaunch point_lio_unilidar mapping_unilidar_l1.launch

# Ver recursos
docker stats tmr2026_pointlio

# Detener todo
docker-compose down
```

---

**Última actualización**: 2026-04-01
**Autor**: Claude Code + TMR2026 Team
