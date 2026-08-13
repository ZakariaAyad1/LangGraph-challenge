# LangGraph-challenge


Claro. Si ya conocías LangGraph, conviene refrescarlo desde la perspectiva actual y no simplemente repasar la API antigua: el ecosistema LangChain/LangGraph ha cambiado bastante y muchos tutoriales anteriores a LangGraph/LangChain v1 ya no representan la forma recomendada de construir agentes.

## 1. El modelo mental correcto

**LangGraph es un framework de orquestación para sistemas agénticos con estado.**

La idea central no es realmente «hacer un agente», sino modelar una ejecución como un **grafo dirigido que evoluciona sobre un estado compartido**:

[
State_t \xrightarrow{Node} State_{t+1}
]

Sus piezas fundamentales son:

* **State** → información compartida durante la ejecución.
* **Node** → función que lee y/o modifica ese estado.
* **Edge** → determina qué nodo se ejecuta después.
* **Conditional edge** → routing dinámico.
* **Checkpointer** → persiste el estado de ejecución.
* **Thread** → identifica una ejecución/conversación persistente.
* **Interrupt** → suspende la ejecución y permite intervención externa/humana.

Esto es importante porque LangGraph no debería entenderse simplemente como «LangChain para agentes». Su valor aparece cuando necesitas controlar **qué ocurre, cuándo ocurre y qué pasa cuando algo falla o requiere intervención**.

---

## 2. Del workflow al agente

Imagina un sistema de investigación:

```text
START
  │
  ▼
Planner
  │
  ▼
Researcher
  │
  ▼
Evaluator
  │
  ├── calidad insuficiente ──► Researcher
  │
  └── calidad suficiente
              │
              ▼
            Writer
              │
              ▼
             END
```

Cada caja puede contener un LLM, herramientas, código convencional o incluso otro grafo.

El detalle importante es el bucle:

```text
Researcher → Evaluator → Researcher
```

Un pipeline convencional suele ser:

```text
A → B → C → D
```

LangGraph empieza a aportar mucho más cuando tienes:

```text
A → B → C
    ▲   │
    └───┘
```

porque el sistema puede **decidir dinámicamente cómo continuar**.

---

## 3. State: probablemente el concepto más importante

Por ejemplo:

```python
class AgentState(TypedDict):
    messages: list
    objective: str
    research: list[str]
    iteration: int
    final_answer: str
```

Los nodos reciben ese estado:

```python
def researcher(state: AgentState):
    ...
    return {
        "research": new_results,
        "iteration": state["iteration"] + 1
    }
```

LangGraph combina esas actualizaciones con el estado existente.

Aquí aparece otro concepto importante: los **reducers**. Si varios nodos escriben sobre el mismo campo, necesitas definir cómo se combinan esos valores.

Por ejemplo, en lugar de reemplazar:

```text
messages = new_messages
```

puedes acumular:

```text
messages = messages + new_messages
```

Entender `State + reducers` evita bastantes diseños frágiles.

---

## 4. Routing: donde aparece la autonomía

Supongamos que un agente puede:

```text
User
 │
 ▼
LLM
 │
 ├── responde directamente ─────────► END
 │
 └── solicita tool
           │
           ▼
         Tools
           │
           └──────────────► LLM
```

El modelo puede producir una llamada a una herramienta y el grafo determina que debe ir al nodo `Tools`.

Después:

```text
tool result → LLM
```

y el ciclo continúa hasta que el modelo genera una respuesta final.

Este patrón es esencialmente la familia **ReAct**:

[
Reasoning \rightarrow Action \rightarrow Observation \rightarrow Reasoning
]

con LangGraph controlando la ejecución.

---

## 5. Pero no confundas «agentic» con «LLM decide todo»

Este es uno de los errores arquitectónicos más frecuentes.

Un sistema:

```text
LLM → decide cualquier cosa → llama cualquier tool
```

parece muy agéntico, pero suele ser difícil de:

* depurar;
* evaluar;
* controlar;
* asegurar;
* reproducir;
* operar en producción.

Un diseño más robusto suele mezclar:

```text
deterministic workflow
        +
LLM decisions
        +
tool execution
        +
validation
        +
persistence
```

Es decir:

> **Autonomía donde aporta valor; determinismo donde necesitas garantías.**

LangGraph encaja especialmente bien con esta filosofía.

---

## 6. Persistencia y durable execution

Aquí LangGraph se diferencia bastante de implementar simplemente un loop en Python.

Con un **checkpointer**, el estado del grafo puede persistirse asociado a un `thread_id`.

Conceptualmente:

```python
config = {
    "configurable": {
        "thread_id": "conversation-123"
    }
}
```

Entonces tienes algo parecido a:

```text
Thread 123

checkpoint 1
     ↓
checkpoint 2
     ↓
checkpoint 3
     ↓
checkpoint 4
```

Esto permite recuperar ejecuciones, mantener conversaciones con estado y construir procesos que no dependen exclusivamente de que un único proceso Python permanezca vivo.

La documentación actual sigue situando la persistencia y ejecución durable entre las capacidades centrales del ecosistema LangGraph/LangChain. ([Docs by LangChain][1])

---

## 7. Human-in-the-loop

Supón que el agente prepara:

> Transferir £20,000 al proveedor X.

No quieres:

```text
LLM → payment API
```

Quieres:

```text
LLM
 │
 ▼
prepare transaction
 │
 ▼
INTERRUPT
 │
 ▼
Human approval
 │
 ├── reject → END
 │
 └── approve
       │
       ▼
 payment API
```

El grafo puede suspenderse conservando su estado y continuar después de recibir la decisión.

Esto convierte HITL en una propiedad arquitectónica del workflow en lugar de un `input()` improvisado.

---

## 8. Tools

Una tool no tiene por qué ser sofisticada:

```python
@tool
def search_customer(customer_id: str):
    ...
```

Puede representar:

```text
database
REST API
search engine
vector DB
filesystem
Python function
CRM
ERP
email
browser
another agent
```

El LLM decide **qué quiere hacer**; tu infraestructura determina **qué está autorizado a hacer realmente**.

Esta distinción es crucial en producción.

---

## 9. Multi-agent

Aquí conviene ser especialmente crítico.

Es tentador diseñar:

```text
Manager Agent
   │
   ├── Research Agent
   ├── Coding Agent
   ├── Data Agent
   ├── Critic Agent
   ├── Security Agent
   └── Writer Agent
```

porque visualmente parece sofisticado.

Pero más agentes ≠ mejor sistema.

Cada agente adicional introduce:

* más llamadas al modelo;
* más latencia;
* más coste;
* más estado;
* más posibilidades de loops;
* más dificultad para atribuir errores;
* más superficie de evaluación.

A menudo:

```text
1 agent + 8 tools
```

es mejor arquitectura que:

```text
8 agents + 8 tools
```

Usaría multi-agent cuando exista **separación real de contexto, capacidades, permisos o responsabilidad**, no simplemente para representar distintos prompts.

---

## 10. LangGraph vs LangChain hoy

Hay un cambio importante respecto a tutoriales antiguos.

Actualmente, **LangChain ofrece APIs de agentes de más alto nivel construidas sobre LangGraph**, mientras LangGraph queda como la capa de orquestación de menor nivel cuando necesitas controlar explícitamente ejecución, estado y workflows. Por eso no necesitas escribir manualmente un `StateGraph` para cualquier chatbot con herramientas. ([Docs by LangChain][1])

Una regla práctica:

```text
¿Agente relativamente estándar?
        │
        └── LangChain agent abstractions

¿Workflow complejo / custom?
        │
        └── LangGraph

¿Control de estado, branching,
loops, HITL, persistencia?
        │
        └── LangGraph
```

Y cuidado con tutoriales antiguos basados en APIs que posteriormente cambiaron o quedaron obsoletas.

---

## 11. Arquitectura que merece la pena dominar

Si quieres recuperar un nivel profesional en LangGraph, yo estudiaría este sistema:

```text
                   ┌───────────────┐
                   │     USER      │
                   └───────┬───────┘
                           ▼
                    ┌─────────────┐
                    │   ROUTER    │
                    └──────┬──────┘
                           │
               ┌───────────┴───────────┐
               ▼                       ▼
          simple query            complex task
               │                       │
               ▼                       ▼
            answer                  PLANNER
                                       │
                                       ▼
                                   EXECUTOR
                                       │
                              ┌────────┴────────┐
                              ▼                 ▼
                            TOOL             TOOL
                              │                 │
                              └────────┬────────┘
                                       ▼
                                   EVALUATOR
                                       │
                              ┌────────┴────────┐
                              │                 │
                            retry              OK
                              │                 │
                              └──► EXECUTOR     ▼
                                             HUMAN
                                            APPROVAL
                                               │
                                               ▼
                                             FINAL
```

Con:

```text
State
  ├── messages
  ├── task
  ├── plan
  ├── observations
  ├── tool_results
  ├── retries
  ├── validation
  └── final_answer
```

y además:

```text
checkpointing
structured outputs
tool calling
timeouts
retry policies
observability
evaluations
permission boundaries
```

Eso ya se aproxima mucho más a un **sistema agéntico de producción** que los ejemplos típicos de veinte líneas.

---

## 12. La distinción que quiero que retengas

Hay tres niveles que suelen mezclarse:

```text
LLM application
      ↓
LLM + tools
      ↓
Agent
      ↓
Agentic system
```

Un **agente** suele tener un loop del tipo:

[
Observe \rightarrow Reason \rightarrow Act \rightarrow Observe
]

Un **sistema agéntico** puede contener múltiples loops, agentes, workflows deterministas, memoria, herramientas, validadores y humanos.

Y **LangGraph no es el agente**.

LangGraph es principalmente la **infraestructura de orquestación que permite construir y controlar ese sistema**.

Esa diferencia conceptual es más importante que memorizar `add_node()` o `add_edge()`.

### Ruta rápida para refrescarlo

Yo lo haría mediante un proyecto incremental, no leyendo documentación de principio a fin:

1. **StateGraph básico** → state, nodes, edges y conditional edges.
2. **Agente ReAct** → model + tools + loop.
3. **Persistencia** → checkpointer + threads.
4. **Human-in-the-loop** → interrupts y resume.
5. **Agente robusto** → structured output, retries, validation y límites.
6. **Sistema avanzado** → planner/executor/evaluator.
7. **Multi-agent** → sólo después de poder justificar por qué varios agentes son mejores que uno.

El último punto es importante: empezar directamente por multi-agent suele enseñar patrones llamativos antes de enseñar buena arquitectura.

Si quieres hacerlo de forma práctica, podemos construir **desde cero un pequeño sistema agéntico en LangGraph**, aumentando su complejidad paso a paso y examinando en cada etapa el `State`, el grafo y las decisiones arquitectónicas.

[1]: https://docs.langchain.com/oss/python/deepagents/models?utm_source=chatgpt.com "Models - Docs by LangChain"








---------------------------------------------------------------------------------------------------------------------------------------

Claro. Esos son los conceptos de **Python que considero más importantes antes de meterte seriamente con agentes de IA**. Te los explico pensando específicamente en para qué los vas a utilizar.

### 1. Functions — funciones

Una función encapsula una operación reutilizable.

```python
def calculate_price(price, tax):
    return price * (1 + tax)

total = calculate_price(100, 0.21)
```

En IA agéntica son fundamentales porque las **tools de un agente suelen terminar siendo funciones**:

```python
def search_customer(customer_id: str):
    # consultar CRM
    ...
```

El LLM puede decidir: *"Necesito llamar a `search_customer`"*.

**Debes dominar:** parámetros, `return`, argumentos opcionales, `*args`, `**kwargs`, scope y funciones como objetos.

---

### 2. Classes — clases

Una clase agrupa **datos + comportamiento**.

```python
class Customer:
    def __init__(self, name: str):
        self.name = name

    def greet(self):
        return f"Hello {self.name}"
```

Después:

```python
customer = Customer("Carlos")
print(customer.greet())
```

Son importantes porque SDKs, clientes de APIs, agentes, herramientas y servicios suelen estar organizados mediante clases.

Por ejemplo:

```python
class EmailTool:
    def search(self, query):
        ...

    def send(self, recipient, message):
        ...
```

No necesitas convertirte en fanático de la programación orientada a objetos. Necesitas **leer y diseñar clases razonablemente bien**.

---

### 3. Type hints — tipos

Permiten expresar qué tipo de datos espera y devuelve una función.

Sin tipos:

```python
def search_customer(id):
    ...
```

Con tipos:

```python
def search_customer(customer_id: int) -> dict:
    ...
```

Esto te permite detectar errores antes y entender mucho mejor código grande.

En AI engineering son especialmente importantes porque trabajamos continuamente con **datos estructurados y schemas**.

```python
def calculate_total(
    price: float,
    quantity: int
) -> float:
    return price * quantity
```

Aprende bien:

```python
str
int
float
bool
list[str]
dict[str, int]
Optional
Literal
Union
```

y después:

```python
TypedDict
Protocol
Generic
```

Los últimos pueden esperar.

---

### 4. Dataclasses

Python permite crear clases utilizadas principalmente para almacenar datos.

En lugar de escribir mucho código:

```python
from dataclasses import dataclass

@dataclass
class Customer:
    name: str
    email: str
    age: int
```

Puedes hacer:

```python
customer = Customer(
    name="Ana",
    email="ana@example.com",
    age=32
)
```

Son útiles para representar:

```text
Customer
Order
AgentState
ToolResult
Configuration
Message
```

Por ejemplo:

```python
@dataclass
class AgentState:
    task: str
    attempts: int
    finished: bool
```

Esto empieza a ser muy relevante cuando construyes agentes con **estado**.

---

### 5. Pydantic

Pydantic es parecido conceptualmente a `dataclass`, pero está orientado fuertemente a **validación de datos**.

```python
from pydantic import BaseModel

class Customer(BaseModel):
    name: str
    age: int
```

Ahora:

```python
Customer(
    name="Ana",
    age="banana"
)
```

produce un error porque `"banana"` no es una edad válida.

¿Por qué es tan importante para IA?

Porque nunca deberías confiar ciegamente en los datos que vienen de:

```text
LLM
API
usuario
tool
database
```

Puedes definir:

```python
class SearchCustomerInput(BaseModel):
    customer_id: int
```

y validar los argumentos antes de ejecutar una tool.

Además, **FastAPI, frameworks de agentes y structured outputs** utilizan muchísimo este patrón.

De toda esta lista, **Pydantic merece especial atención**.

---

### 6. Decorators — decoradores

Probablemente hayas visto cosas como:

```python
@app.get("/customers")
def customers():
    ...
```

Ese:

```python
@app.get(...)
```

es un decorador.

Un decorador modifica o añade comportamiento a una función/clase.

También encontrarás cosas como:

```python
@tool
def search_database(query: str):
    ...
```

Conceptualmente:

```text
function
   ↓
decorator
   ↓
function + additional behavior
```

No necesitas escribir decoradores complejos inicialmente.

Pero sí debes poder **entenderlos y utilizarlos cómodamente**.

---

### 7. Generators — generadores

Una función normal puede devolver:

```python
return result
```

Un generator puede producir resultados progresivamente:

```python
def numbers():
    yield 1
    yield 2
    yield 3
```

Puedes consumirlos:

```python
for number in numbers():
    print(number)
```

Resultado:

```text
1
2
3
```

La diferencia importante es que no necesitas cargar todos los resultados simultáneamente en memoria.

Esto resulta útil procesando:

* documentos grandes
* datasets
* streams
* resultados progresivos

También te ayuda a comprender posteriormente el **streaming de respuestas de LLMs**.

---

### 8. Context managers

Seguramente has visto:

```python
with open("data.txt") as file:
    text = file.read()
```

`with` crea un contexto controlado.

Al terminar:

```python
with ...
```

Python se encarga de limpiar/cerrar recursos correctamente.

Es importante para:

```text
files
database connections
HTTP sessions
transactions
locks
resources
```

Ejemplo conceptual:

```python
with database.transaction():
    update_customer()
```

Si algo falla, puedes gestionar correctamente la transacción.

Al principio basta con **saber utilizarlos**. Después aprenderás a crear los tuyos.

---

### 9. async / await

Este es de los **más importantes para backend + agentes**.

Imagina que necesitas llamar tres APIs:

```text
OpenAI API → 2 segundos
CRM API    → 1 segundo
Search API → 2 segundos
```

Si haces todo secuencialmente:

```text
2 + 1 + 2 = ~5 segundos
```

Con concurrencia puedes realizar determinadas operaciones simultáneamente.

Python utiliza:

```python
async def search():
    result = await api_call()
    return result
```

En sistemas de agentes tendrás continuamente:

```python
await llm_call()
await search_database()
await call_api()
await execute_tool()
```

Porque gran parte del tiempo el programa está **esperando I/O**, no calculando.

Necesitas comprender muy bien:

```text
async def
await
coroutines
event loop
asyncio
gather
tasks
timeouts
```

No lo ignores. Mucha gente aprende Python superficialmente y después `async` se convierte en una fuente constante de errores.

---

### 10. Exceptions — excepciones

¿Qué pasa si Salesforce no responde?

```python
try:
    customer = search_salesforce()

except ConnectionError:
    ...
```

Las excepciones permiten gestionar errores.

En un agente real tendrás:

```text
API unavailable
timeout
invalid JSON
authentication expired
tool failure
database error
rate limit
LLM error
```

No puedes asumir:

```python
tool()
→ siempre funciona
```

Necesitas diseñar:

```text
try
 ↓
operation
 ↓
error?
 ├── no → continue
 └── yes
       ↓
    retry?
       ↓
    fallback?
       ↓
    abort?
```

Esta mentalidad es más importante que memorizar la sintaxis.

---

### 11. Modules — módulos

Cuando un proyecto crece, no quieres esto:

```text
agent.py
```

con 4.000 líneas.

Quieres algo como:

```text
agent_project/

    main.py

    agents/
        research_agent.py

    tools/
        search.py
        email.py
        database.py

    models/
        customer.py
        agent_state.py

    services/
        openai.py

    config/
        settings.py

    tests/
        test_tools.py
```

Los módulos/packages permiten dividir correctamente tu aplicación.

Aprenderás:

```python
import
from ... import ...
__init__.py
packages
relative imports
absolute imports
```

Esto parece trivial hasta que construyes proyectos medianos. Entonces se vuelve fundamental.

---

### 12. pytest — testing

Supongamos que tienes:

```python
def calculate_total(price, quantity):
    return price * quantity
```

Puedes escribir:

```python
def test_calculate_total():
    result = calculate_total(10, 3)

    assert result == 30
```

Ejecutas:

```bash
pytest
```

y automáticamente verificas que tu aplicación sigue funcionando.

Para agentes esto es crucial.

Puedes comprobar, por ejemplo:

```text
¿La tool rechaza parámetros inválidos?

¿El agente tiene prohibido enviar emails sin autorización?

¿Qué ocurre cuando una API devuelve 500?

¿Qué ocurre cuando hay timeout?

¿El parser procesa correctamente la respuesta?
```

Más adelante esto evoluciona hacia **agent evals**, donde no solamente verificas código sino el comportamiento del sistema de IA.

---

## Qué nivel necesitas realmente

No necesitas dominar los 12 conceptos al mismo nivel antes de continuar.

Yo los priorizaría así:

| Prioridad         | Conceptos                                  |
| ----------------- | ------------------------------------------ |
| 🔴 Imprescindible | Functions, exceptions, modules, type hints |
| 🔴 Imprescindible | `async/await`                              |
| 🔴 Imprescindible | Pydantic                                   |
| 🟠 Importante     | Classes                                    |
| 🟠 Importante     | pytest                                     |
| 🟠 Importante     | context managers                           |
| 🟡 Saber manejar  | decorators                                 |
| 🟡 Saber manejar  | dataclasses                                |
| 🟡 Saber manejar  | generators                                 |

Y hay una manera bastante mejor de aprenderlos que hacer **12 tutoriales independientes**.

Construye progresivamente una pequeña aplicación:

```text
Customer Support API
        │
        ├── FastAPI
        │
        ├── Pydantic models
        │
        ├── PostgreSQL
        │
        ├── async API calls
        │
        ├── exception handling
        │
        ├── external AI API
        │
        └── pytest
```

Cada concepto aparecerá porque **lo necesitas para resolver un problema**, no porque toque estudiarlo en un temario. Ese enfoque te prepara mucho mejor para IA agéntica.



-----------------------------------------------------------------------------------------------

Sí. La **Fase 3 — LLM Engineering** es donde pasas de “sé usar ChatGPT” a **“sé integrar un LLM dentro de software fiable”**.

La idea central es esta:

```text
Aplicación Python
      │
      ▼
   API del LLM
      │
      ▼
     Modelo
      │
      ▼
Respuesta estructurada
      │
      ▼
Tu aplicación decide qué hacer
```

Voy concepto por concepto.

### 1. Trabajar con APIs de modelos

Cuando usas ChatGPT, la interfaz se ocupa de casi todo. Como desarrollador, tu programa habla directamente con un modelo mediante una API.

Conceptualmente:

```python
response = model(
    instructions="Eres un analista financiero.",
    input="Analiza esta factura.",
)
```

El modelo devuelve una respuesta y tu programa continúa procesándola.

Por ejemplo:

```text
Usuario
   ↓
FastAPI
   ↓
LLM API
   ↓
Respuesta
   ↓
Python
   ↓
Database
```

Eso permite construir aplicaciones con IA, no simplemente conversaciones.

---

## 2. Prompting

Un prompt es la información/instrucción que proporcionas al modelo.

Malo:

```text
Analiza esto.
```

Mejor:

```text
Analiza la siguiente factura.

Identifica:
- proveedor
- fecha
- importe
- moneda

No inventes información ausente.
```

Pero hay un punto importante: **prompt engineering no consiste en encontrar palabras mágicas**.

En producción importa mucho más:

```text
instrucciones claras
+
contexto correcto
+
datos correctos
+
tools correctas
+
output estructurado
+
evaluaciones
```

que escribir prompts gigantescos.

---

# 3. System instructions

Permiten definir las reglas generales del comportamiento del modelo.

Por ejemplo:

```text
SYSTEM:

You are an invoice extraction system.

Extract only information explicitly present
in the provided document.

Never infer missing financial information.
```

Después llega el input:

```text
USER:

Invoice #1827
Microsoft Ltd
Date: 12/08/2026
Total: £4,250
```

La separación conceptual es:

```text
SYSTEM
¿Qué eres?
¿Qué reglas tienes?
¿Cómo debes comportarte?

USER
¿Qué quiere el usuario?
¿Qué datos proporciona?
```

En un agente esto se vuelve todavía más importante porque puedes establecer reglas como:

```text
Nunca ejecutes operaciones financieras
sin aprobación humana.
```

Aunque ojo: una instrucción del sistema **no sustituye controles de seguridad implementados en código**.

---

# 4. Structured Outputs

Este concepto es extremadamente importante.

Imagina que quieres extraer una factura.

Podrías decirle al LLM:

```text
Dime proveedor, precio y moneda.
```

Y obtener:

```text
Parece que el proveedor es Microsoft y
la factura tiene un valor de £4,250.
```

Esto está bien para una persona.

Es malo para software.

Tu aplicación necesita algo predecible:

```json
{
    "supplier": "Microsoft",
    "amount": 4250,
    "currency": "GBP"
}
```

Ahora Python puede hacer:

```python
invoice.amount
invoice.currency
invoice.supplier
```

Y puedes guardar esos datos en PostgreSQL.

Aquí está el cambio mental importante:

```text
Chatbot
LLM → texto → humano

Aplicación IA
LLM → datos estructurados → software
```

---

# 5. JSON Schema

Entonces aparece otra pregunta.

¿Cómo especificamos **exactamente** qué estructura queremos?

Con un schema.

Por ejemplo:

```json
{
    "supplier": "string",
    "amount": "number",
    "currency": "string"
}
```

Conceptualmente estás diciéndole al sistema:

> La respuesta tiene que cumplir este contrato.

En Python normalmente utilizarás herramientas como **Pydantic** para representar ese contrato:

```python
from pydantic import BaseModel

class Invoice(BaseModel):
    supplier: str
    amount: float
    currency: str
```

Entonces quieres conseguir:

```text
PDF
 ↓
LLM
 ↓
Invoice
 ↓
Pydantic validation
 ↓
Database
```

Aquí puedes ver por qué antes te recomendaba aprender **Pydantic**.

---

# 6. Streaming

Probablemente hayas observado que ChatGPT no espera 20 segundos y muestra toda la respuesta de golpe.

Va apareciendo progresivamente.

Eso es **streaming**.

Sin streaming:

```text
request
   ↓

[esperas 10 segundos]

   ↓
respuesta completa
```

Con streaming:

```text
request
   ↓
"The"
   ↓
"customer"
   ↓
"has"
   ↓
"three"
...
```

Esto mejora muchísimo la experiencia del usuario.

En Python encontrarás patrones similares a:

```python
for event in stream:
    print(event)
```

o con async:

```python
async for event in stream:
    ...
```

Otra razón por la que `async/await` termina siendo importante.

---

# 7. Retries

Las APIs fallan.

Por ejemplo:

```text
Tu aplicación
     ↓
LLM API
     ↓
503 Service Unavailable
```

Una aplicación amateur simplemente falla.

Una aplicación robusta puede hacer:

```text
request
 ↓
ERROR
 ↓
esperar 1 segundo
 ↓
retry
 ↓
ERROR
 ↓
esperar 2 segundos
 ↓
retry
```

Esto suele llamarse **exponential backoff**.

Pero tampoco debes hacer:

```python
while True:
    retry()
```

porque puedes crear loops infinitos, costes innecesarios y saturar servicios.

---

# 8. Rate limits

Los proveedores limitan cuánto puedes utilizar sus APIs en determinado periodo.

Conceptualmente:

```text
100 requests/minute
1,000,000 tokens/minute
```

Imagina que tienes 10.000 usuarios simultáneos.

Todos llaman:

```text
LLM
LLM
LLM
LLM
LLM
...
```

El proveedor puede responder:

```text
429 Too Many Requests
```

Necesitas saber gestionar:

```text
queues
rate limiting
backoff
concurrency
batching
```

Esto se vuelve muy importante al pasar de:

```text
"funciona en mi portátil"
```

a:

```text
"lo utilizan 50.000 personas"
```

---

# 9. Token / cost management

Los LLM no procesan conceptualmente “páginas”; procesan **tokens**.

Simplificando:

```text
texto
 ↓
tokenizer
 ↓
tokens
 ↓
LLM
```

Por ejemplo una frase puede convertirse conceptualmente en:

```text
"Artificial intelligence is powerful"

Artificial
intelligence
is
powerful
```

La tokenización real es más compleja, pero la idea basta inicialmente.

¿Por qué importa?

Porque normalmente:

```text
más tokens
   ↓
más coste
+
más latencia
+
más contexto utilizado
```

Imagina que envías 300 páginas al modelo para responder:

> ¿Cuál es el número de factura?

Eso es un mal diseño.

Tal vez necesitas recuperar solamente el fragmento relevante:

```text
300 páginas
     ↓
retrieval
     ↓
2 fragmentos relevantes
     ↓
LLM
```

Esto conecta directamente con **RAG**.

---

# 10. Context management

Este concepto será fundamental cuando construyas agentes.

Imagina una conversación de 500 mensajes.

Una estrategia ingenua sería enviar siempre:

```text
mensaje 1
mensaje 2
mensaje 3
...
mensaje 499
mensaje 500
```

al modelo.

Pero quizás el modelo solamente necesita:

```text
system instructions

información relevante del usuario

resumen de conversación

últimos 10 mensajes

documentos relevantes
```

Por tanto necesitas decidir:

> **¿Qué información necesita realmente el modelo para tomar esta decisión?**

Eso es context management.

En agentes profesionales es una competencia enorme.

---

# 11. Model selection

Otro error frecuente es utilizar siempre “el modelo más potente”.

Imagina:

```text
Tarea A:
clasificar email como spam/no spam

Tarea B:
analizar un contrato complejo

Tarea C:
planificar una investigación

Tarea D:
extraer tres campos de una factura
```

No necesariamente deberían utilizar el mismo modelo.

Puedes diseñar:

```text
simple classification
        ↓
modelo pequeño/barato

document extraction
        ↓
modelo intermedio

complex reasoning
        ↓
modelo avanzado
```

Tu objetivo es optimizar:

```text
calidad
latencia
coste
fiabilidad
```

No simplemente:

```text
"usar el mejor modelo"
```

---

# 12. Error handling

Esto une varias cosas anteriores.

Supongamos que tienes:

```python
invoice = extract_invoice(pdf)
```

¿Qué podría salir mal?

Muchísimas cosas:

```text
PDF corrupto
↓
texto ilegible
↓
API timeout
↓
rate limit
↓
modelo no encuentra importe
↓
estructura inválida
↓
database caída
```

Debes pensar explícitamente:

```text
¿Qué hago si esto falla?
```

Por ejemplo:

```text
LLM extraction
      ↓
validación
      ↓
¿válido?
 /          \
sí           no
↓             ↓
guardar     retry
              ↓
          sigue mal
              ↓
       revisión humana
```

Eso es mucho más cercano al trabajo profesional que simplemente aprender prompts.

---

# El proyecto Document Analyzer

Ahora podemos juntar todo.

Supongamos que quieres construir una aplicación donde una empresa sube facturas PDF.

Una factura contiene:

```text
INVOICE #19282

Supplier: ACME Ltd
Date: 10 August 2026

Laptop       £1,200
Monitor        £400

TOTAL        £1,600
```

Tu sistema debería hacer:

```text
             PDF
              │
              ▼
        extracción texto
              │
              ▼
             LLM
              │
              ▼
      Structured Output
              │
              ▼
           Pydantic
              │
         validación
              │
              ▼
         PostgreSQL
              │
              ▼
             API
```

El LLM podría producir:

```json
{
    "invoice_number": "19282",
    "supplier": "ACME Ltd",
    "date": "2026-08-10",
    "total": 1600,
    "currency": "GBP"
}
```

Y defines:

```python
class Invoice(BaseModel):
    invoice_number: str
    supplier: str
    date: date
    total: float
    currency: str
```

Entonces puedes guardar:

```text
invoice_number | supplier | date       | total | currency
19282          | ACME Ltd | 2026-08-10 | 1600  | GBP
```

Y crear endpoints:

```text
POST /invoices
GET  /invoices
GET  /invoices/19282
```

---

## Lo que realmente quiero que aprendas en esta fase

No memorices doce conceptos aislados. Aprende este flujo:

```text
                    ┌──────────────┐
                    │   Usuario    │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   FastAPI    │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   LLM API    │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │ Structured   │
                    │   Output     │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   Pydantic   │
                    │  Validation  │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │ PostgreSQL   │
                    └──────────────┘
```

Si puedes construir esto **sin copiar ciegamente un tutorial**, habrás dado un salto importante.

Y observa algo: **todavía no necesitas LangGraph, CrewAI ni sistemas multiagente**. Primero debes ser capaz de convertir un LLM en un componente fiable de una aplicación. Después, tool calling y agent loops resultarán mucho más fáciles de entender.










-----------------------------------------------









Perfecto. Vamos a aprender **RAG desde cero**, pero con mentalidad de desarrollador: primero entender qué problema resuelve, después cada pieza del pipeline y finalmente construirlo.

## 1. ¿Qué es RAG?

**RAG = Retrieval-Augmented Generation**, que podríamos traducir como **generación aumentada mediante recuperación de información**.

El problema fundamental es que un LLM no conoce necesariamente tus datos privados o recientes.

Imagina que tu empresa tiene 50.000 documentos internos y preguntas:

> "¿Cuál es nuestra política para aprobar gastos superiores a 10.000 €?"

El LLM tiene tres problemas:

1. Puede no conocer la política.
2. Puede conocer información desactualizada.
3. Puede inventarse una respuesta plausible.

Una solución ingenua sería meter los 50.000 documentos en el prompt.

Eso no es práctico.

RAG hace algo mucho más inteligente:

```text
Pregunta del usuario
        ↓
Buscar información relevante
        ↓
Encontrar 3-10 fragmentos relevantes
        ↓
Entregárselos al LLM
        ↓
LLM responde utilizando esa información
```

Por ejemplo:

```text
Usuario:
"¿Quién puede aprobar gastos >10.000€?"

                ↓

         RETRIEVAL SYSTEM

                ↓

Encuentra:

[Documento: Expense Policy]
"Expenses between €10,000 and €50,000
require approval from the Finance Director."

                ↓

               LLM

                ↓

"Los gastos entre 10.000€ y 50.000€
requieren aprobación del Finance Director.

Fuente: Expense Policy."
```

La idea esencial es:

> **El LLM no tiene que memorizar el conocimiento. Tiene que saber utilizar el conocimiento correcto cuando lo necesita.**

---

# 2. El pipeline completo

Tu esquema era:

```text
Documents
    ↓
Parsing
    ↓
Chunking
    ↓
Embeddings
    ↓
Vector / lexical index
    ↓
Retrieval
    ↓
Reranking
    ↓
Context
    ↓
LLM
```

Vamos pieza por pieza.

---

# 3. Documents

Primero tienes tus fuentes de conocimiento.

Pueden ser:

```text
PDF
Word
HTML
Wiki
Notion
Confluence
SharePoint
emails
manuales
tickets
bases de datos
```

Por ejemplo:

```text
documents/

├── expense_policy.pdf
├── remote_work_policy.pdf
├── security_policy.pdf
└── travel_policy.pdf
```

Todavía no podemos hacer demasiado con ellos.

Primero necesitamos **extraer su contenido**.

---

# 4. Parsing

Parsing significa convertir el documento original en información que nuestro sistema pueda procesar.

Por ejemplo:

```text
expense_policy.pdf
```

se transforma en:

```text
Expense Policy

Employees may submit travel expenses...

Expenses below €10,000 require...

Expenses between €10,000 and €50,000 require...
```

Pero un parser bueno debería conservar más que texto.

Idealmente:

```python
{
    "text": "Expenses between €10,000...",
    "page": 17,
    "document": "expense_policy.pdf",
    "section": "Approval limits"
}
```

Esto será importantísimo para generar **citas** posteriormente.

---

# 5. Chunking

Aquí aparece uno de los conceptos más importantes de RAG.

Supongamos que un PDF tiene 200 páginas.

No queremos tratar las 200 páginas como una única pieza.

Lo dividimos en fragmentos llamados **chunks**.

Por ejemplo:

```text
DOCUMENTO

████████████████████████████████████

                ↓

CHUNKS

██████
██████
██████
██████
██████
██████
```

Podríamos terminar con:

```text
chunk 1 → introducción
chunk 2 → política general
chunk 3 → gastos <10k
chunk 4 → gastos 10k-50k
chunk 5 → gastos >50k
...
```

¿Por qué?

Porque cuando alguien pregunta:

> ¿Quién aprueba gastos de 20.000 €?

queremos recuperar:

```text
chunk 4
```

no las 200 páginas.

---

# 6. Embeddings

Ahora llegamos al concepto que suele confundir inicialmente.

Un **embedding** convierte texto en una representación numérica.

Por ejemplo:

```text
"El perro está durmiendo"
```

podría convertirse conceptualmente en:

```text
[0.21, -0.73, 0.18, 0.91, ...]
```

Ese vector puede tener cientos o miles de dimensiones.

Otro texto:

```text
"Mi cachorro está dormido"
```

produce otro vector:

```text
[0.19, -0.70, 0.22, 0.88, ...]
```

Los números concretos no te interesan demasiado.

Lo importante es que textos con significado parecido tienden a ocupar regiones cercanas en ese espacio vectorial.

Conceptualmente:

```text
                  animales

       "perro dormido"
             ●
           ● "cachorro duerme"


                           "gato descansando"
                                ●



       "política financiera"
                  ●
```

Así podemos buscar por **significado**, no solamente por palabras exactas.

---

# 7. Semantic Search

Supongamos que el documento dice:

> "Employees must obtain authorization from their manager."

Pero el usuario pregunta:

> "¿Necesito permiso de mi jefe?"

Las palabras no coinciden exactamente:

```text
authorization ≠ permiso
manager       ≠ jefe
```

Pero semánticamente significan cosas parecidas.

Los embeddings permiten encontrar esa relación.

El proceso es:

```text
Pregunta
   ↓
embedding model
   ↓
vector de la pregunta
   ↓
comparar contra vectores almacenados
   ↓
encontrar vectores similares
```

Eso es **semantic search**.

---

# 8. Cosine Similarity

Necesitamos alguna manera matemática de preguntar:

> ¿Qué tan parecidos son estos dos vectores?

Una métrica muy utilizada es **cosine similarity**.

No necesitas dominar inicialmente toda la matemática.

Piensa en dos flechas:

```text
A ─────────►

B ────────►
```

Apuntan prácticamente en la misma dirección.

→ alta similitud.

Pero:

```text
A ─────────►

B
↑
│
│
```

apuntan en direcciones diferentes.

→ baja similitud.

Conceptualmente podríamos obtener:

```text
"perro durmiendo"
"cachorro dormido"

similarity = 0.94
```

frente a:

```text
"perro durmiendo"
"política fiscal"

similarity = 0.12
```

No confundas esto con una probabilidad. `0.94` **no significa "94% de probabilidades de que sean iguales"**.

---

# 9. Vector Database

Tenemos ahora miles o millones de embeddings.

Necesitamos almacenarlos y buscarlos eficientemente.

Ahí aparecen:

* PostgreSQL + pgvector
* Qdrant
* Pinecone
* Elasticsearch/OpenSearch

Para empezar, sigo recomendándote:

**PostgreSQL + pgvector.**

Podrías almacenar conceptualmente:

| id | text              | embedding     |
| -- | ----------------- | ------------- |
| 1  | Expense policy... | `[0.21,...]`  |
| 2  | Remote work...    | `[0.74,...]`  |
| 3  | Security...       | `[-0.18,...]` |

Cuando llega una pregunta, calculamos su embedding y buscamos los vectores más cercanos.

---

# 10. Pero semantic search no es suficiente

Aquí aparece una debilidad que muchos tutoriales esconden.

Imagina que buscas:

> `INC-938172`

Eso es un identificador exacto.

Semantic search puede ser peor que una búsqueda tradicional.

Para nombres, códigos, números y términos exactos, queremos también **keyword search**.

Una tecnología clásica para esto es:

**BM25**.

Simplificando:

```text
Semantic search
→ encuentra significado parecido

BM25
→ encuentra coincidencia textual relevante
```

---

# 11. Hybrid Search

Entonces podemos combinar ambas.

```text
                Query
                  │
          ┌───────┴───────┐
          ▼               ▼
   Semantic Search     BM25 Search
          │               │
          └───────┬───────┘
                  ▼
             combinar
                  ↓
              resultados
```

Esto se denomina **hybrid search**.

Y suele ser bastante más robusto que asumir:

> "Embeddings solucionan todas las búsquedas."

No lo hacen.

---

# 12. Metadata filtering

Supongamos que tenemos documentos de:

```text
Finance
HR
Engineering
Sales
Legal
```

Y el usuario pregunta sobre una política de **Finance**.

Podemos filtrar antes:

```python
department = "finance"
```

O:

```python
country = "UK"
year >= 2025
document_type = "policy"
```

Así evitamos buscar entre información irrelevante.

Un chunk podría tener:

```json
{
  "text": "...",
  "department": "finance",
  "country": "UK",
  "year": 2026,
  "document": "expense_policy.pdf",
  "page": 17
}
```

Los metadatos son extremadamente importantes en sistemas empresariales.

---

# 13. Retrieval

Ahora podemos entender qué significa realmente **retrieval**.

El usuario pregunta:

```text
"¿Quién aprueba un gasto de 20.000€?"
```

El sistema recupera quizá:

```text
Resultado 1 — score 0.91
Expense Policy — page 17

Resultado 2 — score 0.82
Finance Handbook — page 42

Resultado 3 — score 0.74
Travel Policy — page 8

...
```

Podríamos recuperar los **top 20** resultados.

Pero todavía tenemos un problema.

Que un sistema de búsqueda considere algo relevante no significa que sea realmente el mejor contexto para el LLM.

Ahí aparece el siguiente paso.

---

# 14. Reranking

El primer retrieval busca rápidamente candidatos.

Por ejemplo:

```text
10.000 chunks
      ↓
retrieval
      ↓
top 20
```

Después utilizamos un sistema más preciso para reordenar esos 20:

```text
20 candidates
      ↓
reranker
      ↓
1. Expense Policy p17
2. Finance Handbook p42
3. Expense Policy p18
...
```

Y quizá solamente enviamos los mejores 5 al LLM.

```text
10.000 documentos/chunks
        ↓
retrieval rápido
        ↓
20 candidatos
        ↓
reranking preciso
        ↓
5 mejores
        ↓
LLM
```

Esta arquitectura es muy común porque equilibra **velocidad y precisión**.

---

# 15. Context

Ahora construimos el contexto que recibirá el LLM.

Por ejemplo:

```text
SYSTEM:

Answer using only the supplied sources.
If the information isn't available, say so.

CONTEXT:

[Source 1]
Expense Policy, page 17:
Expenses between €10,000 and €50,000
require Finance Director approval.

[Source 2]
Finance Handbook, page 42:
...

USER:

Who must approve a €20,000 expense?
```

El LLM ahora tiene la información necesaria.

Y responde:

```text
Un gasto de 20.000 € debe ser aprobado
por el Finance Director.

Fuente: Expense Policy, página 17.
```

Eso es RAG.

---

# 16. Query rewriting

Hay otro problema.

Los usuarios escribimos preguntas terribles.

Por ejemplo:

> "¿y para los de más de 50k?"

Para una persona que conoce la conversación, está claro.

Para un buscador aislado, no.

Podemos transformar:

```text
"¿y para los de más de 50k?"
```

en:

```text
"approval requirements for expenses
greater than €50,000"
```

Eso es **query rewriting**.

El LLM puede convertir la pregunta original en una consulta optimizada para retrieval.

---

# 17. Retrieval evaluation

Ahora llegamos a una parte que no deberías saltarte.

Supongamos que haces un cambio y dices:

> "Creo que ahora mi RAG funciona mejor."

¿En qué te basas?

Necesitamos tests.

Creamos preguntas donde conocemos la respuesta/documento relevante:

```text
Question:
Who approves €20k expenses?

Expected document:
expense_policy.pdf

Expected section:
Approval Limits
```

Después ejecutamos 100 preguntas.

Podemos medir cosas como:

```text
¿apareció el documento correcto en top 1?

¿apareció en top 5?

¿apareció en top 10?
```

Esto permite comparar:

```text
RAG v1
72% correct retrieval

RAG v2
84%

RAG v3
91%
```

Ahora estás haciendo ingeniería, no ajustando parámetros por intuición.

---

# 18. La arquitectura completa

Juntándolo todo:

```text
                DOCUMENTOS
                    │
                    ▼
                 Parsing
                    │
                    ▼
                 Chunking
                    │
                    ▼
                Embeddings
                    │
                    ▼
          PostgreSQL + pgvector
                    │
                    │
────────────────────────────────────────

                  USER
                    │
                    ▼
            "Pregunta..."
                    │
                    ▼
             Query rewriting
                    │
                    ▼
              ┌─────┴─────┐
              ▼           ▼
           Vector        BM25
           Search        Search
              │           │
              └─────┬─────┘
                    ▼
              Hybrid Search
                    │
                    ▼
               Top candidates
                    │
                    ▼
                 Reranker
                    │
                    ▼
               Best chunks
                    │
                    ▼
                 Context
                    │
                    ▼
                   LLM
                    │
                    ▼
          Answer + citations
```

Éste es el modelo mental que quiero que retengas.

---

## Tu primera versión debería ser mucho más sencilla

No cometas el error de intentar construir todo eso inmediatamente.

**RAG v1:**

```text
PDF
 ↓
parse
 ↓
chunk
 ↓
embeddings
 ↓
PostgreSQL + pgvector
 ↓
semantic search
 ↓
top 5 chunks
 ↓
LLM
 ↓
answer
```

Cuando eso funcione y puedas **medir sus fallos**, añade:

```text
V2 → metadata filtering
V3 → BM25
V4 → hybrid search
V5 → reranking
V6 → query rewriting
V7 → evaluation avanzada
```

Ese orden importa. Si introduces diez componentes desde el principio y el sistema devuelve malas respuestas, no sabrás **qué componente está fallando**.

### El ejercicio que deberías poder resolver

Antes de pasar a agentes, deberías ser capaz de construir esto:

```text
50-100 PDFs
     ↓
PostgreSQL + pgvector
     ↓

Usuario:
"¿Cuál es nuestra política X?"

     ↓

tu aplicación recupera
los fragmentos correctos

     ↓

LLM responde

     ↓

respuesta + documento + página
```

Y, sobre todo, deberías poder diagnosticar independientemente dos preguntas:

**¿He recuperado la información correcta?** → problema de retrieval.

**¿El LLM recibió la información correcta pero respondió mal?** → problema de generation.

Esa separación entre **retrieval** y **generation** es una de las ideas más importantes que debes dominar antes de construir tu *Knowledge Agent*.


