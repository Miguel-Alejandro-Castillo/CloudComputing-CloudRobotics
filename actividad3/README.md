# Actividad 3 - SimuPi + Node-RED

Entorno IoT desarrollado con **SimuPi**, **Node-RED**, **MQTT**, **Docker Compose** y **FlowFuse Dashboard**.

El proyecto implementa un escenario personalizado de un **tanque de agua con bomba**, con monitorización del nivel, la turbidez y la presión del agua, automatización de la bomba y alarmas visuales en un dashboard.

## Arquitectura

El entorno está compuesto por dos servicios Docker:

- **SimuPi**: simulador de Raspberry Pi y broker MQTT.
- **Node-RED**: procesamiento de sensores, automatización y dashboard.

```text
SimuPi
  |
  | MQTT :1883
  v
Node-RED
  |
  v
FlowFuse Dashboard
```

Dentro de la red de Docker, Node-RED se conecta al broker mediante el host `simupi` y el puerto `1883`.

## Estructura del proyecto

```text
actividad3/
├── docker-compose.yml
├── Dockerfile.simupi
├── README.md
├── simupi/                    # Submódulo Git
├── simupi-data/               # Datos persistentes de SimuPi
│   ├── state.json
│   └── scenarios/
│       └── water_tank.json
└── nodered/
    ├── Dockerfile
    └── data/
        └── flows.json
```

### `simupi/`

Contiene el proyecto SimuPi como **submódulo Git**. Es una dependencia externa y no forma parte del código desarrollado para esta actividad.

### `simupi-data/`

Contiene el estado persistente de SimuPi y los escenarios disponibles en la aplicación.

El escenario propio de esta actividad se encuentra en:

```text
simupi-data/scenarios/water_tank.json
```

### `nodered/`

Contiene la imagen y los datos persistentes de Node-RED. El Dockerfile instala:

- `@flowfuse/node-red-dashboard`
- `node-red-node-pi-gpiod`

Los flujos están almacenados en:

```text
nodered/data/flows.json
```

## Requisitos

- Git
- Docker
- Docker Compose

No es necesario instalar Go, Node.js ni Node-RED directamente en el equipo host.

## Clonar y ejecutar la Actividad 3

La Actividad 3 forma parte del repositorio `CloudComputing-CloudRobotics`. Por ello, primero se debe clonar el repositorio completo y después acceder al directorio `actividad3`.

### Primera instalación

Clonar el repositorio junto con el submódulo de SimuPi:

```bash
git clone --recurse-submodules https://github.com/Miguel-Alejandro-Castillo/CloudComputing-CloudRobotics.git
```

Entrar en el directorio de la Actividad 3:

```bash
cd CloudComputing-CloudRobotics/actividad3
```

Construir e iniciar los servicios:

```bash
docker compose up -d --build
```

Comprobar que ambos contenedores están en ejecución:

```bash
docker compose ps
```

### Repositorio clonado sin el submódulo

Si el repositorio ya fue clonado con el comando siguiente:

```bash
git clone https://github.com/Miguel-Alejandro-Castillo/CloudComputing-CloudRobotics.git
```

el contenido de `actividad3/simupi` todavía no estará inicializado.

Desde la raíz del repositorio, descargar e inicializar los submódulos:

```bash
cd CloudComputing-CloudRobotics
git submodule update --init --recursive
```

Después, entrar en la Actividad 3 y levantar el entorno:

```bash
cd actividad3
docker compose up -d --build
```

### Verificar el submódulo

Desde la raíz del repositorio:

```bash
git submodule status
```

Debe aparecer `actividad3/simupi` asociado al commit de SimuPi utilizado por el proyecto.

## Acceso a los servicios

### SimuPi

- Interfaz web: `http://localhost:5000`
- MQTT desde el host: `localhost:1884`
- MQTT desde Node-RED: host `simupi`, puerto `1883`

### Node-RED

- Editor: `http://localhost:1888`
- Dashboard: `http://localhost:1888/dashboard`

## Escenario: tanque de agua con bomba

El escenario personalizado supervisa tres variables:

| Sensor | Variable | Modo | Rango | Unidad |
|---|---|---|---:|---|
| `levelWater` | Nivel de agua | `triangle` | 0-100 | % |
| `turbidity` | Turbidez | `walk` | 0-100 | NTU |
| `waterPressure` | Presión del agua | `walk` | 0-3,5 | bar |

Tópicos MQTT:

```text
sensor/levelWater
sensor/turbidity
sensor/waterPressure
```

### Control de la bomba

La bomba utiliza el tópico:

```text
gpio/out/27
```

Valores enviados:

```text
1 = encendida
0 = apagada
```

La automatización utiliza una banda de histéresis:

```text
Nivel <= 5 %  -> encender bomba
Nivel >= 95 % -> apagar bomba
Entre 5 % y 95 % -> mantener el estado anterior
```

Esta separación evita encendidos y apagados repetidos ante pequeñas variaciones de la medición.

### Alarmas

El dashboard evalúa las siguientes condiciones:

```text
turbidity >= 20 NTU
waterPressure >= 3 bar
```

Cuando no existen valores fuera de rango, se muestra **Funcionamiento normal**. Si existe una anomalía, el dashboard presenta una caja roja con la variable y el valor correspondientes.

## Persistencia

SimuPi utiliza este bind mount:

```yaml
- ./simupi-data:/app/data
```

Los escenarios creados o modificados desde SimuPi permanecen almacenados en el host:

```text
simupi-data/scenarios/
```

Node-RED utiliza:

```yaml
- ./nodered/data:/data
```

Por tanto, el siguiente ciclo no elimina los datos persistentes:

```bash
docker compose down
docker compose up -d
```

> No eliminar `simupi-data/` ni `nodered/data/` si se desea conservar el estado, los escenarios, los flujos y la configuración.

## Comandos útiles

### Reconstruir todo

```bash
docker compose up -d --build
```

### Reconstruir solo Node-RED

```bash
docker compose build --no-cache nodered
docker compose up -d nodered
```

### Reconstruir solo SimuPi

```bash
docker compose build --no-cache simupi
docker compose up -d simupi
```

### Ver logs

```bash
docker compose logs -f
```

Logs de SimuPi:

```bash
docker compose logs -f simupi
```

Logs de Node-RED:

```bash
docker compose logs -f nodered
```

### Verificar los escenarios dentro del contenedor

```bash
docker compose exec simupi find /app/data/scenarios -type f -print
```

### Detener el entorno

```bash
docker compose down
```

## Actualizar el submódulo de SimuPi

Desde la raíz de `CloudComputing-CloudRobotics`, actualizar el submódulo con:

```bash
git submodule update --remote actividad3/simupi
```

Registrar la nueva referencia del submódulo en el repositorio principal:

```bash
git add actividad3/simupi
git commit -m "chore: update SimuPi submodule"
git push
```

## Solución de problemas

### La carpeta `simupi/` está vacía después de clonar

Desde la raíz del repositorio ejecutar:

```bash
git submodule update --init --recursive
```

### Node-RED no conecta con MQTT

Comprobar que el nodo MQTT utiliza:

```text
Host: simupi
Puerto: 1883
```

Para la comunicación entre contenedores no debe utilizarse `localhost:1884`.

### El escenario no aparece en SimuPi

Comprobar que el escenario existe dentro del contenedor:

```bash
docker compose exec simupi ls -la /app/data/scenarios
```

Después, reiniciar SimuPi:

```bash
docker compose restart simupi
```

### Los cambios de los escenarios desaparecen

Comprobar que `docker-compose.yml` contiene:

```yaml
- ./simupi-data:/app/data
```

No debe montarse el submódulo `./simupi` sobre `/app/data`, ya que `simupi/` contiene el código fuente y no los datos persistentes.

## Dependencia externa

SimuPi es una dependencia externa utilizada para simular el entorno IoT. El proyecto se referencia mediante un submódulo Git para mantener separado el código externo del código desarrollado para esta actividad.

Repositorio original de SimuPi:

[Ver repositorio original de SimuPi](https://github.com/seguren/simupi)

## Autores

Grupo 16

- Miguel Alejandro Castillo Figueroa
- Evelyn Leonor Ramirez
