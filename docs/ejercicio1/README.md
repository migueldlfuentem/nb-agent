# Ejercicio 1: análisis de las sesiones de negociación del agente NB3 de ejemplo

## Enunciado

> *Manual NB3, sección 5: Exercises, Exercise 1*
>
> Start a negotiation session by running the class `RunMarketAndFiveAgents`. After 30 seconds you
> should see the results of the negotiations.
>
> Now go to the `logfolderPath` specified in the source code of this class and check that indeed a
> folder has been created with the log files for all negotiators and the server.
>
> Open the server log file and determine how many proposals each agent has made. For each proposal
> message sent, the log file should contain something like:
>
> ```
> sent message:
> message ID: Bob 1
> conversation ID:
> from: Bob
> to: [Charles]
> particle: PROPOSE
> content: Proposal ID: Bob1
> Participants: [Bob, Charles]
> Proposed Deal: [agent 2 gives 7 units of 1 to agent 3, agent 3 gives 10 units of 0 to agent 2, agent 3 gives 10 units of 2 to agent 2]
> ```
>
> Of course, the details are different for each proposal. The line `particle: PROPOSE` indicates that
> this is a proposal. The log will also contain messages of type `REGISTER`, `INFORM` and `START`. You
> can ignore these, they form part of the communication protocol.
>
> If the negotiations were successful you will also find messages of type `ACCEPT` and `CONFIRM`.
> **How many times has a proposal been accepted? How many proposals have been confirmed?**
>
> Remember that the instance of the CommodityMarket is created randomly every time you run a new
> negotiation session. Therefore, in some cases the agents may not find any good proposal at all, so you
> may need to run several negotiation sessions before anything interesting happens. If after a couple of
> times the agents still don't make any proposals it may be better to continue with the other exercises
> to improve the agent and get back to this exercise with your improved agent.

Por mantener la coherencia respecto a las materias primas se mantienen los nombres en Inglés dados a estas en la práctica a lo largo de esta documentación.

## Sesiones realizadas

Se han realizado **7 sesiones** independientes de 30 s. Cada una genera una instancia aleatoria distinta
del mercado (activos y requisitos). En cada sesión se crea la carpeta de logs con un `server.log`, y dos
logs por agente: `<Agente>_nego.log` y `<Agente>_search.log`.

Los logs de cada sesión están en [`logs/`](logs/):

- `server.log`: todos los mensajes intercambiados, los activos y requisitos iniciales y las decisiones
  del Notario.
- `console.log`: salida de la sesión, que incluye la tabla final de utilidades (antes/después).
- `<Agente>_nego.log`: decisiones de cada agente (WAIT, PROPOSE, ACCEPT) y sus niveles de aspiración.

Los `<Agente>_search.log` (árbol de búsqueda) no se incluyen porque ocupan entre 20 y 80 MB por agente y
sesión.

## 1. Cómo se han contado los mensajes

El enunciado sugiere contar las propuestas buscando en el log las entradas con `particle: PROPOSE`. Sin
embargo, al analizar los logs se ha detectado que **sumar las ocurrencias de `PROPOSE` no es válido, porque
duplica registros**. El servidor escribe cada mensaje varias veces (broadcast), una al recibirlo del emisor
(`received message:`) y otra por cada destinatario al que lo reenvía (`sent message:`). Una misma
propuesta aparece, por tanto, como mínimo dos veces, y más si va dirigida a varios agentes.

La diferencia es considerable:

| Sesión | Propuestas reales (`received message`) | Entradas `sent message` | Total de líneas `particle: PROPOSE` |
|---|---|---|---|
| 1 | 2 | 2 | 4 |
| 4 | 38 | 49 | 87 |
| 6 | 31 | 33 | 64 |

Ni siquiera contar solo los bloques `sent message`, como en el ejemplo del enunciado, da el número
correcto. En la sesión 4 había propuestas a tres bandas (`[Bob, Eve, Charles]`) que se reenviaron a dos
destinatarios y figuran dos veces. Por ello se ha seguido este criterio:

- **PROPOSE** y **ACCEPT** se cuentan solo en los bloques `received message:`: cada bloque corresponde a
  un único mensaje enviado por un agente.
- **CONFIRM** e **ILLEGAL** se cuentan como **IDs de propuesta distintos**, porque los envía el `NOTARY`
  una vez por destinatario.
- `agent N` se traduce al nombre del agente con el orden de la tabla `REQUIREMENTS`. Ese orden es el del
  registro en el Notario, y sirve también como mapa ID → nombre: es el array `agentNames` con el que se
  inicializa el algoritmo NB3.

## 2. Resultados

Esta sección presenta los datos extraídos de los logs de las 7 sesiones: la actividad de cada sesión, las propuestas de cada agente y los acuerdos alcanzados. Aquí se describen los resultados; sus causas se analizan en la [sección 4](#4-interpretación).

### 2.1 Resumen por sesión

La siguiente tabla recoge, para cada sesión, el número de propuestas realizadas, los mensajes de aceptación (ACCEPT), los acuerdos confirmados por el Notario (CONFIRM) y las propuestas declaradas ilegales (ILLEGAL), junto con un enlace a sus logs:

| Sesión | Logs | Propuestas | ACCEPT | Confirmados | ILLEGAL |
|---|---|---|---|---|---|
| 1 | [sesion1](logs/sesion1_2026-11-14__20-28-22/) | 2 | 1 | 1 | 0 |
| 2 | [sesion2](logs/sesion2_2026-11-14__20-28-56/) | 0 | 0 | 0 | 0 |
| 3 | [sesion3](logs/sesion3_2026-11-14__20-29-31/) | 0 | 0 | 0 | 0 |
| 4 | [sesion4](logs/sesion4_2026-11-14__20-30-05/) | 38 | 12 | 2 | 0 |
| 5 | [sesion5](logs/sesion5_2026-11-14__20-30-41/) | 15 | 1 | 1 | 0 |
| 6 | [sesion6](logs/sesion6_2026-11-14__20-31-14/) | 31 | 3 | 1 | 1 |
| 7 | [sesion7](logs/sesion7_2026-11-14__20-32-49/) | 0 | 0 | 0 | 0 |
| **Total** | | **86** | **17** | **5** | **1** |

La actividad varía mucho de una sesión a otra: en 3 de las 7 no se hizo ninguna propuesta, mientras que las sesiones 4 y 6 concentran 69 de las 86. Además, el número de acuerdos confirmados es muy inferior al de aceptaciones (5 frente a 17), y una propuesta llegó a declararse ilegal (ver [4.1](#41-por-qué-en-algunas-sesiones-no-pasa-nada) y [4.2](#42-muchas-aceptaciones-pocas-confirmaciones)).

### 2.2 Propuestas por agente

La siguiente tabla desglosa las propuestas de la tabla anterior según el agente que las realizó, con una columna por sesión (S1–S7) y el total acumulado de cada agente:

| Agente | S1 | S2 | S3 | S4 | S5 | S6 | S7 | **Total** |
|---|---|---|---|---|---|---|---|---|
| Alice | 0 | 0 | 0 | 0 | 15 | 0 | 0 | **15** |
| Bob | 0 | 0 | 0 | 23 | 0 | 0 | 0 | **23** |
| Charles | 1 | 0 | 0 | 0 | 0 | 24 | 0 | **25** |
| David | 1 | 0 | 0 | 14 | 0 | 0 | 0 | **15** |
| Eve | 0 | 0 | 0 | 1 | 0 | 7 | 0 | **8** |

En cada sesión con actividad solo proponen uno o dos agentes, y quiénes son cambia de una sesión a otra. Ningún agente destaca de forma sistemática, algo esperable porque los cinco ejecutan el mismo código: el número de propuestas depende de la instancia (activos y requisitos de cada agente), no del agente.

### 2.3 Acuerdos confirmados y efecto en la utilidad

La siguiente tabla detalla los 5 acuerdos confirmados: quién lo propuso (el prefijo del ID de la propuesta), qué agentes participaron, qué intercambios incluía y cómo cambió la utilidad (dólares producidos) de cada participante entre el inicio y el final de la sesión:

| Sesión | Acuerdo (proponente) | Participantes | Contenido | Ganancia de utilidad |
|---|---|---|---|---|
| 1 | `David1` | David, Bob | David da 6 Wood; Bob da 10 Gold | David 12→14 (+2), Bob 2→4 (+2) |
| 4 | `David34` | David, Charles | David da 4 Grain; Charles da 150 Gold (15 acciones de 10) | David 13→30 (+17), Charles 15→17 (+2) |
| 4 | `Bob7` | Bob, Eve | Bob da 7 Gold; Eve da 70 Iron (7 acciones de 10) | Bob 4→18 (+14), Eve 15→17 (+2) |
| 5 | `Alice13` | Alice, Bob | Alice da 2 Gold; Bob da 130 Wood (13 acciones de 10) | Alice 25→39 (+14), Bob 5→6 (+1) |
| 6 | `Eve2` | Eve, Bob | Eve da 8 Grain; Bob da 20 Iron (2 acciones de 10) | Eve 20→23 (+3), Bob 22→26 (+4) |

Los 5 acuerdos confirmados son bilaterales, y en la mayoría la ganancia se reparte de forma muy desigual: en 3 de ellos el proponente gana entre 7 y 14 veces más que el otro participante (ver [4.3](#43-el-proponente-se-lleva-la-mayor-parte-de-la-ganancia)). También destaca que 4 de los 5 acuerdos repiten la misma acción de 10 unidades varias veces (ver [4.4](#44-acciones-repetidas-en-los-acuerdos)).

En las 7 sesiones **ningún agente terminó con menos utilidad de la inicial**, y en cada acuerdo
confirmado **todos los participantes ganan**. Es lo que el manual describe como resultado correcto
(sección 3.1).

## 3. Respuestas al ejercicio

Antes de dar respuesta a las preguntas del enunciado es importante mencionar que se están respondiendo en orden por lo que no se ha ejecutado ninguna mejora sobre los agentes para obtener los resultados de este ejercicio. Con este contexto se responden a las siguientes preguntas:

- **¿Cuántas propuestas ha hecho cada agente?** Depende mucho de la sesión, entre 0 y 24 por agente y
  sesión (tabla 2.2). En total hubo 86 propuestas en 7 sesiones, y en 3 sesiones nadie propuso nada. En
  cada sesión con actividad solo proponen 1 o 2 agentes. Esto será mejorado en los ejercicios 2 y 3 de la práctica.
- **¿Cuántas veces se ha aceptado una propuesta?** 17 mensajes ACCEPT en total (1, 0, 0, 12, 1, 3, 0).
- **¿Cuántas propuestas se han confirmado?** 5 acuerdos (1, 0, 0, 2, 1, 1, 0), además de 1 propuesta
  declarada ILLEGAL. Hubo acuerdo en 4 de las 7 sesiones.

## 4. Interpretación

### 4.1 Por qué en algunas sesiones no pasa nada

En las sesiones sin propuestas, todos los `*_nego.log` contienen solo decisiones `WAIT`:

```
Decision: WAIT, at time 29675 Currently none of the deals found by me or proposed to me is selfish enough.
my aspiration level: 0.0116 opponent aspiration level: 0.9896
```

Al final de la sesión la aspiración propia $m^{\alpha}_{\alpha}(t)$ ha bajado casi a 0 y la del oponente $m^{\alpha}_{\beta}(t)$ ha
subido casi a 1. Aun así, ningún agente encontró un plan que le diera utilidad normalizada positiva. Es
decir, en esa instancia no existe ningún acuerdo rentable, o la búsqueda no lo encontró en 30 s (el
espacio de acuerdos es enorme). El enunciado de la práctica ya nos advierte de que puede ocurrir: *"you may need to run a couple of negotiation sessions before you see any positive result"*.

Los valores del log coinciden con las fórmulas (1) y (2) del manual para los grados de concesión por
defecto **a1 = 2, a2 = 4**. Por ejemplo, en t = 6 s de 30 (t/t_dead = 0,2):
$m^{\alpha}_{\alpha} = 1 - \frac{e^{-0{,}4} - 1}{e^{-2} - 1} \approx 0{,}619$ y $m^{\alpha}_{\beta} = \frac{e^{-0{,}8} - 1}{e^{-4} - 1} \approx 0{,}561$.
El log muestra 0,621 y 0,557. Los agentes deciden cada 300 ms (el 1 % de la duración de la sesión), así
que salen 100 decisiones por agente.

### 4.2 Muchas aceptaciones, pocas confirmaciones

En el protocolo no estructurado un acuerdo solo es **vinculante cuando lo aceptan todos sus
participantes**. Proponer cuenta como aceptar.

- **Sesión 4:** Charles aceptó 10 propuestas de Bob (`Bob21`, `Bob23`, …), pero todas eran a tres bandas
  (`[Bob, Eve, Charles]`) y Eve nunca las aceptó, así que no se confirmó ninguna. Lo que sí se confirmó
  fueron dos acuerdos bilaterales (`David34` y `Bob7`).
- **Sesión 6, el ILLEGAL:** Charles propuso `Charles10`, que contiene exactamente las mismas acciones de
  `Eve2` (ya confirmado) más un intercambio propio con Eve. Eve tenía 14 de grano y en `Eve2` ya había dado
  8. `Charles10` le pide dar otras 8, lo que la dejaría con −2. Cuando Bob acepta, el Notario detecta la
  incompatibilidad con un acuerdo anterior y lo marca **ILLEGAL**. En sus logs, los agentes lo reflejan
  como *"Proposal Charles10 is inconsistent with earlier confirmed agreements"*.

### 4.3 El proponente se lleva la mayor parte de la ganancia

En 3 de los 5 acuerdos el proponente gana mucho más que quien acepta (+17 frente a +2, +14 frente a +2,
+14 frente a +1). Encaja con la estrategia NB3 (sección 2.4.1):

- al **proponer**, el agente elige el plan **más egoísta** de entre los que superan la aspiración del
  oponente;
- al **aceptar**, basta con que el plan sea individualmente racional y supere la aspiración propia, que
  baja con el tiempo.

Quien encuentra primero el acuerdo fija un reparto favorable para sí, y el otro lo acepta porque mejora
su situación aunque sea poco. NB3 nunca rechaza: las propuestas malas simplemente se ignoran.

### 4.4 Acciones repetidas en los acuerdos

Al confirmar un acuerdo, el log muestra una sola acción para un participante y la misma acción repetida
muchas veces para el otro. Por ejemplo, `Bob7` en la sesión 4:

```
CommodityMarketNotary.verifyValidity() DEAL CONFIRMED: Agents: [Bob, Eve]
Bob supplies 7 units of GOLD to Eve
Eve supplies 10 units of IRON to Bob      (×7)
```

Es literal: el acuerdo tiene 8 acciones y 7 son idénticas. El motivo es que `getSplitLabels()`, al crear
los nodos de cantidad, solo genera valores de 1 a 10 (`int maxValue = 10;`). Por tanto una acción mueve
como mucho 10 unidades, y para transferir más la búsqueda tiene que encadenar varias acciones con el
mismo proveedor, mercancía y consumidor. El límite lo pone el agente; el dominio admitiría hasta 99
unidades por transacción (`CommodityAssets.MAX_TRANSACTION_VALUE`).

Que un lado tenga 1 acción y el otro 7 depende de lo que necesita cada uno:

| | Requisitos por dólar | Activos iniciales | Cuello de botella | Con el acuerdo |
|---|---|---|---|---|
| Bob | 4 Oil, 5 Iron | 88 Oil, 20 Iron | Iron: 20/5 = 4 | +70 Iron → 90/5 = **18** |
| Eve | 5 Gold, 7 Wood | 78 Gold, 151 Wood | Gold: 78/5 = 15 | +7 Gold → 85/5 = **17** |

Bob necesita mucho hierro (70 unidades = 7 acciones de 10), mientras que a Eve le bastan 7 unidades de
oro (1 acción).

A continuación se detallan algunas consecuencias derivadas de esta situación:

- **Ramas muy profundas.** Cada acción ocupa 4 niveles del árbol (proveedor → mercancía → cantidad →
  consumidor), así que `Bob7` necesita 32 niveles y `Alice13` (130 de madera, 14 acciones), 56. Los
  acuerdos que requieren grandes cantidades son difíciles de alcanzar en 30 s.
- **Representaciones múltiples.** La misma transferencia total puede componerse de muchas formas
  (10+10+…, 9+10+…), y eso va en contra de la recomendación de la sección 4 del manual de que cada acuerdo
  tenga una única representación en el árbol.

Es un margen de mejora claro del agente: por ejemplo, cantidades con saltos crecientes (1, 2, 5, 10, 20,
50…) o ajustadas a lo que realmente necesita el consumidor.

### 4.5 El ruido en los modelos de los oponentes

Cada agente conoce sus propios requisitos exactos, pero los de los demás le llegan **con ruido**. Por eso
calcula las cotas y la utilidad de los oponentes de forma aproximada. En estas 7 sesiones no causó
pérdidas, pero sí lo observamos en una ejecución exploratoria previa (un agente bajó de 3 a 0 de
utilidad). Así que no está garantizado que el resultado sea individualmente racional para todos.

## 5. Conclusiones

- Al ser instancias aleatorias, los números concretos cambian de una sesión a otra. Sin embargo, el comportamiento cualitativo se repite: sesiones sin acuerdo, muchas más aceptaciones que confirmaciones, ventaja del proponente y acciones repetidas.
- El agente de ejemplo es funcional pero conservador. Con los grados de concesión por defecto y 30 s de
  búsqueda, solo en algo más de la mitad de las sesiones llega a algún acuerdo. Los ejercicios 2 (mejor cota
  superior, más poda) y 3 (ajuste de los grados de concesión) atacan precisamente estas limitaciones.
