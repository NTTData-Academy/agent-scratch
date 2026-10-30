# RESULTS — equipo-01 · dominio `fees`

Libreta del equipo: mediciones y decisiones. El `README.md` del proyecto no se toca.

## Semana 1 · Sesión 2 — Medición base

- Modelo: `gpt-5-mini` · Endpoint: Foundry (`aif-workshop-jsu3`) · Auth: `entra_id`
- `MAX_TURNS=10` · `FOUNDRY_TIMEOUT_S=30` · `FOUNDRY_MAX_RETRIES=2`
- Fecha: 2026-10-30
- Cada fila es una conversación nueva de un solo turno, para que `prompt_tokens` no arrastre historial.

| Mensaje | Intención esperada | Intención devuelta | latency_ms | prompt_tokens | completion_tokens |
|---|---|---|---|---|---|
| Hola | `greeting` | `greeting` ✓ | 4927 | 377 | 193 |
| ¿Cuánto cobran por retirar en un cajero de otro banco? | `product_question` | `product_question` ✓ | 4780 | 389 | 507 |
| Quiero ver las comisiones que me cobraron este mes | `data_query` | `data_query` ✓ | 6180 | 388 | 626 |
| Quiero que me devuelvan una comisión cobrada dos veces | `sensitive_action` | `sensitive_action` ✓ | 6702 | 389 | 712 |
| ¿A cuánto está el dólar hoy? | `out_of_scope` | `out_of_scope` ✓ | 3311 | 384 | 270 |
| **Promedio** | | **5/5 correctas** | **5180** | **385** | **462** |

### Observaciones

**1. La clasificación acertó 5/5.** El structured output de `AgentReply` funciona sin RAG ni tools. Base sólida, pero son 5 casos con el fraseo exacto del enunciado: no es evidencia de robustez en la frontera `data_query` / `sensitive_action`, que es la que importa.

**2. `prompt_tokens` es prácticamente constante (377–389, σ ≈ 5).** Casi todo es el system prompt de `fees.md` más el JSON schema; el mensaje del cliente aporta ~10 tokens. En el turno 1 el costo de entrada es fijo — pero ver el trade-off 2, porque eso cambia en cuanto la conversación avanza.

**3. `completion_tokens` varía 3.7× (193 → 712) y es lo que manda en la latencia.** Ajustando las filas con respuesta larga, el costo marginal sale en **~9.4 ms por token de salida**, con overhead fijo casi nulo. La latencia de este agente es el largo de la respuesta, no la red.

**4. El `greeting` es un outlier de 3.1 s.** Con 193 tokens de salida debería haber tardado ~1.8 s y tardó 4927 ms. Fue la primera llamada de la tanda: el exceso es la adquisición del token de Entra ID (`DefaultAzureCredential` en [credentials.py:21](src/agent/infrastructure/llm/credentials.py#L21)) más el handshake TLS. Se paga una vez por proceso, no por turno — **no lo tomes como latencia del modelo.**

**5. Pagas ~4× más tokens de salida de los que el cliente ve.** `gpt-5-mini` es un modelo de razonamiento: los *reasoning tokens* se facturan dentro de `completion_tokens` pero no aparecen en `answer`. Estimando el texto visible (~3.5 car/token en español):

| Mensaje | completion facturado | texto visible (est.) | invisible (est.) |
|---|---|---|---|
| Hola | 193 | ~54 | ~139 |
| cajero otro banco | 507 | ~123 | ~384 |
| comisiones del mes | 626 | ~103 | ~523 |
| devolución duplicada | 712 | ~146 | ~566 |
| dólar hoy | 270 | ~67 | ~203 |
| **Total** | **2308** | **~493** | **~1815 (79%)** |

Cuatro de cada cinco tokens de salida que pagamos son razonamiento interno. **Es el hallazgo más accionable de la medición** y es donde está el dinero, no en el prompt.

---

## Trade-offs de modelo y costo para el gate

### 1. Enrutar por intención: modelo chico para `greeting` y `out_of_scope`

**Decisión:** no lo hacemos todavía. Primero bajamos el razonamiento; enrutar queda como opción para T2.

**Evidencia:** `greeting` y `out_of_scope` son los dos turnos más baratos (193 y 270 tokens de salida) y juntos no llegan al 20% del consumo medido. El problema es estructural: para saber que un mensaje *es* un saludo hay que clasificarlo, y hoy `intent` y `answer` salen de **una sola llamada** ([models.py:33-37](src/agent/domain/models.py#L33-L37)). Enrutar obliga a partirlo en dos: una llamada barata que clasifica y otra que responde. Eso suma una ida y vuelta a **los cinco** tipos de mensaje — incluidos los caros — para ahorrar en los dos baratos. Con ~9.4 ms/token y un clasificador de ~20 tokens, la llamada extra cuesta poco en sí, pero paga de nuevo el overhead por request.

**Costo / riesgo que aceptamos:** seguimos usando `gpt-5-mini` para responder "Hola". Es desperdicio conocido y acotado. `Usage` ya es sumable justamente para esto ([models.py:42](src/agent/domain/models.py#L42)), así que la puerta queda abierta sin refactor.

**Antes de enrutar, probar lo barato:** bajar el esfuerzo de razonamiento para `greeting` / `out_of_scope`. Ataca el 79% invisible sin agregar una llamada.

### 2. `MAX_TURNS=10` vs `MAX_TURNS=4`

**Decisión:** bajar a **4**.

**Evidencia:** los 385 tokens de entrada medidos son el **mejor caso** — turno 1, sin historial. [chat_service.py:18](src/agent/application/chat_service.py#L18) recorta a `max_turns * 2` mensajes, o sea hasta 10 intercambios. Lo que se guarda es solo el texto visible (`result.reply.answer`), no el razonamiento: ~99 tokens de respuesta + ~15 del cliente ≈ **114 tokens por intercambio acumulado**.

| | prompt turno 1 | prompt en régimen | crecimiento |
|---|---|---|---|
| `MAX_TURNS=10` | 385 | ~1525 | 4.0× |
| `MAX_TURNS=4` | 385 | ~841 | 2.2× |

Son **~684 tokens de entrada por turno** de diferencia en conversaciones largas. Fórmula para el gate:

```
costo_mes = conversaciones/mes × turnos/conversación
            × (prompt_tokens × $/1M_entrada + 462 × $/1M_salida)
```

> **Pendiente:** completar `$/1M` con el tarifario vigente de `gpt-5-mini` en Azure AI Foundry para la región del workshop, y fijar el supuesto de tráfico. No inventamos las tarifas aquí.

**Costo / riesgo que aceptamos:** con 4 intercambios el agente "olvida" el contexto en reclamos largos. En `fees` un caso de devolución pide fecha, monto, descripción y últimos 4 dígitos — eso puede tomar 4–5 turnos de ida y vuelta, así que 4 queda **justo en el límite**. Si en pruebas el agente vuelve a pedir datos ya entregados, subir a 6 antes que a 10.

### 3. Timeout y reintentos (hallazgo operativo)

**Decisión:** mantener `FOUNDRY_TIMEOUT_S=30` y `FOUNDRY_MAX_RETRIES=2`.

**Evidencia:** la latencia medida va de 3.3 s a 6.7 s — **2× entre el mejor y el peor caso**, y el peor es `sensitive_action`, justo el que más importa. Un timeout de 5 s (probado durante la sesión) cortaba `data_query` y `sensitive_action`. El techo no lo pone la red sino el largo de la respuesta, así que un prompt que genere respuestas más largas corre el límite hacia arriba.

**Costo / riesgo que aceptamos:** con 30 s y 2 reintentos, el peor caso de cara al usuario es ~90 s antes del 504 de [routes.py:14](src/agent/infrastructure/api/routes.py#L14). Para un canal de chat es demasiado: revisar en T2, cuando haya datos de p95 real.
