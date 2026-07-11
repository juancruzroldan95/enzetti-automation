# Proyecto: Automatización de Notificaciones Judiciales - Estudio Jurídico

## Objetivo General
Optimizar y automatizar el flujo de trabajo administrativo del estudio jurídico mediante el procesamiento inteligente de notificaciones judiciales y su posterior inyección en el software de gestión de escritorio local Lex Doctor (software legacy). El sistema busca reducir de 3-4 horas diarias a minutos el tiempo dedicado a la carga manual de novedades.

---

## Contexto y Filosofía de Automatización (Las 3 Fases)

Este proyecto nace bajo la premisa de que **los modelos de lenguaje (LLMs) carecen de interactividad física directa con sistemas legacy aislados**. Claude o Gemini actúan como el "cerebro", pero la infraestructura de red y los scripts de RPA actúan como "las manos". 

Para garantizar el éxito y la estabilidad del sistema frente a entornos hostiles (sistemas judiciales con bloqueos y Captchas) y sistemas cerrados (Lex Doctor sin APIs), la automatización se concibe en tres fases incrementales:

### Fase 1: El Puente IA-RPA (Enfoque Actual / MVP)
*   **Dinámica:** Semi-automatizada en la entrada, 100% automatizada en el procesamiento y carga.
*   **Flujo:** El equipo del estudio descarga las notificaciones judiciales del día (PDF/HTML) de forma manual y las deposita en una carpeta compartida de OneDrive/Google Drive. A partir de ahí, **n8n + Claude** toman el control en la nube, procesan el texto denso, extraen las variables clave en un JSON, y el **Worker local de Python** inyecta los datos de forma autónoma en Lex Doctor mediante atajos de teclado.
*   **Motivación:** Elimina el 80% del tiempo de carga administrativa manual con un riesgo de fallo cercano a cero, aislando el desarrollo de los cambios repentinos en las webs judiciales.

### Fase 2: Automatización del Web Scraping (Evolución a Mediano Plazo)
*   **Dinámica:** 100% automatizada de extremo a extremo.
*   **Flujo:** Se integra un módulo de scraping web en la nube o local (utilizando herramientas como *Playwright* o *Puppeteer*) que ingresa automáticamente a los portales MEV y PJN utilizando las credenciales del abogado. El bot navega, descarga las notificaciones del día de forma directa y las empuja al flujo de n8n de la Fase 1.
*   **Desafío:** Requiere sortear Captchas avanzados (mediante servicios de resolución de captchas por API o visión artificial) y mantenimiento constante ante cambios en el DOM de las webs del Estado.

### Fase 3: Integración Nativa / Software Customizado (Visión de Futuro)
*   **Dinámica:** Modernización tecnológica completa del estudio.
*   **Flujo:** Reemplazo definitivo o migración del software legacy Lex Doctor hacia un sistema de gestión interno web customizado (Backend propio con base de datos moderna y API REST nativa). 
*   **Motivación:** En esta instancia, la capa local de Python (RPA) desaparece. n8n e IA se comunican directamente vía API con el nuevo sistema del estudio, logrando una arquitectura limpia, hiper veloz, escalable y mantenible en la nube.

---

## Alcance del Proyecto (Scope)

El proyecto se implementa bajo una arquitectura de **Monorepo** desacoplada, dividida en dos componentes principales:

### 1. Capa Nube / Orquestación (n8n)
*   **Tecnología:** n8n (Self-Hosted en Docker) desplegado en **Railway** con persistencia en **PostgreSQL**.
*   **Función:** Recepción y centralización de la data de las notificaciones judiciales crudas de los portales de Provincia de Buenos Aires (MEV) y CABA/Nación (PJN).
*   **Cerebro IA:** Integración mediante API con **Anthropic (Claude)** utilizando prompts estructurados con Few-Shot prompting para normalizar y estructurar el texto legal denso.
*   **Entregable:** Generación diaria de un payload unificado en formato **JSON** estructurado con los campos limpios necesarios para el estudio (`expediente`, `caratula`, `juzgado`, `resumen_ejecutivo`).

### 2. Capa Local / Ejecución (Python)
*   **Tecnología:** Python 3.x (`pyautogui`, `pywinauto`).
*   **Despliegue:** Ejecución desatendida local en la PC del estudio mediante el **Programador de Tareas de Windows (Task Scheduler)** corriendo en formato Cron (una vez al día, al cierre de tribunales).
*   **Función:** Descarga o accede de alguna manera al JSON generado por n8n, interactúa directamente con la interfaz Win32 nativa de **Lex Doctor** simulando atajos de teclado (`Ctrl+E`, `Tab`, `Enter`) e inyecta las novedades procesadas de forma segura y automatizada.

---

## Fuera de Alcance (Out of Scope)
*   **Scraping Directo de Portales (Fase 1):** Debido a las restricciones de Captchas y políticas de seguridad del PJN/MEV, la extracción inicial automatizada mediante scraping queda fuera del MVP. El flujo inicial asume que el usuario descarga los PDF/HTML del día en un repositorio compartido (OneDrive/Google Drive) que n8n vigila de manera nativa.
*   **Modificación directa de BD Lex Doctor:** No se realizarán conexiones a nivel base de datos (DBF/propietaria) del software para mitigar riesgos de corrupción de datos. Toda interacción se realiza estrictamente emulando la interfaz de usuario (RPA).
*   **Infraestructura Multi-Worker:** Al ser un volumen acotado para un estudio mediano, no se requiere arquitectura distribuida con Redis; se utiliza la infraestructura monolítica oficial de n8n.

---

## Estructura del Repositorio

```text
/
├── GEMINI.md           # Contexto del proyecto para IA (Este archivo).
├── README.md           # Guía de setup e infraestructura para humanos.
├── .gitignore          # Excluye entornos virtuales, n8n_data locales y JSONs temporales.
│
├── n8n/                # Capa Nube: Configuración de infraestructura y orquestación.
│   ├── .env            # Variables de entorno locales para n8n y PostgreSQL.
│   ├── docker-compose.yml # n8n + PostgreSQL (Template oficial de n8n-hosting).
│   └── init-data.sh    # Script oficial de inicialización segura de base de datos.
│
└── python/             # Capa Local: Código del inyector de escritorio Windows (RPA).
    ├── main.py         # Script principal en Python para automatización de UI.
    ├── requirements.txt # Dependencias de Python (pywinauto, pyautogui, requests).
    └── notificaciones.json # Payload JSON estructurado de prueba (Mock para desarrollo).
```

---

## Tech Stack

La solución está construida sobre un conjunto de tecnologías estándar de la industria, seleccionadas por su confiabilidad, facilidad de integración con modelos de lenguaje y bajo costo operativo.

### Orquestación e Infraestructura
*   **n8n (Self-Hosted):** Motor de automatización basado en nodos que actúa como backend centralizador. Se encarga de capturar eventos, manejar la lógica de control del flujo y coordinar las llamadas externas.
*   **PostgreSQL:** Motor de base de datos relacional que almacena de forma persistente el historial de ejecuciones, los metadatos de los flujos y las credenciales encriptadas del sistema.
*   **Docker y Docker Compose:** Entorno de contenedores para garantizar la paridad entre el entorno de desarrollo local y el entorno de producción.
*   **Railway:** Plataforma de alojamiento en la nube (PaaS) que ejecuta los contenedores con un esquema de facturación basado en consumo real de recursos (CPU y RAM fraccionados por segundo), eliminando costos fijos altos.

### Inteligencia Artificial y Procesamiento de Lenguaje
*   **Anthropic Claude API:** Modelo de lenguaje utilizado como el motor cognitivo para la normalización de texto. Su ventana de contexto y capacidad de comprensión de estructuras sintácticas complejas (como la prosa judicial argentina) se utilizan para transformar texto desestructurado en esquemas de datos rígidos.

### Automatización Local (RPA)
*   **Python 3.x:** Entorno de ejecución en la PC del cliente para correr el script del Worker local.
*   **PyWinauto:** Librería de Python para interactuar de forma nativa con el árbol de controles de la API de accesibilidad de Windows (Win32), permitiendo identificar elementos de la interfaz de Lex Doctor sin depender de su ubicación visual.
*   **PyAutoGUI:** Librería complementaria de Python utilizada para enviar instrucciones de teclado de bajo nivel y atajos del sistema operativo de manera confiable.

---

## Directivas para la IA (System Instructions Context)
Cuando trabajes en este repositorio, tené en cuenta:

1. **Seguridad:** Jamás expongas API Keys, credenciales de PostgreSQL ni datos reales de expedientes en el código. Utilizá variables de entorno (`.env`).

2. **Robustez en UI (Python):** Al diseñar interacciones en `local-worker`, priorizá el uso de atajos de teclado y el árbol de control de `pywinauto` sobre coordenadas fijas de pixeles (`pyautogui`), para evitar fallas por cambios de resolución de pantalla.

3. **Prompt Engineering Legal:** El prompt que se conecte a Claude en n8n debe ser ultra preciso con la jerga judicial argentina (distinguir autos, proveídos, traslados, etc.) y exigir siempre un formato de salida JSON válido y tipado.