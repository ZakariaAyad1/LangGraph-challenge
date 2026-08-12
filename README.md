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
