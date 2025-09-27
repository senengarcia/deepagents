# Descripción general de DeepAgents

Este documento resume la arquitectura del paquete `deepagents` para ayudar a una persona recién incorporada al proyecto a orientarse en el código fuente.

## Visión general

`deepagents` implementa una variante enriquecida de agentes basados en LLM que añade planificación, subagentes y un sistema de archivos virtual, siguiendo ideas popularizadas por productos como Claude Code. El paquete expone constructores de agentes (sincrónicos y asíncronos) que ensamblan estas piezas usando LangChain y LangGraph.

La organización principal del código vive en `src/deepagents/` y se complementa con pruebas en `tests/` y ejemplos en `examples/`.

## Componentes clave

### Construcción del agente (`graph.py`)

- Define `create_deep_agent` y `async_create_deep_agent`, los puntos de entrada que usan `agent_builder` para componer el agente final.
- Ensambla un conjunto de *middleware* propios (planificación, sistema de archivos virtual, subagentes) y otros de LangChain (resúmenes, caché de prompt, aprobaciones humanas opcionales).
- Permite personalizar herramientas, instrucciones, modelo LLM, subagentes, *middleware* adicional y configuración de interrupciones para flujo human-in-the-loop.【F:src/deepagents/graph.py†L1-L114】

### Middleware propio (`middleware.py`)

- `PlanningMiddleware` añade la herramienta `write_todos` y extiende el *prompt* para fomentar la planificación explícita.
- `FilesystemMiddleware` agrega herramientas para un sistema de archivos virtual y su *prompt* correspondiente.
- `SubAgentMiddleware` habilita el tool `task`, que delega en subagentes (predefinidos o personalizados) y reutiliza la misma pila de middleware por defecto. Soporta ejecución sincrónica o asíncrona y admite redefinir modelos y middleware por subagente.【F:src/deepagents/middleware.py†L1-L164】

### Estado compartido (`state.py`)

- Extiende `AgentState` para almacenar lista de *todos* y archivos virtuales, con reductores que permiten combinar actualizaciones sucesivas dentro del grafo de LangGraph.【F:src/deepagents/state.py†L1-L32】

### Herramientas internas (`tools.py`)

- Implementa las herramientas de planificación (`write_todos`) y del sistema de archivos virtual (`ls`, `read_file`, `write_file`, `edit_file`).
- Gestiona la serialización de resultados mediante `Command` y `ToolMessage` para sincronizar el estado con el agente principal.【F:src/deepagents/tools.py†L1-L86】

### *Prompts* y mensajes guía (`prompts.py`)

- Contiene plantillas extensas que instruyen al agente sobre cuándo y cómo usar cada herramienta, con ejemplos detallados de uso y anti-ejemplos para la lista de tareas y demás comportamientos.【F:src/deepagents/prompts.py†L1-L120】

### Tipos y modelos (`types.py`, `model.py`)

- Define los *TypedDict* `SubAgent` y `CustomSubAgent`, incluyendo la opción de usar instancias de modelo o configuraciones en dict para subagentes.
- `get_default_model` devuelve la configuración por defecto (`ChatAnthropic` con Sonnet).【F:src/deepagents/types.py†L1-L18】【F:src/deepagents/model.py†L1-L6】

### Punto de entrada del paquete (`__init__.py`)

- Reexporta los constructores, middleware y tipos principales para facilitar la importación desde `deepagents` en lugar de rutas internas.【F:src/deepagents/__init__.py†L1-L6】

## Pruebas automatizadas

- `tests/test_deepagents.py` valida que el agente resultante incluye las herramientas incorporadas, soporta middleware externo, maneja subagentes personalizados y respeta canales de *streaming* adicionales.
- `tests/test_hitl.py` y `tests/test_middleware.py` cubren la lógica de interrupciones human-in-the-loop y del middleware respectivo (revisar para entender casos límite y expectativas de comportamiento).【F:tests/test_deepagents.py†L1-L104】

## Recomendaciones para aprender a continuación

1. **Revisar LangChain y LangGraph**: comprender cómo `create_agent`, los `AgentMiddleware` y los `Command` de LangGraph se combinan es crucial para extender el proyecto.
2. **Estudiar middleware personalizado**: los ejemplos en `tests/utils.py` muestran cómo inyectar herramientas y estado extra mediante middleware, útil para escenarios avanzados.【F:tests/utils.py†L1-L180】
3. **Explorar los prompts**: ajustar el comportamiento del agente implica modificar `prompts.py`; conviene entender la estructura y tono para realizar cambios seguros.
4. **Profundizar en pruebas**: ejecutar y leer las pruebas ayuda a internalizar los supuestos del diseño y proporciona una plantilla para añadir casos nuevos.
5. **Ejecutar los ejemplos**: `examples/research/research_agent.py` ilustra un caso de uso completo; replicarlo con tus herramientas facilita la puesta en marcha.【F:README.md†L1-L128】

## Preguntas frecuentes iniciales

- **¿Dónde agregar nuevas herramientas?** En la llamada a `create_deep_agent` (parámetro `tools`) o extendiendo `SubAgentMiddleware` para incluirlas en subagentes específicos.
- **¿Cómo compartir estado adicional?** Definir un `AgentState` propio (o extender `DeepAgentState`) y asociarlo en `agent_builder` mediante `context_schema` y middleware que gestione el reductor apropiado.
- **¿Se puede cambiar el modelo global?** Sí, pasando un objeto `LanguageModelLike` o una cadena soportada por LangChain al parámetro `model`. También se permite configurar modelos por subagente dentro de cada diccionario `SubAgent`.

Con esta base, la persona nueva debería centrarse en comprender el flujo de datos del estado (`AgentState`), las responsabilidades de cada middleware y cómo los prompts dirigen el comportamiento del agente. A partir de ahí, resulta natural iterar sobre nuevas herramientas, subagentes y mecanismos de revisión humana.
