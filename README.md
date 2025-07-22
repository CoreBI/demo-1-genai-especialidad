# Corebi Dialogflow Webhook for BigQuery + Vertex AI Gemini

> **Resumen:** Función HTTP (Cloud Functions / Cloud Run) que actúa como webhook de Dialogflow CX o ES para responder preguntas en lenguaje natural sobre datos en BigQuery y/o resultados de una búsqueda en Vertex AI Search (Data Store). Usa modelos Gemini de Vertex AI para: (1) generar SQL a partir de texto libre, (2) ejecutar la consulta en BigQuery, (3) sintetizar la respuesta en lenguaje natural, o (4) responder a partir de resultados de búsqueda en un Data Store.

---

## Tabla de Contenidos
- [Arquitectura en Alto Nivel](#arquitectura-en-alto-nivel)
- [Características Clave](#características-clave)
- [Requisitos Previos](#requisitos-previos)
- [Variables de Configuración](#variables-de-configuración)
- [Instalación Local](#instalación-local)
- [Despliegue en Google Cloud](#despliegue-en-google-cloud)
- [Integración con Dialogflow](#integración-con-dialogflow)
- [Flujos Lógicos de Webhooks](#flujos-lógicos-de-webhooks)  
  - [Flujo BigQuery (`bq_webhook`)](#flujo-bigquery-bq_webhook)  
  - [Flujo Data Store / Vertex AI Search (`ds_webhook`)](#flujo-data-store--vertex-ai-search-ds_webhook)
- [Límites de Tokens y Manejo de Sesión de Chat](#límites-de-tokens-y-manejo-de-sesión-de-chat)
- [Prompts del Sistema](#prompts-del-sistema)
- [Logging y Observabilidad](#logging-y-observabilidad)
- [Buenas Prácticas de Seguridad y Costos](#buenas-prácticas-de-seguridad-y-costos)
- [Pruebas Locales](#pruebas-locales)
- [Solución de Problemas](#solución-de-problemas)
- [Roadmap / Próximos Pasos](#roadmap--próximos-pasos)
- [Licencia y Atribuciones](#licencia-y-atribuciones)
- [Avisos de Terceros / NOTICE](#avisos-de-terceros--notice)
- [Créditos](#créditos)

---

## Arquitectura en Alto Nivel


1. El usuario hace una pregunta en lenguaje natural en un bot de Dialogflow.  
2. Dialogflow enruta al fulfillment webhook con un `tag` que indica el flujo (BigQuery vs Data Store).  
3. La función inicializa Vertex AI en la región configurada y crea/usa una sesión de chat global para mantener contexto corto.  
4. **Flujo BigQuery**  
   - Obtiene metadata de columnas.  
   - Gemini genera SQL válido.  
   - Se ejecuta la consulta.  
   - Los resultados se devuelven al modelo para formar una respuesta en texto.  
5. **Flujo Data Store**  
   - Ejecuta búsqueda semántica en Vertex AI Search.  
   - Los resultados se inyectan en un prompt y Gemini sintetiza la respuesta.  
6. Se retorna la respuesta a Dialogflow en formato fulfillment.

---

## Características Clave
- Generación automática de SQL a partir de instrucciones en lenguaje natural.  
- Respuestas conversacionales basadas en resultados reales de BigQuery.  
- Soporte de búsqueda (RAG) con Vertex AI Search.  
- Manejo de contexto multivuelta con reinicio por límite de tokens.  
- Parametrización completa vía `configs.py`.  
- Controles básicos de seguridad de contenido.  
- Logging estructurado.

---

## Requisitos Previos
- Proyecto GCP con facturación.  
- APIs habilitadas: Cloud Functions/Run, Vertex AI, BigQuery, Logging.  
- Cuenta de servicio con permisos mínimos (ver sección de seguridad).  
- Python 3.10+.  
- Vertex AI Search Data Store (opcional para `ds_webhook`).  
- Instancia de Dialogflow CX/ES.

---

## Variables de Configuración

| Variable              | Descripción                                                       | Ejemplo                   |
|-----------------------|-------------------------------------------------------------------|---------------------------|
| `PROJECT_ID`          | ID del proyecto GCP.                                              | `"mi-proyecto"`           |
| `BQ_DATASET`          | Dataset de BigQuery.                                              | `"ingesta"`               |
| `BQ_TABLE`            | Tabla principal.                                                  | `"productos"`             |
| `LOCATION_ID`         | Región Vertex AI.                                                | `"us-central1"`           |
| `ENGINE_ID`           | ID de Vertex AI Search Engine.                                    | `"my-search-engine"`      |
| `DATASTORE_LOCATION`  | Región del Data Store.                                            | `"global"`                |
| `MODEL`               | Modelo Gemini (override opcional).                               | `"gemini-2.5-pro"`        |

---

## Dependencias (requirements.txt sugerido)

```txt
functions-framework>=3.0.0
vertexai>=1.68.0            # o google-cloud-aiplatform[adk,agent_engines]
google-cloud-bigquery>=3.34.0
google-cloud-logging>=3.10.0
pandas>=2.2.0
pyarrow>=16.0.0
fastparquet>=2024.2.0       # opcional
requests>=2.31.0
pydantic>=2.7.0             # opcional
tiktoken>=0.7.0

Instalación Local
bash
Copiar
Editar
git clone https://github.com/corebi-latam/dialogflow-bq-webhook.git
cd dialogflow-bq-webhook
python -m venv .venv
source .venv/bin/activate       # o .venv\\Scripts\\activate en Windows
pip install -r requirements.txt
gcloud auth application-default login
# crea/edita configs.py con tus valores


Despliegue en Google Cloud
Cloud Functions 2nd gen
bash
Copiar
Editar
gcloud functions deploy dialogflow-webhook \
  --gen2 --runtime python312 --region=us-central1 \
  --source=. --entry-point=dialogflow_webhook \
  --trigger-http --allow-unauthenticated 

  Cloud Run
bash
Copiar
Editar
gcloud builds submit --tag gcr.io/$PROJECT_ID/dialogflow-webhook
gcloud run deploy dialogflow-webhook \
  --image gcr.io/$PROJECT_ID/dialogflow-webhook \
  --region=us-central1 --allow-unauthenticated
Integración con Dialogflow
Configura un Webhook Fulfillment apuntando a la URL pública y usa los tags:

jsonc
Copiar
Editar
{ "fulfillmentInfo": { "tag": "bq_webhook" }, "text": "¿Cuántos productos activos hay?" }
Tags:

bq_webhook → preguntas sobre BigQuery

ds_webhook → respuestas usando Vertex AI Search

Flujos Lógicos de Webhooks
Flujo BigQuery (bq_webhook)
Recibe user_query.

Obtiene columnas (get_table_columns).

Prompt BQ_SQL_GENERATION_PROMPT ➜ Gemini genera SQL.

Ejecuta SQL (run_query).

Prompt de respuesta con resultados tabulares.

Retorna texto.

Flujo Data Store (ds_webhook)
Búsqueda search_sample.

Prompt DATASTORE_RESPONSE_PROMPT.

Gemini responde.

Retorna fulfillment.

Límites de Tokens y Manejo de Sesión de Chat
python
Copiar
Editar
MAX_TOKENS = 32760
token_count = calculate_token_count(user_input)
if token_count > MAX_TOKENS or chat is None:
    chat = model.start_chat()
Prompts del Sistema
BQ_SQL_GENERATION_PROMPT

BQ_RESPONSE_GENERATION_PROMPT

DATASTORE_RESPONSE_PROMPT

BQ_GET_COLUMNS_SQL

Personalízalos según tu caso.

Logging y Observabilidad
python
Copiar
Editar
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
)
Recomendado: structured logging y custom metrics.

Buenas Prácticas de Seguridad y Costos
Cuenta de servicio (mínimos):

roles/bigquery.dataViewer

roles/bigquery.jobUser

roles/aiplatform.user

roles/discoveryengine.viewer

roles/logging.logWriter

Controles de costo: LIMIT, caché, alertas presupuestales, modelo gemini-2.5-flash si aplica.

Pruebas Locales
bash
Copiar
Editar
functions-framework --target=dialogflow_webhook --port=8080
curl -X POST localhost:8080 \
  -H 'Content-Type: application/json' \
  -d '{ "fulfillmentInfo": {"tag":"bq_webhook"}, "text":"cuantos productos hay?" }'

Solución de Problemas
Síntoma	Causa común	Acción
Invalid webhook tag	Tag incorrecto	Ajustar intent/route en Dialogflow
Error SQL	SQL inválido	Revisar prompt / validar columnas
Respuesta vacía ds_webhook	Sin resultados de búsqueda	Añadir fallback
Límite de tokens excedido	Input muy largo	Reiniciar sesión / resumir
Latencia alta	BigQuery pesado o LLM grande	Muestreo, vistas materializadas, Flash

Roadmap / Próximos Pasos
Validación y sanitización de SQL.

Parser robusto de bloques sql.

Esquema whitelist por seguridad.

Soporte multi‑tabla controlado.

Cache de resultados.

Internacionalización de respuestas.

Licencia y Atribuciones
Este proyecto combina código original de Corebi con muestras de Google LLC publicadas bajo Apache License 2.0.

yaml
Copiar
Editar
Copyright © 2025 Corebi.
Portions © 2024 Google LLC.
Distribuido bajo Apache 2.0. Véase LICENSE para detalles completos.

Avisos de Terceros / NOTICE
kotlin
Copiar
Editar
This product includes software developed by Corebi and
incorporates portions of Google LLC open‑source samples
licensed under the Apache License 2.0.
Créditos
Equipo de Ciencia de Datos & IA – Corebi LATAM

Muestras originales: Google Cloud OSS

Fecha del documento: 22 jul 2025 (America/Bogotá)