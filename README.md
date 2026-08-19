# Inteligencia Artificial Generativa para creación de agentes

Curso diseñado para desarrolladores que desean diseñar, construir y desplegar soluciones de inteligencia artificial generativa utilizando modelos de OpenAI y Claude. A lo largo del curso se abordará el consumo de APIs, el desarrollo de aplicaciones basadas en agentes, la implementación de arquitecturas RAG y GraphRAG, el uso de Agent Skills, Harnesses, Model Context Protocol (MCP) y técnicas para evaluar, observar y desplegar soluciones de IA generativa sobre Microsoft Azure.

## Estructura

- `CapituloXX/README.md`: guía de laboratorio por capítulo.

## Lista de laboratorios

### Capítulo 1

- [1. Práctica 1. Desarrollar un script en Python que evalúe y compare automáticamente el costo, tiempo de respuesta y calidad de las respuestas obtenidas con modelos de OpenAI y Claude para un conjunto de solicitudes representativas del negocio.](Capitulo01/README.md#1-práctica-1-desarrollar-un-script-en-python-que-evalúe-y-compare-automáticamente-el-costo-tiempo-de-respuesta-y-calidad-de-las-respuestas-obtenidas-con-modelos-de-openai-y-claude-para-un-conjunto-de-solicitudes-representativas-del-negocio)
  - Descripción: Desarrollar un script en Python para evaluar y comparar automáticamente modelos de OpenAI y Claude considerando costo, tiempo de respuesta y calidad de las respuestas ante solicitudes representativas del negocio.
  - Duración estimada: 50 min
- [2. Práctica 2. Construir la arquitectura base de una solución GenAI utilizando FastAPI, implementando un Router de Modelos y preparando su integración con Azure AI.](Capitulo01/README.md#2-práctica-2-construir-la-arquitectura-base-de-una-solución-genai-utilizando-fastapi-implementando-un-router-de-modelos-y-preparando-su-integración-con-azure-ai)
  - Descripción: Construir la arquitectura base de una solución GenAI con FastAPI, incorporar un Router de Modelos y dejar preparada su integración con Azure AI.
  - Duración estimada: 60 min

### Capítulo 2

- [1. Práctica 1. Implementar un cliente Python que consuma modelos de OpenAI y Claude mediante sus SDK oficiales, configurando parámetros de inferencia y comparando sus respuestas frente a distintos escenarios de negocio.](Capitulo02/README.md#1-práctica-1-implementar-un-cliente-python-que-consuma-modelos-de-openai-y-claude-mediante-sus-sdk-oficiales-configurando-parámetros-de-inferencia-y-comparando-sus-respuestas-frente-a-distintos-escenarios-de-negocio)
  - Descripción: Implementar un cliente Python con los SDK oficiales de OpenAI y Claude, configurar parámetros de inferencia y comparar las respuestas de ambos modelos ante distintos escenarios de negocio.
  - Duración estimada: 30 min
- [1. Práctica 2. Desarrollar un Agent Skill reutilizable para encapsular una capacidad específica (por ejemplo, clasificación, extracción o resumen) e implementar un Harness que permita validar automáticamente distintos escenarios de entrada y salida.](Capitulo02/README.md#1-práctica-2-desarrollar-un-agent-skill-reutilizable-para-encapsular-una-capacidad-específica-por-ejemplo-clasificación-extracción-o-resumen-e-implementar-un-harness-que-permita-validar-automáticamente-distintos-escenarios-de-entrada-y-salida)
  - Descripción: Desarrollar un Agent Skill reutilizable que encapsule una capacidad específica e implementar un Harness para validar automáticamente distintos escenarios de entrada y salida.
  - Duración estimada: 35 min
- [2. Práctica 3. Implementar un flujo automatizado para revisión de código utilizando OpenAI o Claude para detectar oportunidades de mejora, vulnerabilidades y recomendaciones de refactorización.](Capitulo02/README.md#2-práctica-3-implementar-un-flujo-automatizado-para-revisión-de-código-utilizando-openai-o-claude-para-detectar-oportunidades-de-mejora-vulnerabilidades-y-recomendaciones-de-refactorización)
  - Descripción: Implementar un flujo automatizado de revisión de código con OpenAI o Claude orientado a detectar oportunidades de mejora, vulnerabilidades y recomendaciones de refactorización.
  - Duración estimada: 35 min
- [2. Práctica 4. Integrar los Agent Skills desarrollados en el módulo anterior dentro del flujo de revisión e implementar un Harness de evaluación para validar automáticamente la calidad, seguridad y consistencia de las respuestas antes de incorporarlas al proceso de desarrollo.](Capitulo02/README.md#2-práctica-4-integrar-los-agent-skills-desarrollados-en-el-módulo-anterior-dentro-del-flujo-de-revisión-e-implementar-un-harness-de-evaluación-para-validar-automáticamente-la-calidad-seguridad-y-consistencia-de-las-respuestas-antes-de-incorporarlas-al-proceso-de-desarrollo)
  - Descripción: Integrar los Agent Skills desarrollados previamente en el flujo de revisión e implementar un Harness de evaluación que valide automáticamente la calidad, seguridad y consistencia de las respuestas antes de incorporarlas al proceso de desarrollo.
  - Duración estimada: 40 min

### Capítulo 3

- [1. Práctica 1. Construir un pipeline de recuperación utilizando Semantic Chunking y Embeddings sobre una base documental técnica.](Capitulo03/README.md#1-práctica-1-construir-un-pipeline-de-recuperación-utilizando-semantic-chunking-y-embeddings-sobre-una-base-documental-técnica)
  - Descripción: Construir un pipeline de recuperación sobre una base documental técnica utilizando Semantic Chunking y Embeddings.
  - Duración estimada: 35 min
- [1. Práctica 2. Extender el pipeline mediante GraphRAG para responder preguntas considerando relaciones entre entidades almacenadas en un grafo de conocimiento.](Capitulo03/README.md#1-práctica-2-extender-el-pipeline-mediante-graphrag-para-responder-preguntas-considerando-relaciones-entre-entidades-almacenadas-en-un-grafo-de-conocimiento)
  - Descripción: Extender el pipeline de recuperación mediante GraphRAG para responder preguntas considerando las relaciones entre entidades almacenadas en un grafo de conocimiento.
  - Duración estimada: 40 min
- [2. Práctica 3. Implementar persistencia vectorial para almacenar y recuperar contexto mediante búsquedas semánticas.](Capitulo03/README.md#2-práctica-3-implementar-persistencia-vectorial-para-almacenar-y-recuperar-contexto-mediante-búsquedas-semánticas)
  - Descripción: Implementar persistencia vectorial para almacenar contexto y recuperarlo mediante búsquedas semánticas dentro de una solución GenAI.
  - Duración estimada: 40 min
- [2. Práctica 4. Implementar una solución GraphRAG utilizando FalkorDB y comparar su arquitectura, capacidades y casos de uso con plataformas como Weaviate y Neo4j para seleccionar la alternativa más adecuada según distintos escenarios.](Capitulo03/README.md#2-práctica-4-implementar-una-solución-graphrag-utilizando-falkordb-y-comparar-su-arquitectura-capacidades-y-casos-de-uso-con-plataformas-como-weaviate-y-neo4j-para-seleccionar-la-alternativa-más-adecuada-según-distintos-escenarios)
  - Descripción: Implementar una solución GraphRAG con FalkorDB y comparar su arquitectura, capacidades y casos de uso con Weaviate y Neo4j para seleccionar la alternativa más adecuada según distintos escenarios.
  - Duración estimada: 35 min

### Capítulo 4

- [1. Práctica 1. Preparar un dataset validado para Fine-Tuning a partir de una base de preguntas y respuestas utilizando Python.](Capitulo04/README.md#1-práctica-1-preparar-un-dataset-validado-para-fine-tuning-a-partir-de-una-base-de-preguntas-y-respuestas-utilizando-python)
  - Descripción: Preparar con Python un dataset validado para Fine-Tuning a partir de una base de preguntas y respuestas.
  - Duración estimada: 75 min

### Capítulo 5

- [1. Práctica 1. Implementar un agente con Function Calling que utilice herramientas externas para consultar información, ejecutar operaciones y colaborar con otros componentes de una solución GenAI.](Capitulo05/README.md#1-práctica-1-implementar-un-agente-con-function-calling-que-utilice-herramientas-externas-para-consultar-información-ejecutar-operaciones-y-colaborar-con-otros-componentes-de-una-solución-genai)
  - Descripción: Implementar un agente con Function Calling capaz de utilizar herramientas externas para consultar información, ejecutar operaciones y colaborar con otros componentes de una solución GenAI.
  - Duración estimada: 75 min
- [2. Práctica 2. Desarrollar un servidor MCP que permita a un agente interactuar de forma segura con sistemas externos y administrar múltiples herramientas disponibles.](Capitulo05/README.md#2-práctica-2-desarrollar-un-servidor-mcp-que-permita-a-un-agente-interactuar-de-forma-segura-con-sistemas-externos-y-administrar-múltiples-herramientas-disponibles)
  - Descripción: Desarrollar un servidor MCP para que un agente interactúe de forma segura con sistemas externos y administre múltiples herramientas disponibles.
  - Duración estimada: 75 min

### Capítulo 6

- [1. Práctica 1. Implementar un sistema de evaluación automática utilizando agentes especializados para medir precisión, consistencia y fidelidad de las respuestas generadas.](Capitulo06/README.md#1-práctica-1-implementar-un-sistema-de-evaluación-automática-utilizando-agentes-especializados-para-medir-precisión-consistencia-y-fidelidad-de-las-respuestas-generadas)
  - Descripción: Implementar un sistema de evaluación automática con agentes especializados para medir precisión, consistencia y fidelidad de las respuestas generadas.
  - Duración estimada: 80 min
- [2. Práctica 2. Instrumentar una solución basada en LangChain utilizando LangSmith para identificar cuellos de botella, errores y regresiones durante la ejecución de agentes.](Capitulo06/README.md#2-práctica-2-instrumentar-una-solución-basada-en-langchain-utilizando-langsmith-para-identificar-cuellos-de-botella-errores-y-regresiones-durante-la-ejecución-de-agentes)
  - Descripción: Instrumentar una solución basada en LangChain con LangSmith para identificar cuellos de botella, errores y regresiones durante la ejecución de agentes.
  - Duración estimada: 80 min

### Capítulo 7

- [1. Práctica 1. Preparar el despliegue de una solución GenAI utilizando Azure AI Foundry, Azure OpenAI Service y Azure Container Apps, aplicando buenas prácticas de seguridad y administración de secretos.](Capitulo07/README.md#1-práctica-1-preparar-el-despliegue-de-una-solución-genai-utilizando-azure-ai-foundry-azure-openai-service-y-azure-container-apps-aplicando-buenas-prácticas-de-seguridad-y-administración-de-secretos)
  - Descripción: Preparar el despliegue de una solución GenAI con Azure AI Foundry, Azure OpenAI Service y Azure Container Apps, aplicando buenas prácticas de seguridad y administración de secretos.
  - Duración estimada: 45 min
- [2. Práctica 2. Proyecto Integrador Técnico](Capitulo07/README.md#2-práctica-2-proyecto-integrador-técnico)
  - Descripción: Desarrollar una solución basada en Python que integre consumo de modelos OpenAI o Claude mediante Azure AI Foundry, API desarrollada con FastAPI, Agent Skills, Harnesses de evaluación, GraphRAG, FalkorDB, Model Context Protocol (MCP), evaluación automática mediante agentes y preparación para su despliegue en Microsoft Azure.
  - Duración estimada: 60 min

## Flujo de colaboración

- Trabajar en `changes_course`.
- Crear Pull Request hacia `main`.
- Merge por `Squash and merge`.
