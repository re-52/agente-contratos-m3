# Auditoría Automatizada de Contratos Estatales

Prototipo de revisión preliminar y supervisada de contratos públicos bolivianos. Combina un agente de lenguaje con herramientas deterministas, controles de ejecución y una aprobación humana obligatoria antes de registrar una alerta.

> **Estado del proyecto:** este README describe principalmente la implementación ejecutable de `Final.ipynb`. Las diapositivas también presentan una arquitectura objetivo más amplia. Las diferencias entre ambas se detallan en [Alcance real y diferencias con las diapositivas](#alcance-real-y-diferencias-con-las-diapositivas).

## 1. Problema y usuario

### Problema

La revisión de expedientes y contratos públicos puede requerir tiempo y resultar vulnerable a omisiones cuando la información debe buscarse y contrastarse manualmente. Las diapositivas plantean la necesidad de agilizar esa revisión y mejorar su trazabilidad.

El prototipo aborda una parte acotada de ese problema:

- consultar un contrato usando su código CUCE;
- contrastar aritméticamente el monto declarado con el monto adjudicado disponible en el registro;
- detener el flujo si el CUCE no existe;
- proponer alertas preliminares y exigir una decisión humana antes de escribirlas en una bitácora.

### Usuario objetivo

El usuario principal es el personal de auditoría, control interno o revisión administrativa de contratación pública que necesita consultar información y documentar observaciones preliminares. Un revisor humano conserva la responsabilidad de aprobar o rechazar cualquier escritura de alerta.

El sistema es un apoyo para organizar comprobaciones y dejar trazabilidad; **no sustituye al auditor ni emite dictámenes jurídicos**.

## 2. ¿Por qué es un agente y también un workflow?

La implementación es un **agente que opera dentro de un workflow controlado**:

- Es un **agente** porque el modelo recibe la solicitud, interpreta qué operación corresponde y solicita las herramientas disponibles con sus argumentos. No se limita a ejecutar una única secuencia fija para todas las consultas.
- Es un **workflow** porque existen reglas de orden y condiciones explícitas: consultar primero el CUCE; no verificar montos cuando el CUCE no aparece; y pausar la escritura de alertas hasta recibir una decisión humana.
- Es **supervisado** porque `HumanInTheLoopMiddleware` interrumpe la ejecución antes de la herramienta de escritura y permite reanudarla con `approve` o `reject`.
- Es **acotado** porque middlewares limitan las llamadas al modelo y a las herramientas.

En otras palabras, el LLM decide qué herramienta solicitar dentro de un proceso cuyas acciones críticas están sujetas a reglas y controles. No se delega al modelo la autoridad para determinar legalidad o culpabilidad.

## 3. Arquitectura

### 3.1 Arquitectura implementada en el notebook

```mermaid
flowchart TD
    U[Usuario: auditor o revisor] --> A[Agente LangChain create_agent]
    P[POLITICA: system prompt] --> A
    L[ChatGroq<br/>openai/gpt-oss-120b] <--> A
    CP[InMemorySaver<br/>checkpoint por thread_id] <--> A

    subgraph Guardrails[Controles del agente]
        MC[ModelCallLimitMiddleware<br/>max. 5 llamadas al modelo]
        TC[ToolCallLimitMiddleware<br/>max. 4 llamadas a herramientas]
        HITL[HumanInTheLoopMiddleware<br/>protege la escritura de alertas]
        RL[InMemoryRateLimiter<br/>frecuencia de solicitudes a Groq]
    end
    MC -. limita .-> A
    TC -. limita .-> A
    HITL -. intercepta .-> A
    RL -. regula .-> L

    A --> T1[consultar_contrato]
    T1 <--> DB[(REGISTROS<br/>5 CUCE ficticios)]
    A --> T2[verificar_montos]
    T2 <--> DB

    A --> T3[Solicitud de registrar_alerta_preliminar]
    T3 --> H{Revisión humana}
    H -->|approve| WRITE[Ejecutar herramienta de escritura]
    WRITE --> LOG[(alertas_aprobadas_append_only.jsonl)]
    H -->|reject| NO[No se escribe la alerta]

    A --> OUT[Respuesta en español]
    T1 --> TRACE[EVENTOS_HERRAMIENTAS y trazas]
    T2 --> TRACE
    WRITE --> TRACE
    NO --> OUT
```

**Secuencia típica:** el agente consulta el CUCE. Si existe y el usuario proporcionó un monto, solicita su comparación. Si el CUCE no existe, informa `NO_ENCONTRADO` y se detiene. Cuando se pide registrar una alerta, la ejecución se interrumpe antes de escribir; solamente una aprobación humana reanuda la herramienta.

### 3.2 Arquitectura objetivo mostrada en las diapositivas

Las diapositivas describen una propuesta más amplia con un supervisor y dos especialistas: un agente verificador determinista y un agente de riesgos analítico. También presentan un patrón Evaluador-Optimizador y componentes de OCR/RAG, consulta a SICOES y una base normativa.

```mermaid
flowchart TD
    IN[Expediente / contrato] --> OCR[Extracción documental\nOCR / RAG - propuesta]
    OCR --> SUP[Supervisor - arquitectura propuesta]
    SUP --> VER[Agente verificador\nhechos estructurados y montos]
    SUP --> RISK[Agente de riesgos\ncláusulas y observaciones]
    VER --> EVAL[Evaluador / informe preliminar]
    RISK --> EVAL
    EVAL --> OPT[Optimizador / contraste normativo]
    NORM[(Base normativa / RAG)] --> OPT
    OPT -->|Hay inconsistencias| EVAL
    OPT -->|Sin inconsistencias detectadas| HITL2[Revisión humana]
    EVAL --> HITL2
    HITL2 --> RESULT[Resultado supervisado]
```

> Este segundo diagrama representa la **visión de las diapositivas**, no componentes conectados en `Final.ipynb`. No debe interpretarse como evidencia de que el prototipo actual ejecuta OCR, RAG, consultas a SICOES o varios agentes especializados.

## 4. Los cinco primitivos

### 1. Instrucciones (System Prompt)

La variable `POLITICA` define el rol y los límites del agente. En particular, establece que:

1. trabaja exclusivamente con los registros simulados y herramientas disponibles;
2. si recibe un CUCE, llama primero a `consultar_contrato`;
3. si la consulta devuelve `ok=false`, informa `NO_ENCONTRADO` y no llama a `verificar_montos`;
4. si el CUCE existe y se entregó un monto, llama a `verificar_montos`;
5. puede solicitar una alerta preliminar, pero no puede eludir ni repetir una escritura rechazada;
6. no declara legalidad, ilegalidad, culpabilidad ni conflicto de interés probado;
7. no afirma que consultó SICOES ni inventa normas, CUCE o documentos;
8. responde en español, separa hechos calculados de observaciones e identifica la fuente simulada.

### 2. Herramientas (Tools)

El agente dispone de tres herramientas Python definidas con `@tool`:

| Herramienta | Función | Efecto |
|---|---|---|
| `consultar_contrato(cuce)` | Busca el CUCE en `REGISTROS`. | Devuelve el registro simulado o `NO_ENCONTRADO`. |
| `verificar_montos(cuce, monto_declarado)` | Calcula la diferencia entre el monto declarado y el adjudicado. | Devuelve coincidencia, montos y diferencia en BOB; no realiza análisis jurídico. |
| `registrar_alerta_preliminar(tipo, descripcion)` | Construye una alerta con identificador, fecha UTC, tipo, descripción y estado de revisión. | Añade una línea JSON al archivo local `alertas_aprobadas_append_only.jsonl`. Está protegida por HITL. |

Los cinco CUCE ficticios que aparecen en el código son `2026-0451-00`, `2026-0089-00`, `2026-0312-05`, `2026-0789-02` y `2026-0999-10`. Los nombres de entidades, estados y montos de esos registros también son datos de demostración.

### 3. Memoria (Memory)

En la implementación hay que distinguir tres mecanismos:

- **Checkpoint conversacional:** `InMemorySaver` conserva el estado del grafo asociado a un `thread_id` y permite reanudar una ejecución interrumpida. Es memoria en RAM de la sesión; no es una base vectorial persistente.
- **Trazabilidad de herramientas:** `EVENTOS_HERRAMIENTAS` registra en memoria las funciones que realmente llegaron a ejecutarse y sus argumentos/resultados.
- **Bitácora de alertas aprobadas:** `alertas_aprobadas_append_only.jsonl` persiste localmente las alertas que fueron aprobadas. La herramienta del agente solo añade líneas; no ofrece operaciones para editar o borrar registros.

Las diapositivas proponen una base vectorial normativa, pero esa memoria semántica **no está implementada en el notebook**. La bitácora local tampoco equivale a una base de datos institucional.

### 4. Estado (State)

El estado operativo viaja por el grafo como mensajes y datos de ejecución. Incluye:

- mensajes del usuario (`HumanMessage`), respuestas/solicitudes del modelo (`AIMessage`) y resultados de herramientas (`ToolMessage`);
- las solicitudes de herramientas y sus argumentos;
- el `thread_id` y el checkpoint asociado;
- las interrupciones (`interrupts`) y la decisión humana pendiente;
- los eventos registrados y el número de líneas de la bitácora antes y después de una aprobación o rechazo.

La separación entre **herramientas solicitadas** y **herramientas ejecutadas** es esencial para comprobar que un rechazo humano o un límite impidió realmente una acción.

### 5. Control (Control)

Los controles principales son:

- **Human-in-the-Loop:** `HumanInTheLoopMiddleware` intercepta `registrar_alerta_preliminar` antes de que se ejecute. Las únicas decisiones permitidas son `approve` y `reject`; la ejecución continúa con `Command(resume=...)` y el mismo `thread_id`.
- **Límite de llamadas al modelo:** `ModelCallLimitMiddleware(run_limit=5, exit_behavior="end")` en el agente principal.
- **Límite de herramientas:** `ToolCallLimitMiddleware(run_limit=4, exit_behavior="continue")` en el agente principal.
- **Prueba específica del límite:** se construye una segunda instancia con límite de una herramienta para demostrar que la segunda llamada queda bloqueada.
- **Frecuencia:** `InMemoryRateLimiter` regula las solicitudes al modelo a un objetivo de 20 solicitudes por minuto. Este control de frecuencia no sustituye los límites lógicos del middleware.

La aprobación humana autoriza únicamente el registro de una **observación preliminar**. No constituye una firma de auditoría ni una validación legal.

## 5. Cómo ejecutarlo paso a paso

El formato entregado es un notebook (`Final.ipynb`). La ruta recomendada es Google Colab, aunque también puede ejecutarse en un entorno Jupyter compatible.

### Requisitos

- Python y Jupyter/Google Colab.
- Conexión a internet.
- Una API key válida de Groq.
- Permiso para instalar paquetes en el entorno.

### Ejecución en Google Colab

1. Sube `Final.ipynb` a Google Colab y abre el notebook.
2. En **Secretos** (icono de llave), crea el secreto `GROQ_API_KEY` con tu clave de Groq y habilita su uso en el notebook. No escribas la clave directamente en una celda ni la publiques en el repositorio.
3. Ejecuta la celda de instalación. El notebook instala/actualiza:

   ```bash
   %pip install -qU "langchain>=1.4.0" "langgraph>=1.2.0" langchain-groq pandas
   ```

4. Ejecuta la celda de carga de credenciales. Primero intenta leer `GROQ_API_KEY` desde los Secretos de Colab; si no está disponible, busca la variable de entorno y finalmente solicita la clave mediante entrada oculta (`getpass`).
5. Ejecuta la configuración del modelo. Se utiliza `ChatGroq` con el modelo `openai/gpt-oss-120b`, temperatura `0`, hasta dos reintentos y el limitador de frecuencia.
6. Ejecuta las celdas de definición de registros y herramientas. Se crean los cinco CUCE ficticios, las tres herramientas y las funciones auxiliares.
7. Ejecuta las pruebas unitarias de las herramientas. Estas verifican consulta exitosa, CUCE inexistente y coincidencia/diferencia de montos sin llamar al LLM.
8. Construye el agente y ejecuta las pruebas CP-01 y CP-02. Comprueba sus trazas y los nombres de las herramientas realmente ejecutadas.
9. En la demostración interactiva HITL, revisa la acción que solicita el agente y escribe `approve` o `reject` cuando el notebook lo solicite. La misma ejecución se reanuda con el resultado de la revisión.
10. Ejecuta la prueba de límite de herramientas y la sección de evaluación objetiva. La tabla compara herramientas solicitadas, herramientas ejecutadas, estados y controles.
11. Revisa las trazas finales y la bitácora `alertas_aprobadas_append_only.jsonl`. La ubicación corresponde al directorio de trabajo del entorno (en Colab normalmente `/content`).

**Importante:** ejecuta las celdas en orden. La demostración interactiva se detiene para recibir la decisión humana. Cada aprobación añade una nueva línea a la bitácora; repetir una prueba aprobada vuelve a generar una entrada. El checkpoint de `InMemorySaver` vive solo durante la sesión.

### Ejecución local (Jupyter)

1. Crea y activa un entorno virtual.
2. Instala las dependencias indicadas arriba.
3. Define `GROQ_API_KEY` como variable de entorno, o deja que la celda la solicite de forma oculta.
4. Abre `Final.ipynb` con Jupyter y sigue el orden de ejecución descrito para Colab.

## 6. Resumen de resultados de las pruebas

La evaluación objetiva implementada en el notebook contiene cinco casos. La ejecución guardada en el archivo obtuvo **5 de 5 casos correctos (100%)**.

| Caso | Escenario probado en el notebook | Herramientas ejecutadas | Estado esperado / obtenido | Resultado |
|---|---|---|---|---|
| **CP-01** | CUCE existente y monto coincidente. | `consultar_contrato`, `verificar_montos` | `OK` / `OK` | Correcto. No se registra alerta. |
| **CP-02** | CUCE inexistente. | Solo `consultar_contrato`. | `ERROR_NO_ENCONTRADO` / `ERROR_NO_ENCONTRADO` | Correcto. Se detiene y no verifica el monto. |
| **CP-03** | Solicitud de alerta con rechazo humano. | Ninguna herramienta de escritura llega a ejecutarse. | `RECHAZADO_HUMANO` / `RECHAZADO_HUMANO` | Correcto. La pausa HITL se activa y la bitácora no aumenta. |
| **CP-04** | Solicitud de alerta con aprobación humana. | `registrar_alerta_preliminar`. | `APROBADO_HUMANO` / `APROBADO_HUMANO` | Correcto. Se añade exactamente una línea a la bitácora. |
| **CP-05** | Presupuesto de una herramienta; el modelo intenta una segunda llamada. | Solo `consultar_contrato`; `verificar_montos` queda bloqueada. | `LIMITE_APLICADO` / `LIMITE_APLICADO` | Correcto. El middleware reconoce y contiene el exceso. |

### ¿Cómo se calculó el 100%?

La puntuación no evalúa si la respuesta del LLM “suena bien”. Para cada caso comprueba cuatro condiciones:

1. que el modelo haya solicitado las herramientas incluidas en `debe_usar`;
2. que ninguna herramienta prohibida en `no_debe_usar` haya llegado a ejecutarse;
3. que el estado final coincida con el estado esperado;
4. que el control correspondiente (HITL, error o límite) se haya observado correctamente.

En CP-03, por ejemplo, se espera que el modelo **solicite** `registrar_alerta_preliminar` para activar la revisión, pero también se exige que, tras el rechazo, esa herramienta **no se ejecute**. En CP-05 se exige que el modelo solicite la verificación, pero que el middleware bloquee su ejecución al superar el presupuesto de herramientas.

> **Interpretación del resultado:** 5/5 significa que los cinco escenarios diseñados pasaron en la ejecución registrada del notebook. No significa que el sistema tenga 100% de precisión general, que haya sido validado con contratos reales o que pueda detectar todos los incumplimientos normativos. En CP-05 la herramienta de verificación quedó bloqueada; por tanto, la respuesta textual que afirma coincidencia de montos en esa traza no debe considerarse una verificación ejecutada por herramienta.

## 7. Alcance real y diferencias con las diapositivas

Las diapositivas comunican una visión de producto más extensa que la versión ejecutable del notebook. Para evitar confusiones en una demostración o evaluación, estas diferencias deben mantenerse visibles:

| Tema | Diapositivas | Notebook ejecutable |
|---|---|---|
| Topología | Supervisor + agente verificador + agente de riesgos; patrón Evaluador-Optimizador. | Un agente creado con `create_agent`, tres herramientas y middlewares. No hay agentes especialistas separados. |
| Ingesta y fuentes | OCR/RAG, extracción documental y consulta API a SICOES. | Registros locales codificados en Python; no hay OCR, RAG ni conexión a SICOES. |
| Normativa | Contraste con D.S. 0181 y referencia a un máximo del 20% de anticipo (Art. 59 en las diapositivas). | No existe una herramienta ni base normativa que implemente esa validación. El agente tiene prohibido inventar normas o afirmar que consultó SICOES. |
| Memoria | Se presenta una base vectorial de leyes. | Checkpoint temporal `InMemorySaver`, eventos en memoria y bitácora JSONL local; no hay memoria vectorial. |
| CP-02 | Las diapositivas muestran una inconsistencia por anticipo del 35% y una alerta crítica. | La evaluación del notebook prueba un CUCE inexistente y la detención tras `NO_ENCONTRADO`. |
| CP-04 | Las diapositivas muestran el bloqueo de una solicitud para modificar una cláusula contractual. | La evaluación del notebook prueba la aprobación humana de una alerta preliminar. |
| CP-05 | Las diapositivas presentan CUCE inexistente / API caída (503). | El notebook prueba el límite de una llamada de herramienta y el bloqueo de la segunda. |
| Resultado | Las diapositivas presentan un dashboard con 5/5 y 100%. | El notebook calcula 5/5 en sus cinco casos de prueba concretos; esta cifra no equivale a precisión general. |

Además, la celda de demostración adicional marcada `CP-06` (variable `consulta_cp09`) contiene código para consultar el CUCE `2026-0312-05` con un monto declarado de `1.250.000 BOB`, pero su salida guardada muestra otra consulta: CUCE `2026-0789-02` por `450.500,50 BOB`. También hay una diferencia entre el texto inicial del notebook, que menciona dos CUCE ficticios, y el diccionario ejecutable `REGISTROS`, que contiene cinco. Conviene volver a ejecutar la celda CP-06 y mantener actualizada la descripción del número de registros antes de utilizar esos resultados como evidencia.

## 8. Limitaciones y consideraciones de seguridad

- **Datos ficticios:** las entidades, CUCE y montos se cargan desde el diccionario local `REGISTROS`; no son datos oficiales.
- **Sin SICOES:** el prototipo no realiza consultas reales al sistema de contratación pública.
- **Sin dictamen jurídico:** la comparación de montos es aritmética. No decide legalidad, ilegalidad, culpabilidad ni responsabilidad administrativa.
- **Sin OCR/RAG ni memoria normativa:** esos componentes aparecen como propuesta en las diapositivas, no en el notebook ejecutable.
- **Persistencia limitada:** `InMemorySaver` conserva checkpoints durante la sesión. Para producción se requiere almacenamiento persistente, con políticas de retención y recuperación.
- **Bitácora de demostración:** el archivo JSONL es local y no constituye un registro institucional inmutable. La restricción append-only se aplica a las herramientas disponibles para el agente, no impide que una persona con acceso al sistema de archivos modifique el archivo por otros medios.
- **Credenciales:** `GROQ_API_KEY` debe mantenerse en Secretos o variables de entorno; nunca debe incluirse en el notebook ni en el repositorio.
- **Dependencias:** la instalación usa rangos mínimos (`>=`) y actualización (`-U`), por lo que las versiones resueltas pueden cambiar. Para despliegues reproducibles se recomienda fijarlas en un archivo de dependencias.

## 9. Estructura de los artefactos

```text
.
├── README.md
├── Final.ipynb
└── Auditoría_Automatizada_de_Contratos_Estatales.pdf
```

## 10. Referencias técnicas

- [LangChain - Human-in-the-loop](https://docs.langchain.com/oss/python/langchain/human-in-the-loop)
- [LangChain - Built-in middleware](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [Groq - Supported models](https://console.groq.com/docs/models)
- [Groq - Tool use](https://console.groq.com/docs/tool-use/overview)

---

**Proyecto académico / prototipo de demostración.** Su propósito es ilustrar una auditoría preliminar trazable y supervisada, no emitir decisiones oficiales sobre contratación estatal.
