# 🤖 AI Cinema & Music Agent: n8n, Groq (Llama 3.1), MinIO & MusicGen

> 🏆 **Academic Grade: 9+/10** | *Universidad Politécnica de Madrid (UPM) - 2024*
> 👥 **Team Project:** Co-authored with my colleagues from the Data Science & AI degree.

### 📝 About the Project
This project implements an intelligent system for the automation, extraction, analysis, and distribution of Madrid's cinema listings and musical events. 

The architecture orchestrates multiple local Docker microservices via **n8n**, delegating cognitive logic to **Llama 3.1 (via Groq)** and storing historical execution data in a local **MinIO (S3)** bucket. It also integrates **ComfyUI** for dynamic, real-time soundtrack generation using HuggingFace's **MusicGen** model.

The ecosystem is completely containerized and decoupled[cite: 1]:
- **Scrapper Microservice:** Built with FastAPI and Playwright for on-demand data extraction from IMDb and eCartelera[cite: 1].
- **Orchestration:** n8n acts as the central hub triggering workflows via Telegram webhooks and schedules[cite: 1].
- **Generative AI:** Groq (Llama 3.1 8b) is used for intent classification, text formatting, and spoiler-checking[cite: 1].
- **Audio Generation Pipeline:** ComfyUI generates audio locally via CPU optimized with FP32, using a shared volume directly bridged to n8n to instantly send the `.wav` files via Telegram[cite: 1].

### 👨‍💻 My Contribution & Role
Within this collaborative project, my primary focus was on the orchestration, automation, and user-interface layer. My specific hands-on contributions included:

- **n8n Workflow Design & Orchestration:** Designed and implemented the core n8n pipelines (Weekly Cinema, Interactive Chat, and Concerts Agent) to connect all microservices.
- **Telegram Bot Integration:** Developed the interactive chatbot logic, configuring the system to receive messages, route them through the AI nodes for intent classification, and return formatted responses or audio files directly to the user's chat.
- **AI & Cognitive Routing:** Integrated Groq (Llama 3.1) nodes within the n8n workflows to validate scraped data, ensure spoiler-free content, and dynamically format markdown messages with emojis.
- **Data Flow Management:** Configured the n8n nodes to communicate with the Python scraper API and handle the persistence of JSON files into the local MinIO (S3) bucket.

---
*Note: This is a forked repository to showcase my specific contributions to this academic project. The full original documentation is available in the Spanish `.md` files within this repository.*
