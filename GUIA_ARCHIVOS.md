# 📁 Guía Rápida de Archivos del Proyecto

Esta guía describe de forma directa y concisa la función de cada archivo en el repositorio para facilitar la corrección y entendimiento del proyecto.

---

## 🐍 Código de la Aplicación (Python)

*   **[cartelera.py](file:///media/brian/ssd_extra/CDIA%203%C2%BA/2%C2%BA%20Cuatrimestre/Sistemas%20Inteligentes/practica2-agentes/cartelera.py):** Script principal que raspa la cartelera madrileña de eCartelera y filtra las películas en base al perfil y nota mínima del usuario.
*   **[scrapper.py](file:///media/brian/ssd_extra/CDIA%203%C2%BA/2%C2%BA%20Cuatrimestre/Sistemas%20Inteligentes/practica2-agentes/scrapper.py):** Motor de extracción detallada que navega e interactúa con IMDb usando Playwright para conseguir sinopsis, notas, votos, director y duración.
*   **[lambda_function.py](file:///media/brian/ssd_extra/CDIA%203%C2%BA/2%C2%BA%20Cuatrimestre/Sistemas%20Inteligentes/practica2-agentes/lambda_function.py):** Controlador e integrador diseñado para ejecutar el raspador dentro del entorno Serverless de AWS Lambda (invocado por la Alexa Skill).
*   **[server.py](file:///media/brian/ssd_extra/CDIA%203%C2%BA/2%C2%BA%20Cuatrimestre/Sistemas%20Inteligentes/practica2-agentes/server.py):** API REST local desarrollada con FastAPI que expone el scraper a través de puertos HTTP para permitir que n8n invoque el raspado bajo demanda.
*   **[poller.py](file:///media/brian/ssd_extra/CDIA%203%C2%BA/2%C2%BA%20Cuatrimestre/Sistemas%20Inteligentes/practica2-agentes/poller.py):** Script ligero en Python que intercepta mensajes de Telegram mediante Long Polling y los inyecta de forma segura a n8n sin requerir túneles públicos.

---

## 🐳 Contenedores e Infraestructura

*   **[docker-compose.yml](file:///media/brian/ssd_extra/CDIA%203%C2%BA/2%C2%BA%20Cuatrimestre/Sistemas%20Inteligentes/practica2-agentes/docker-compose.yml):** Orquestador de Docker que monta todo el stack opcional: n8n, scrapper (FastAPI), MinIO S3 local, Telegram poller y el servidor ComfyUI.
*   **[Dockerfile](file:///media/brian/ssd_extra/CDIA%203%C2%BA/2%C2%BA%20Cuatrimestre/Sistemas%20Inteligentes/practica2-agentes/Dockerfile):** Configuración de empaquetado optimizada para subir y ejecutar el scraper de películas como una función en AWS Lambda.
*   **[Dockerfile.scrapper](file:///media/brian/ssd_extra/CDIA%203%C2%BA/2%C2%BA%20Cuatrimestre/Sistemas%20Inteligentes/practica2-agentes/Dockerfile.scrapper):** Configuración de empaquetado local para levantar el microservicio de scraping FastAPI y Uvicorn en tu stack Docker.
*   **[requirements.txt](file:///media/brian/ssd_extra/CDIA%203%C2%BA/2%C2%BA%20Cuatrimestre/Sistemas%20Inteligentes/practica2-agentes/requirements.txt):** Lista de dependencias y librerías de Python necesarias para ejecutar la lógica de raspado y API.
*   **[.env.example](file:///media/brian/ssd_extra/CDIA%203%C2%BA/2%C2%BA%20Cuatrimestre/Sistemas%20Inteligentes/practica2-agentes/.env.example):** Plantilla limpia de variables de entorno utilizada para configurar las claves de API, tokens de Telegram y perfiles de usuario.

---

## ⚡ Workflows y Flujos Visuales

### Carpetas:
*   **[n8n_workflows/](file:///media/brian/ssd_extra/CDIA%203%C2%BA/2%C2%BA%20Cuatrimestre/Sistemas%20Inteligentes/practica2-agentes/n8n_workflows/):** Contiene los flujos JSON listos para importar directamente en n8n:
    1.  `Cartelera Semanal.json`: Extrae la cartelera, la valida con Groq para eliminar spoilers y la guarda en MinIO S3.
    2.  `Chat interactivo para la cartelera.json`: Bot de Telegram que chatea como acomodador de cine y enruta de forma cognitiva hacia el sintetizador ComfyUI.
    3.  `Agente Conciertos.json`: Recopila conciertos de la API de Datos Abiertos de Madrid, los maqueta con Groq y los difunde por Telegram y Gmail.
*   **[comfyui_workflows/](file:///media/brian/ssd_extra/CDIA%203%C2%BA/2%C2%BA%20Cuatrimestre/Sistemas%20Inteligentes/practica2-agentes/comfyui_workflows/):** Almacena el flujo visual JSON **`comfyUi_MusicGen-HF.json`**, el cual se puede arrastrar y soltar en ComfyUI para generar melodías a partir de texto con HuggingFace MusicGen.
