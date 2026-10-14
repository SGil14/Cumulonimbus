# Cumulonimbus

**¿Está lloviendo donde voy a estar?** Cumulonimbus consulta en vivo los pluviómetros del SAB / IDIGER de Bogotá y te lo dice en una frase: dónde llueve ahora, si llueve cerca de un destino, si te va a mojar en tu ruta y cuánto ha llovido antes. Todo sobre un mapa, en el celular o en el computador, y con un histórico propio guardado en PostgreSQL para analizar la lluvia con el tiempo.

![Cumulonimbus con el diseño Clima, tema claro](docs/img/01-diseno-clima-claro.png)

> **Sobre las capturas de este documento.** Cuando escribí esta guía casi no llovía en Bogotá, así que las imágenes que muestran lluvia (zonas de color, radar encendido, el panel «7 de 71») usan **estaciones y ecos de radar simulados** para poder enseñar cómo se ve. Las imágenes del buscador, la ruta, el historial y la documentación de la API usan **datos reales** del momento de la captura.

---

## Contenido

1. [¿Para qué sirve?](#1-para-qué-sirve)
2. [Qué puedes hacer con la app](#2-qué-puedes-hacer-con-la-app)
3. [Cómo funciona por dentro](#3-cómo-funciona-por-dentro)
4. [Instalación paso a paso](#4-instalación-paso-a-paso)
5. [Cómo usar la app](#5-cómo-usar-la-app)
6. [Configuración (`.env`)](#6-configuración-env)
7. [La API](#7-la-api)
8. [Base de datos](#8-base-de-datos)
9. [Reglas y cálculos](#9-reglas-y-cálculos)
10. [Los cinco diseños](#10-los-cinco-diseños)
11. [Estructura del proyecto](#11-estructura-del-proyecto)
12. [Pruebas](#12-pruebas)
13. [Despliegue en la nube](#13-despliegue-en-la-nube)
14. [Límites que debes conocer](#14-límites-que-debes-conocer)
15. [Solución de problemas](#15-solución-de-problemas)
16. [Fuentes de datos y créditos](#16-fuentes-de-datos-y-créditos)
17. [Hoja de ruta](#17-hoja-de-ruta)

---

## 1. ¿Para qué sirve?

El IDIGER (Instituto Distrital de Gestión de Riesgos y Cambio Climático) mantiene en Bogotá una red de **71 pluviómetros** que reportan cuánta lluvia cae, cada pocos minutos. Esos datos son públicos en la página del SAB, pero están pensados para consultarse estación por estación. Cumulonimbus los convierte en respuestas:

| Pregunta | Qué responde la app |
|---|---|
| ¿Está lloviendo en Bogotá? | Cuántas de las 71 estaciones registran lluvia y en qué localidades. |
| ¿Llueve donde voy? | Escribes un barrio, lugar, dirección, esquina o coordenadas y te dice si llueve cerca, con las 3 estaciones más próximas. |
| ¿Me voy a mojar en el camino? | Calculas una ruta de un lugar a otro y la app marca los tramos donde llueve. |
| ¿Cuánto llovió antes? | Historial por horas o por días, para toda Bogotá, una localidad o un destino. |
| ¿De dónde viene la lluvia? | Capa opcional del radar meteorológico del IDIGER. |

**Solo consulta cuando tú se lo pides**: no se actualiza sola. Se actualiza al abrirla o al pulsar «Actualizar». (El histórico sí se llena solo mientras el servidor está encendido; ver [Recolector](#recolector-del-histórico).)

---

## 2. Qué puedes hacer con la app

### Ver dónde llueve ahora

Al abrir la app, el panel superior responde en una frase («Bogotá está seca» o «7 de 71 estaciones registran lluvia») y debajo lista las localidades donde llueve. En el mapa cada estación es un punto: los que están **secos** son puntos pequeños; los que **registran lluvia** se agrandan, se rodean de ondas y toman el color de su intensidad.

![Zonas de lluvia alrededor de cada estación](docs/img/11-zonas-de-lluvia.png)

**Colores de la intensidad (mm/h):**

| Color | Intensidad | Cuánto llueve |
|---|---|---|
| Azul / celeste | Ligera | menos de 2,5 mm/h |
| Amarillo | Moderada | de 2,5 a 7,6 mm/h |
| Naranja | Fuerte | de 7,6 a 50 mm/h |
| Rojo | Muy fuerte | 50 mm/h o más |

Alrededor de cada estación que llueve se pinta además una **zona de color difuminada** (de unos 2,5 km de radio en la lluvia ligera a unos 5,5 km en la muy fuerte). Es una estimación visual de la zona que cubre la medición: el pluviómetro mide un solo punto. La zona se desvanece al acercar el mapa para no taparlo.

Al tocar una estación aparece su ficha con intensidad actual, acumulado del día y hora de la última lectura.

### Consultar un destino

Escribe en el buscador cualquiera de estas cosas:

- Una **localidad**: `Chapinero`
- **Coordenadas**: `4.6486, -74.0628`
- Un **barrio o lugar**: `Portal 80`, `Galerías`
- Una **dirección**: `Carrera 7 # 55-25`
- Una **esquina**: `Calle 72 con Carrera 10`, `Av Boyacá con Calle 80`

![Resultado de buscar una esquina](docs/img/07-buscar-destino.png)

La respuesta dice con claridad si llueve cerca («Sí, llueve cerca» o «No llueve cerca»), indica el barrio y la localidad, y lista las **3 estaciones más cercanas** con su distancia, su estado y lo que han acumulado hoy. Si la app no encontró el número exacto de la dirección y usó la placa más cercana de la misma cuadra, te avisa que la ubicación es **aproximada**. El enlace «Ver cuándo ha llovido aquí» abre el historial de ese punto.

### Revisar una ruta (¿me mojo en el camino?)

Debajo del buscador está **«¿Vas de un lugar a otro? Revisa la ruta»**. Escribes el origen y el destino, eliges cómo te mueves (carro/moto, bus, bici o a pie) y la app dibuja el trayecto y lo colorea por tramos: **lluvia**, **seco** o **sin datos**.

![Ruta de Portal 80 a Calle 72 con Carrera 10](docs/img/09-ruta.png)

El panel resume distancia, duración y en qué kilómetros llueve, con una barra de salida a llegada. Detalles importantes:

- Muestra la lluvia **de ahora**, no la que habrá cuando llegues a cada punto.
- Un tramo queda **«sin datos»** si la estación más cercana está a más de 4 km o su última lectura es vieja. No se asume que está seco.

### Ver el historial de lluvia

El botón **«Historial»** de la barra abre una ventana con cuánto llovió y cuándo.

![Historial de lluvia de las últimas 24 horas](docs/img/08-historial.png)

- **Periodos:** 6 horas, 24 horas (barras por hora), 7 días y 30 días (barras por día).
- **Dónde:** toda Bogotá, una localidad o «cerca de tu destino» (con las 3 estaciones más próximas a elegir).
- **Indicadores:** total llovido, hora o día más lluvioso y porcentaje de datos disponibles.
- **Ranking** de las localidades más lluviosas del periodo; al pulsar una, el gráfico pasa a esa localidad.
- Los intervalos en los que la app estaba apagada aparecen como **«sin datos»**: no se inventa lluvia ni se supone que no llovió.

### Radar meteorológico (opcional)

El botón **«Radar»** superpone la imagen de reflectividad del radar del IDIGER, la misma capa de la página del SAB.

![Radar encendido sobre el mapa](docs/img/10-radar-encendido.png)

Hay que entenderlo bien, porque **el radar mide lluvia en el aire, no en el suelo**. Puede mostrar gotas que se evaporan antes de llegar o no mostrar una llovizna muy fina. Por eso:

- Está **apagado por defecto** (la app recuerda lo que elijas).
- Cuando lo enciendes, **solo dibuja los ecos de lluvia moderada o más fuerte** (amarillo, naranja, rojo). La llovizna y los ecos débiles (verde, cian, azul), y el blanco o los tonos pálidos, se descartan.
- En el tema claro los colores se oscurecen un poco (dorado, naranja vivo, carmesí) y llevan un borde oscuro para que no se pierdan sobre el fondo.
- La leyenda dice «Radar · lluvia en el aire» con la hora de la imagen.
- Sirve para ver lluvia **entre** estaciones y **fuera** de la ciudad, y de dónde viene. Para decir «llueve aquí» mandan siempre los pluviómetros.

### Cinco diseños, tema claro y oscuro, celular y computador

El botón **«Diseño»** cambia entre cinco estilos visuales y la app recuerda el último. El botón de la luna/sol cambia entre tema claro y oscuro. Funciona igual en computador y en celular: en el celular el panel se vuelve una hoja que sube desde abajo.

![Cumulonimbus en el celular](docs/img/12-celular.png)

Los cinco diseños están descritos en la [sección 10](#10-los-cinco-diseños).

---

## 3. Cómo funciona por dentro

### Arquitectura

```mermaid
flowchart LR
    subgraph Fuentes["Fuentes externas (gratuitas)"]
        SAB["SAB / IDIGER<br/>pluviómetros"]
        KMZ["Radar IDIGER<br/>(KMZ)"]
        CAT["Catastro Bogotá<br/>(direcciones, barrios)"]
        NOM["Nominatim<br/>(OpenStreetMap)"]
        VAL["Valhalla<br/>(rutas)"]
    end

    subgraph Backend["Backend · FastAPI (Python)"]
        API["API /api/*"]
        SRV["Servicios<br/>lluvia, geocodificación,<br/>historial, ruta, radar"]
        REC["Recolector<br/>cada 5 min"]
        CACHE["Caché en memoria"]
    end

    DB[("PostgreSQL<br/>estaciones · lecturas<br/>geocache · consultas")]
    FRONT["Frontend · React + MapLibre<br/>(servido por el mismo backend)"]

    SAB --> SRV
    KMZ --> SRV
    CAT --> SRV
    NOM --> SRV
    VAL --> SRV
    SRV <--> CACHE
    REC --> SAB
    REC --> DB
    SRV <--> DB
    API --> SRV
    FRONT <-->|JSON| API
```

### Qué pasa cuando abres la app

1. El navegador carga el frontend (React) y pide al backend `/api/resumen`, `/api/estaciones`, `/api/localidades` y `/api/radar`.
2. El backend consulta al SAB (`POST https://app.sab.gov.co/sab/ServletTipoSensores`, sensores de tipo 5 = pluviómetro). Guarda la respuesta **90 segundos** en memoria para no saturar al SAB si varias personas consultan a la vez.
3. Cada vez que trae datos frescos, **guarda una foto** (snapshot) de las lecturas en PostgreSQL. Así el histórico se llena también con las consultas de los usuarios.
4. Clasifica cada estación (intensidad, acumulado, si la lectura es vieja) y arma el resumen por localidad.
5. El frontend dibuja el mapa. No hay mapa base de calles: solo el contorno de las localidades y las estaciones.

### Recolector del histórico

El backend arranca con un **recolector** (APScheduler) que cada **5 minutos** consulta el SAB y guarda las lecturas en PostgreSQL, aunque nadie tenga la app abierta. La tabla `lecturas` tiene una restricción única por `(id_sensor, fecha_lectura)`, así que una misma lectura nunca se guarda dos veces. Se controla con `RECOLECTOR_ACTIVO` y `RECOLECTOR_INTERVALO_MINUTOS`.

> El histórico solo se llena **mientras el servidor está encendido**. Si lo apagas, esas horas quedan como «sin datos».

### Tecnologías

| Parte | Tecnología |
|---|---|
| Backend | Python 3.12, FastAPI, SQLAlchemy 2, psycopg 3, Alembic (migraciones), pydantic-settings, APScheduler, httpx |
| Base de datos | PostgreSQL (probado con la 18) |
| Frontend | React 19, Vite, TypeScript, MapLibre GL 5 |
| Pruebas | pytest |

Todos los servicios externos que usa son **gratuitos y no necesitan clave**.

---

## 4. Instalación paso a paso

Pensado para Windows, que es donde se desarrolló. Al final hay los equivalentes para Linux y macOS.

### 4.1 Requisitos

| Programa | Versión | Para qué | Cómo comprobar |
|---|---|---|---|
| Python | 3.12 o superior | Backend | `python --version` |
| Node.js | 20 o superior (con npm) | Compilar el frontend | `node --version` |
| PostgreSQL | probado con la 18 | Base de datos | `psql --version` |

### 4.2 Descargar el proyecto

```bash
git clone https://github.com/SGil14/Cumulonimbus.git
cd Cumulonimbus
```

### 4.3 Crear la base de datos (una sola vez)

Abre `psql` (o pgAdmin) como el usuario administrador de PostgreSQL y ejecuta, cambiando la contraseña:

```sql
CREATE ROLE lluvia_app LOGIN PASSWORD 'pon-aqui-una-contraseña-segura';
CREATE DATABASE lluvia_bogota OWNER lluvia_app;
```

No hay que crear las tablas a mano: las crean las migraciones en el paso 4.7.

### 4.4 Configurar el `.env`

En la carpeta raíz del proyecto copia el archivo de ejemplo y edítalo:

```bash
copy .env.example .env        # en Linux/macOS: cp .env.example .env
```

Lo único obligatorio es `DATABASE_URL`, con la contraseña del paso anterior:

```ini
DATABASE_URL=postgresql+psycopg://lluvia_app:pon-aqui-una-contraseña-segura@localhost:5432/lluvia_bogota
```

> Si tu contraseña tiene caracteres especiales (`@`, `:`, `/`, `#`), escríbelos codificados en la URL (por ejemplo `@` → `%40`). El archivo `.env` está en `.gitignore`: **nunca lo subas al repositorio**.

El resto de variables tiene valores por defecto razonables; están explicadas en la [sección 6](#6-configuración-env).

### 4.5 Instalar el backend

```bash
cd backend
python -m venv .venv
.venv\Scripts\pip install -r requirements.txt
```

### 4.6 Compilar el frontend

```bash
cd ..\frontend
npm install
npm run build
```

Esto genera `frontend/dist`, que el backend sirve solo. (Si usas el acceso directo del paso 4.7, este paso lo hace por ti la primera vez.)

### 4.7 Arrancar

**Forma fácil (Windows):** doble clic en **`Abrir Cumulonimbus.bat`**. Compila el frontend si falta, abre el navegador en http://localhost:8000 y levanta el backend (aplicando antes las migraciones de la base de datos).

**Forma manual:**

```bash
cd backend
iniciar.bat
```

`iniciar.bat` ejecuta `alembic upgrade head` (crea o actualiza las tablas) y luego `uvicorn app.main:app --reload --port 8000`.

### 4.8 Comprobar que todo funciona

| Qué abrir | Qué deberías ver |
|---|---|
| http://localhost:8000 | La app con el mapa de Bogotá. |
| http://localhost:8000/api/health | `{"estado":"ok","base_datos":"ok"}`: la app y la base están bien. |
| http://localhost:8000/docs | La documentación interactiva de la API. |

![Documentación interactiva de la API](docs/img/13-api-docs.png)

### 4.9 Modo desarrollo del frontend (opcional)

Si vas a modificar el diseño, usa el servidor de desarrollo, que recarga en vivo:

```bash
cd frontend
npm run dev                  # http://localhost:5173
```

Reenvía las llamadas `/api` al backend en el puerto 8000. Si tu backend corre en otro puerto, define antes la variable `API_URL` (por ejemplo `set API_URL=http://localhost:8001`).

### 4.10 Linux y macOS

`iniciar.bat` y el acceso directo son solo para Windows. Los equivalentes:

```bash
cd backend
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/alembic upgrade head
.venv/bin/python -m uvicorn app.main:app --port 8000
```

(Y `npm install && npm run build` en `frontend/`.) Estos pasos no se han probado en Linux ni en macOS.

---

## 5. Cómo usar la app

### La pantalla

| Zona | Qué hay |
|---|---|
| **Barra superior** | Nombre; botones **Radar**, **Historial**, **Diseño**, **Actualizar** y el interruptor de tema claro/oscuro. |
| **Panel** | Resumen del momento, buscador de destinos con ejemplos, formulario de ruta y lista de localidades con lo llovido hoy. |
| **Mapa** | Localidades coloreadas por lo acumulado hoy, estaciones, zonas de lluvia, ruta y radar. |

### Guía rápida

1. **Mira el resumen.** El número grande es cuántas de las 71 estaciones registran lluvia ahora.
2. **Busca tu destino.** Escribe en «¿A dónde vas?» y pulsa Enter. También puedes tocar uno de los ejemplos.
3. **Lee la respuesta.** «No llueve cerca» o «Sí, llueve cerca», más las estaciones próximas.
4. **Pulsa «Ver cuándo ha llovido aquí»** para ver qué pasó en las últimas horas en ese punto.
5. **Si vas a moverte,** abre «Revisa la ruta», escribe origen y destino, elige el modo y pulsa «Ver lluvia en el trayecto».
6. **Pulsa «Actualizar»** para traer datos frescos (la app no se refresca sola).
7. **Enciende el «Radar»** si quieres ver de dónde viene la lluvia (recuerda: es lluvia en el aire).
8. **Cambia el «Diseño»** y el tema al que más te guste.

### Colores del mapa

- **Relleno de cada localidad:** cuánto ha llovido hoy, con la escala del SAB: hasta 10 mm (bajo), hasta 30 (moderado), hasta 50 (alto), más de 50 (muy alto).
- **Punto de cada estación:** seco, o el color de su intensidad actual si llueve (tabla de la [sección 2](#ver-dónde-llueve-ahora)).

---

## 6. Configuración (`.env`)

Se lee del archivo `.env` de la carpeta raíz o de variables de entorno (las variables de entorno tienen prioridad, que es lo que se usa en la nube).

| Variable | Por defecto | Qué hace |
|---|---|---|
| `DATABASE_URL` | **obligatoria** | Conexión a PostgreSQL con el driver `psycopg` (`postgresql+psycopg://usuario:clave@host:5432/base`). |
| `CACHE_ESTACIONES_SEGUNDOS` | `90` | Segundos que se reutilizan los datos del SAB antes de volver a consultarlo. |
| `MINUTOS_LECTURA_VIEJA` | `30` | Una lectura más vieja que esto se marca como desactualizada y no cuenta como lluvia. |
| `ESTACIONES_CERCANAS` | `3` | Cuántas estaciones cercanas se muestran al consultar un destino. |
| `RECOLECTOR_ACTIVO` | `true` | Enciende o apaga el recolector del histórico. |
| `RECOLECTOR_INTERVALO_MINUTOS` | `5` | Cada cuántos minutos el recolector guarda lecturas. |
| `CORS_ORIGINS` | `["http://localhost:5173"]` | Orígenes permitidos para el frontend en desarrollo (formato JSON). No hace falta en producción, porque el backend sirve el frontend. |
| `LOG_LEVEL` | `INFO` | Nivel de los registros. |
| `IDECA_API_KEY` | vacío | Opcional. Activa el geocodificador oficial de IDECA (ver [geocodificación](#geocodificación-de-destinos)). |
| `RUTA_URL` | `https://valhalla1.openstreetmap.de/route` | Servicio de rutas. Cámbialo solo si montas tu propia instancia de Valhalla. |
| `SAB_URL`, `IDECA_URL`, `NOMINATIM_USER_AGENT` | ver `backend/app/config.py` | Direcciones y cabeceras de los servicios externos. |

---

## 7. La API

El backend expone una API JSON; el frontend es solo uno de sus clientes. Toda la documentación interactiva, con la posibilidad de probar cada endpoint, está en **`/docs`**.

| Endpoint | Descripción |
|---|---|
| `GET /api/health` | Estado de la app y de la base de datos. |
| `GET /api/estaciones` | Las 71 estaciones con su estado actual. |
| `GET /api/resumen` | Dónde llueve ahora y acumulado por localidad. |
| `GET /api/cercanas?lat=&lon=` | Estaciones más cercanas a un punto. |
| `GET /api/destino?q=` | Resuelve un destino (localidad, coordenadas, barrio, lugar, dirección o esquina) y devuelve ubicación, barrio y lluvia cercana. |
| `GET /api/ruta?origen=&destino=&modo=` | Trayecto de un lugar a otro (`auto`, `bus`, `bicicleta`, `a_pie`) y en qué tramos llueve ahora. |
| `GET /api/historial/serie?periodo=&localidad=` | Cuánto llovió en cada intervalo (por hora en `6h` y `24h`, por día en `7d` y `30d`). |
| `GET /api/historial/ranking?periodo=` | Localidades ordenadas por lo que llovió en el periodo. |
| `GET /api/historial/destino?lat=&lon=&periodo=` | Cuándo llovió cerca de un punto (las 3 estaciones más cercanas). |
| `GET /api/radar` | Imagen del radar (PNG transparente en base64) y la zona que cubre. |
| `GET /api/localidades` | Contorno de las localidades (GeoJSON), usado para dibujar el mapa. |

Con `?actualizar=true`, `/api/estaciones` y `/api/resumen` ignoran la caché y consultan al SAB en ese instante.

**Ejemplos:**

```bash
# ¿Dónde llueve ahora?
curl http://localhost:8000/api/resumen

# ¿Llueve cerca de una esquina?
curl "http://localhost:8000/api/destino?q=Calle%2072%20con%20Carrera%2010"

# Historial de las últimas 24 horas en Chapinero
curl "http://localhost:8000/api/historial/serie?periodo=24h&localidad=Chapinero"

# Ruta en bicicleta
curl "http://localhost:8000/api/ruta?origen=Portal%2080&destino=Chapinero&modo=bicicleta"
```

**Códigos de error útiles:** `502` si el SAB no responde; `503` si un servicio de geocodificación o de rutas falla por red (conviene reintentar: la app **no** responde «no existe» cuando en realidad fue un fallo de red).

---

## 8. Base de datos

Cuatro tablas, creadas por las migraciones de Alembic (`backend/alembic/versions/`).

| Tabla | Para qué sirve | Campos principales |
|---|---|---|
| `estaciones` | Catálogo de pluviómetros. | `id_sensor` (clave), `nombre`, `localidad`, `latitud`, `longitud`, `tecnologia` |
| `lecturas` | **Histórico**: una fila por estación y hora de lectura. Es la tabla para análisis. | `id_sensor`, `fecha_lectura`, `valor_5min`, `acumulado_dia`; única por (`id_sensor`, `fecha_lectura`) |
| `geocache` | Destinos ya resueltos, para no volver a preguntar a internet. | `consulta_normalizada`, `latitud`, `longitud`, `barrio`, `localidad`, `proveedor`, `aproximada`, `usos` |
| `consultas` | Registro de cada búsqueda de destino. | `texto`, `tipo`, `encontrado`, `barrio`, `localidad`, `proveedor`, `lloviendo_cerca` |

Ejemplo de análisis directamente en SQL: las horas más lluviosas del último mes.

```sql
SELECT date_trunc('hour', fecha_lectura) AS hora, SUM(valor_5min) AS mm
FROM lecturas
WHERE fecha_lectura > now() - interval '30 days'
GROUP BY 1 ORDER BY mm DESC LIMIT 10;
```

---

## 9. Reglas y cálculos

### Intensidad actual

El SAB entrega los milímetros caídos en los últimos 5 minutos. La app los multiplica por 12 para tener **mm por hora** y clasifica:

| mm/h | Intensidad |
|---|---|
| 0 | Sin lluvia |
| menos de 2,5 | Ligera |
| de 2,5 a menos de 7,6 | Moderada |
| de 7,6 a menos de 50 | Fuerte |
| 50 o más | Muy fuerte |

### Acumulado del día (escala del SAB)

Bajo: hasta 10 mm · Moderado: hasta 30 · Alto: hasta 50 · Muy alto: más de 50.

### Lectura desactualizada

Si la última lectura de una estación tiene más de **30 minutos**, se marca como desactualizada y **no cuenta como lluvia activa** (una estación sin reportar no debe aparecer como «lloviendo» con un dato viejo).

### Cálculo del historial

- **Por hora:** diferencia entre lecturas del `acumulado_dia` (que el SAB reinicia a medianoche). No importa si las lecturas llegan cada 2 o cada 5 minutos.
- **Por día:** el máximo `acumulado_dia` de cada estación ese día.
- Para una localidad o para Bogotá se **promedian** las estaciones.
- Un intervalo sin lecturas es **«sin datos»**, distinto de «seco». La pantalla muestra el porcentaje de datos disponibles y desde cuándo hay registros.
- El total de cada **día** incluye la lluvia caída antes del primer registro; el total por **hora** solo cuenta desde que hay lecturas guardadas. Por eso, si el histórico empezó a media mañana, los totales de hora y de día pueden diferir.

### Geocodificación de destinos

`/api/destino?q=...` resuelve el texto en este orden:

1. **Localidad** («Chapinero»): usa las estaciones de esa localidad, sin salir a internet.
2. **Coordenadas** («4.6486, -74.0628»).
3. **Barrio, lugar, dirección o esquina**, con estos proveedores en orden:
   1. **Caché** en PostgreSQL (`geocache`).
   2. **IDECA**, solo si hay `IDECA_API_KEY` (opcional).
   3. **Catastro de Bogotá (placas domiciliarias)**: gratis y sin clave. Entiende la nomenclatura bogotana: `Carrera 7 # 55-25`, `Calle 63 Sur # 80-15`, `Carrera 90 bis # 130-44` y esquinas como `Calle 72 con Carrera 10` o `Av Boyacá con Calle 80` (reconoce Boyacá, Caracas, NQS, Autopista Norte, Séptima, Ciudad de Cali y El Dorado). Si el número exacto no existe, usa la placa más cercana de la misma cuadra y marca el resultado como **aproximado**.
   4. **Nominatim** (OpenStreetMap): lugares, parques, centros comerciales y barrios («Portal 80», «Galerías»).

El barrio y la localidad salen de las capas públicas del Catastro (sector catastral y localidad). Cada búsqueda queda en `consultas` y cada destino resuelto en `geocache`.

**IDECA (opcional):** el geocodificador oficial exige una clave gratuita que entrega IDECA (https://mapas.bogota.gov.co/api/). Usa los mismos datos de placas que el proveedor de Catastro, así que hoy no es necesaria; si la consigues, ponla en `IDECA_API_KEY` y se activa sola, sin tocar código.

### Cálculo de la ruta

1. Pide el trayecto a **Valhalla** (servicio gratuito de OpenStreetMap) y decodifica la polilínea.
2. Parte el camino en puntos cada **400 m**.
3. Para cada punto toma la **estación más cercana** y decide: **lluvia** (llueve en esa estación), **seco**, o **sin datos** (estación a más de 4 km o con lectura vieja).
4. Agrupa los puntos consecutivos en tramos y escribe el resumen.

### Radar

El backend descarga el KMZ público de reflectividad, extrae la imagen y la zona geográfica que cubre, y la entrega en `/api/radar` (con 2 minutos de caché). Si el radar falla, la app sigue funcionando sin él. El filtrado de ecos débiles se hace en el navegador, por color (`frontend/src/lib/radarFiltrado.ts`).

---

## 10. Los cinco diseños

La experiencia es la misma; cambia el estilo. Se elige con el botón **«Diseño»**.

| Diseño | Idea |
|---|---|
| **Clima** (por defecto) | App del tiempo amigable: cielo claro o noche azul, tarjetas redondeadas. |
| **Tablero** | Grafito y verde limón, tipografía condensada, bloques sólidos. |
| **Radar** | Consola meteorológica: panel lateral denso, datos en monoespaciada, anillos de alcance centrados en el radar. |
| **Minimalista** | Casi sin adornos: el mapa, una frase y una barra de búsqueda; las localidades se despliegan con «Ver localidades». |
| **Mapa vivo** | El mapa lo es todo: el cielo cambia según llueva o no y cae **lluvia animada** sobre las zonas con lluvia. Respeta la opción «reducir movimiento» del sistema. |

**Clima, tema oscuro**

![Diseño Clima, tema oscuro](docs/img/02-diseno-clima-oscuro.png)

**Tablero**

![Diseño Tablero](docs/img/03-diseno-tablero.png)

**Radar**

![Diseño Radar](docs/img/04-diseno-radar.png)

**Minimalista**

![Diseño Minimalista](docs/img/05-diseno-minimalista.png)

**Mapa vivo**

![Diseño Mapa vivo](docs/img/06-diseno-mapa-vivo.png)

### Cómo agregar un diseño nuevo

Cada diseño es un archivo CSS autocontenido en `frontend/src/disenos/` más su paleta del mapa. Para crear uno:

1. Crea `frontend/src/disenos/mi-diseno.css` (puedes partir de `clima.css`).
2. Regístralo en la lista `DISENOS` de `frontend/src/disenos/index.ts`, con su paleta del mapa para tema claro y oscuro.
3. Define las variables `--h-*` que usan las piezas compartidas (historial, gráfico y ruta, en `comun.css`).

---

## 11. Estructura del proyecto

```
Cumulonimbus/
├── backend/
│   ├── app/
│   │   ├── main.py                # Arranque de FastAPI; sirve también el frontend compilado
│   │   ├── config.py              # Configuración (.env)
│   │   ├── api/routes/            # Endpoints: lluvia, destino, mapa, radar, historial, ruta, health
│   │   ├── services/              # Lógica: cliente SAB, reglas de lluvia, geocodificación,
│   │   │                          #         historial, ruta, radar, barrios
│   │   ├── repositories/          # Lectura y escritura en PostgreSQL
│   │   ├── db/                    # Conexión y tablas
│   │   ├── jobs/recolector.py     # Guarda el histórico cada 5 minutos
│   │   ├── models/                # Modelos de datos (pydantic)
│   │   └── core/                  # Caché y zona horaria
│   ├── alembic/                   # Migraciones de la base de datos
│   ├── tests/                     # Pruebas automáticas
│   ├── iniciar.bat                # Migra y levanta la API (Windows)
│   └── requirements.txt
├── frontend/                      # React + Vite + TypeScript + MapLibre
│   └── src/
│       ├── components/            # Mapa, panel, buscador, historial, ruta, radar…
│       ├── hooks/                 # Datos, tema, diseño, historial
│       ├── lib/                   # API, formatos, geometría, lluvia animada, filtro del radar
│       └── disenos/               # Un CSS por diseño + paletas del mapa
├── docs/img/                      # Capturas de este README
├── Abrir Cumulonimbus.bat            # Un clic: compila el frontend si falta y abre la app
├── .env.example                   # Plantilla de configuración
└── README.md
```

El backend sigue capas: **rutas** (HTTP) → **servicios** (reglas de negocio, funciones puras donde se puede) → **repositorios** (base de datos). Así la lógica se prueba sin base de datos.

---

## 12. Pruebas

```bash
cd backend
.venv\Scripts\python -m pytest
```

Hay **56 pruebas** que cubren las reglas de intensidad y acumulado, la lectura de direcciones del Catastro, la geocodificación, el historial, la ruta y el radar. Usan datos de ejemplo guardados en `backend/tests/fixtures`.

---

## 13. Despliegue en la nube

El proyecto está pensado para desplegarse como **un solo servicio**: el backend de FastAPI sirve la API y también el frontend ya compilado, y se conecta a un PostgreSQL. No hay un Dockerfile todavía (ver [hoja de ruta](#17-hoja-de-ruta)) y **el despliegue en la nube aún no se ha probado**; estos son los pasos que deberían servir en cualquier proveedor que ejecute Python:

**1. Base de datos.** Crea un PostgreSQL gestionado (la mayoría de proveedores tiene plan gratuito) y copia su cadena de conexión. Conviértela al formato del driver: `postgresql+psycopg://usuario:clave@host:5432/base`.

**2. Variables de entorno** en el servicio (no subas el `.env`):

| Variable | Valor |
|---|---|
| `DATABASE_URL` | la cadena de conexión del paso anterior |
| `RECOLECTOR_ACTIVO` | `true` en **una sola** instancia (ver abajo) |

**3. Comando de construcción** (build):

```bash
pip install -r backend/requirements.txt && cd frontend && npm ci && npm run build
```

**4. Comando de arranque:**

```bash
cd backend && alembic upgrade head && python -m uvicorn app.main:app --host 0.0.0.0 --port $PORT
```

El puerto lo suele entregar el proveedor en la variable `PORT`. Si en tu plataforma el directorio de trabajo es distinto, ajusta las rutas.

**Cosas que debes tener en cuenta:**

- **Una sola instancia con el recolector.** Si escalas a varias réplicas, solo una debe tener `RECOLECTOR_ACTIVO=true`. Aun así no se duplican datos (la restricción única lo impide), pero se haría trabajo repetido.
- **Planes gratuitos que «duermen».** Algunos proveedores apagan el servicio tras un rato sin visitas. Mientras está apagado **no se guarda histórico**. Si el histórico te importa, necesitas un plan que no duerma o un servicio externo que lo despierte cada pocos minutos.
- **Servicios públicos con límites.** Nominatim y Valhalla públicos son para uso moderado (Nominatim pide máximo 1 consulta por segundo y un `User-Agent` identificable, que la app ya envía). Para mucho tráfico conviene alojar tu propia instancia (`RUTA_URL`) o usar un geocodificador con clave (`IDECA_API_KEY`).
- **El SAB debe ser accesible desde el servidor.** Si el proveedor bloquea salidas a internet o el SAB bloquea la IP del servidor, la app mostrará un error 502.

---

## 14. Límites que debes conocer

Para no engañarte con lo que muestra la app:

- **Un pluviómetro mide un punto.** Si llueve a 3 km de la estación más cercana, esa estación puede marcar 0. Una llovizna muy fina puede no registrarse: estos pluviómetros cuentan cuando se acumula una cantidad mínima de agua.
- **Las lecturas llegan cada ~2 minutos** y la app solo consulta cuando abres o actualizas; una lluvia que acaba de empezar puede tardar unos minutos en aparecer.
- **El radar ve lluvia en el aire.** Puede mostrar lluvia que no llega al suelo, y sus colores no están calibrados contra los pluviómetros: la equivalencia entre color e intensidad es orientativa.
- **Las zonas de color son una estimación visual** alrededor de cada estación; no hay medición entre estaciones.
- **La ruta usa la lluvia de ahora,** no un pronóstico de lo que pasará cuando llegues.
- **No es un pronóstico.** Cumulonimbus dice lo que está pasando y lo que pasó, no lo que va a pasar.
- **El histórico empieza cuando el servidor se enciende por primera vez.** No hay datos anteriores (importar datos históricos del IDEAM está en la hoja de ruta).

---

## 15. Solución de problemas

| Síntoma | Causa probable y solución |
|---|---|
| `Abrir Cumulonimbus.bat` abre una página que no carga | El servidor tarda unos segundos en arrancar: recarga la página. Si sigue, mira la ventana negra por errores. |
| Error de conexión a la base (`password authentication failed`, `connection refused`) | PostgreSQL apagado, o el `DATABASE_URL` del `.env` no coincide con tu usuario, contraseña, puerto o nombre de base. |
| `/api/health` dice que la base falla | Igual que el anterior; verifica que `psql` entra con esos mismos datos. |
| Veo una versión vieja de la app | Hay un servidor antiguo ocupando el puerto 8000. Cierra su ventana, abre `Abrir Cumulonimbus.bat` y recarga con **Ctrl+F5**. |
| El historial dice «Aún hay pocos datos» | Es normal al empezar: el histórico solo se llena mientras el servidor está encendido. |
| La app no registra la lluvia que estoy viendo | Mira [Límites que debes conocer](#14-límites-que-debes-conocer): la estación más cercana puede estar lejos o no haber registrado todavía una llovizna fina. Prueba con «Actualizar» y el radar. |
| Error 502 «El SAB no está respondiendo» | El servicio del SAB está caído o lento. Intenta de nuevo en unos minutos. |
| Error 503 al buscar una dirección o calcular una ruta | Falló un servicio externo (Catastro, Nominatim, Valhalla), por ejemplo por la red de tu empresa o colegio. Reintenta. |
| El radar no aparece | Está apagado por defecto: pulsa «Radar». Si no hay ecos moderados o fuertes, no se dibuja nada (es lo esperado). |
| `npm run build` falla | Comprueba que Node sea de la versión 20 o superior y borra `frontend/node_modules` para repetir `npm install`. |

---

## 16. Fuentes de datos y créditos

Todos son servicios públicos y gratuitos:

| Dato | Fuente |
|---|---|
| Pluviómetros | SAB / IDIGER: https://app.sab.gov.co/sab/lluvias.htm |
| Radar meteorológico | Radar del IDIGER (reflectividad, KMZ público) |
| Direcciones, barrios y localidades | Catastro Bogotá (placas domiciliarias, sector catastral, localidad) |
| Lugares y barrios | Nominatim, de OpenStreetMap (© colaboradores de OpenStreetMap) |
| Rutas | Valhalla, de OpenStreetMap |

Este proyecto no está afiliado al IDIGER, a la Alcaldía de Bogotá ni a OpenStreetMap. Los datos pertenecen a sus respectivas entidades.

Aún no se ha definido una licencia para el código.

---

## 17. Hoja de ruta

| Estado | Función |
|---|---|
| ✅ | Backend: cliente SAB, reglas, API, PostgreSQL y recolector |
| ✅ | Geocodificación de direcciones, esquinas, barrios y lugares |
| ✅ | Frontend con cinco diseños, tema claro/oscuro y versión para celular |
| ✅ | Historial por horas y por días |
| ✅ | Ruta de un lugar a otro |
| ✅ | Radar con filtro de ecos débiles y zonas de color por estación |
| ⏳ | Atajo para iPhone (una respuesta en texto plano de «¿llueve aquí?») |
| ⏳ | Avisos de lluvia (correo o Teams) |
| ⏳ | Descarga del histórico en CSV |
| ⏳ | Recolector cada 2 minutos (el SAB actualiza cada ~2) |
| ⏳ | Datos históricos del IDEAM |
| ⏳ | Docker y guía de despliegue en un proveedor concreto |
