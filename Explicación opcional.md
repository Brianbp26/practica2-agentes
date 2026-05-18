# Agente Inteligente de Cartelera y Conciertos con n8n, Groq, S3 y ComfyUI (MusicGen-HF)

Este repositorio contiene una de **Sistemas Inteligentes** para la automatización, extracción, análisis, maquetación y distribución de la cartelera de cine de Madrid y eventos musicales/conciertos.

El sistema orquesta múltiples microservicios locales en Docker mediante **n8n**, delegando la lógica cognitiva a **Llama 3.1** (a través de **Groq**) y almacenando el histórico de ejecuciones en un bucket **S3 local (MinIO)**. Incorpora además **ComfyUI** para la generación dinámica de bandas sonoras personalizadas en tiempo real mediante **HuggingFace MusicGen**.

---

## Arquitectura Completa del Stack

El ecosistema está completamente contenedorizado en Docker, desacoplado y optimizado para ejecutarse localmente:

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                DOCKER COMPOSE STACK                                    │
│                                                                                        │
│  ┌──────────────┐     HTTP (REST API)     ┌────────────────┐                           │
│  │   scrapper   │◄────────────────────────│      n8n       │ (Orquestador Principal)   │
│  │  (FastAPI +  │                         │                │                           │
│  │  Playwright) │                         │  - Workflows   │                           │
│  └──────────────┘                         │  - Triggers    │                           │
│                                           │  - Lógica      │                           │
│  ┌──────────────┐     S3 API (Upload)     │  - user: root  │                           │
│  │    MinIO     │◄────────────────────────│                │                           │
│  │ (S3 Storage) │                         │                │                           │
│  └──────────────┘                         │                │                           │
│                                           │                │                           │
│  ┌──────────────┐     HTTP (API Prompt)   │                │                           │
│  │   comfyui    │◄────────────────────────│                │                           │
│  │ (MusicGen)   │                         │                │                           │
│  └──────────────┘                         └────────────────┘                           │
│         │                                          ▲    ▲    │                         │
│         │                                          │    │    │ HTTP (Enviar Telegram)  │
│         │ Escritura Directa en Volumen Compartido  │    │    ▼                         │
│         └───────────────► [ ./data/comfyui/output ] │  ┌──────────────────┐            │
│                           (Montado en n8n como     │  │   Telegram Bot   │            │
│                            /root/.n8n-files) ──────┘  │     (Chat /      │            │
│                                                       │  Notificaciones) │            │
│  ┌──────────────┐             Telegram Webhook        │                  │            │
│  │    poller    │─────────────────────────────────────┴──────────────────┘            │
│  │   (python)   │◄────────────────────────────────────┘                               │
│  └──────────────┘                Long Polling (getUpdates)                             │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

### Componentes y Servicios

| Componente | Imagen / Tecnología | Puerto | Descripción |
| :--- | :--- | :--- | :--- |
| **n8n** | `n8nio/n8n:latest` | `5678` | Orquestador de flujos de trabajo. Ejecutado como `root` para interactuar con el sistema de archivos del host. |
| **Scrapper** | `FastAPI + Playwright` | `8000` | Microservicio de raspado bajo demanda para IMDb y eCartelera. |
| **MinIO** | `minio/minio:latest` | `9000` / `9001` | Almacenamiento S3 local. Actúa como base de datos persistente e histórico semanal de la cartelera. |
| **ComfyUI** | `yanwk/comfyui-boot:cpu` | `8188` | Interfaz estable de IA configurada en modo CPU de bajo consumo y optimizada con FP32 (`--force-fp32`) para MusicGen. |
| **Telegram Poller** | `python:3.10-alpine` | *Interno* | Escucha mensajes mediante Long Polling de Telegram y los reenvía al webhook de n8n. |

---

## ⚡ Workflows n8n Implementados

Los workflows se encuentran en la carpeta `n8n_workflows/` listos para ser importados en tu instancia local de n8n:

### 1.[Cartelera Semanal]
*   **Trigger:** Cada lunes a las 9:00 AM (o manual).
*   **Funcionamiento:** 
    1. Llama al microservicio `scrapper` para extraer las películas de Madrid filtradas por el perfil del usuario.
    2. Valida la cartelera con **Groq (Llama 3.1 8b)** para verificar que no haya spoilers y corregir la gramática.
    3. Envía el mensaje segmentado automáticamente a Telegram.
    4. **Persistencia S3:** Guarda en MinIO `cartelera_latest.json` (consumo inmediato) y `cartelera_YYYY_MM_DD.json` (histórico semanal).

### 2.[Chat Interactivo para la Cartelera]
*   **Trigger:** Al recibir cualquier comando o mensaje por el bot de Telegram.
*   **Funcionamiento con Agente de Generación de Banda Sonora:**
    1. Descarga en caliente la cartelera desde MinIO S3.
    2. **Clasificador de Intención:** Un nodo de **Groq** determina si la intención es de `"chat"` (consulta de cartelera) o de `"soundtrack"` (petición musical).
    3. **Rama Chat:** Responde amigablemente como un acomodador de cine.
    4. **Rama Soundtrack (ComfyUI Real-Time Pipeline):**
        *   Groq crea un prompt de estilo musical según el género de la película elegida.
        *   Llama al endpoint `/prompt` de ComfyUI ejecutando el workflow de **HuggingFace MusicGen**.
        *   **Wait & Execute Command:** El flujo de n8n hace una pausa de 25 segundos y ejecuta un comando Shell (`ls -t /root/.n8n-files/audio/*.wav | head -n 1`) para encontrar el audio generado.
        *   **Read file from disk:** Lee el archivo `.wav` directamente del volumen de salida compartido y lo envía de forma multipart al chat del usuario a través del método `sendAudio` de Telegram.

### 3.[Agente Conciertos]
*   **Trigger:** Cada lunes a las 10:00 AM.
*   **Funcionamiento:**
    1. Obtiene los eventos musicales de la API oficial de datos abiertos del Ayuntamiento de Madrid.
    2. Filtra por eventos musicales del día en adelante (los 10 más próximos).
    3. Groq maqueta la lista con formato Markdown enriquecido con emojis.
    4. Envía el resultado al canal de conciertos y por correo electrónico (SMTP).

---

## Optimización y Compartición de Volúmenes (ComfyUI ➔ n8n)

> [!IMPORTANT]
> **El Puente de Volumen Directo:** Para que la banda sonora se envíe de manera instantánea y automática a Telegram sin descargas por red ni latencias, hemos configurado un volumen compartido directo en Docker:
> 
> ```yaml
> # En n8n:
> volumes:
>   - ./data/comfyui/output:/root/.n8n-files
> ```
> El contenedor de ComfyUI escribe el audio generado en su output local, el cual se refleja al instante en el volumen de n8n. Gracias a que el contenedor de n8n corre bajo `user: "root"`, el flujo de n8n puede ejecutar scripts shell para ubicar el último archivo de audio con un comando simple y leerlo directamente del disco con el nodo `Read/Write Files from Disk`.

> [!TIP]
> **Inferencia Precisa en CPU (MusicGen-HF):** Para la generación de audio, migramos el flujo a **HuggingFace MusicGen (modelo small)**. Esto permite generar pistas musicales muy agradables y personalizadas a partir de prompts de texto en cuestión de segundos, incluso corriendo en CPU, con un consumo de recursos mínimo e ideal para ordenadores con 8GB de RAM. La bandera `--force-fp32` en el contenedor `comfyui` asegura que las matemáticas del modelo sean limpias en CPU y evita cuelgues o corrupciones de audio.

---

## Guía de Despliegue

### Requisitos Previos
*   Docker y Docker Compose.
*   Un bot de Telegram creado mediante [@BotFather](https://t.me/BotFather) y su respectiva `Token`.
*   El `Chat ID` de tu usuario/canal.

### Paso 1: Configurar Variables de Entorno
Copia la plantilla y edita el fichero `.env` con tus tokens y tu API Key de Groq:
```bash
cp .env.example .env
```
Fichero `.env` configurado:
```ini
BOT_TOKEN=tu_token_de_telegram_cartelera
CHAT_ID=tu_chat_id_cartelera

BOT_CONCIERTOS_TOKEN=tu_token_de_telegram_conciertos
CHAT_CONCIERTOS_ID=tu_chat_id_conciertos

PERFIL={"Acción": 6, "Animación": 5, "Ciencia ficción": 7}
GROQ_API_KEY=gsk_tu_clave_de_groq_aqui
```

### Paso 2: Iniciar el Stack de Docker
Construye y arranca todos los contenedores en segundo plano:
```bash
docker compose up -d --build
```

### Paso 3: Configurar el Bucket en MinIO
1. Abre tu navegador e ingresa a la consola de MinIO: [http://localhost:9001](http://localhost:9001).
2. Credenciales por defecto: Usuario `admin` \| Contraseña `supersecret`.
3. Dirígete a **Buckets** ➔ **Create Bucket** y crea un bucket llamado `cartelera`.

### Paso 4: Importar y Configurar Workflows en n8n
1. Accede a n8n: [http://localhost:5678](http://localhost:5678).
2. Crea un nuevo flujo e importa cada uno de los archivos JSON de la carpeta `n8n_workflows/`.
3. **Configurar Credenciales S3:** En los nodos de S3 (`Upload a file`, `Download a file`), crea o asigna una nueva credencial de tipo **S3 Account**:
   * **Access Key:** `admin`
   * **Secret Key:** `supersecret`
   * **Endpoint:** `http://minio:9000` (¡importante usar `minio` en lugar de `localhost` ya que corre dentro de Docker!)
4. **Configurar Credenciales SMTP:** En el nodo `Send Email` del Agente Conciertos, configura tu SMTP de preferencia (por ejemplo, Gmail con contraseña de aplicación).
5. Activa los tres flujos (*Active* toggle arriba a la derecha).
