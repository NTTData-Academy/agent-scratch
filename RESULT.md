| **Mensaje** | **Intención** | **latency_ms** | **prompt_tokens** | **completion_tokens** |
| :--- | :--- | :--- | :--- | :--- |
| Buenos días, Jerry Aguilar. ¿En qué puedo ayudarte hoy respecto a robo o pérdida de tarjeta, movimientos no reconocidos o un reclamo? | greeting | 3092ms | 609 | 796 |
| Las comisiones por retirar en un cajero de otro banco varían según el tipo de tarjeta/cuenta y el cajero operador. No puedo dar un monto exacto aquí. Para ver la tarifa aplicable revisa el tarifario en la web o la app de Banco Demo, o comunícate con Atención al Cliente y te indicarán la comisión específica para tu cuenta | product_question | 10965 ms | 390 | 482 |
| Quiero ver las comisiones que me cobraron este mes | data_query | 5677ms | 492 | 496 |
| Quiero que me devuelvan una comisión cobrada dos veces | sensitive_action | 7556ms | 609 | 796 |
| ¿Cuál es la capital de Vietnam? | out_of_scope | 5477ms | 819 | 363 |

## Análisis de trade-offs: modelo, latencia y costo

### Trade-off 1: Modelo pequeño vs. modelo de mayor capacidad

**Observación:** El agente registra una latencia promedio de 6,553 ms por interacción. La intención `product_question` presenta la mayor latencia, con 10,965 ms.

**Decisión:** Evaluar `gpt-5-nano` para la clasificación de intenciones y mantener `gpt-5-mini` para las respuestas que requieren mayor comprensión del contexto.

**Beneficio esperado:** Reducir el costo de inferencia y potencialmente la latencia de clasificación.

**Riesgo:** Un modelo más pequeño podría clasificar incorrectamente solicitudes ambiguas, especialmente las relacionadas con operaciones sensibles.

**Validación pendiente:** Comparar ambos modelos utilizando los mismos mensajes y medir precisión, latencia y consumo de tokens antes de adoptar el cambio.

### Trade-off 2: Reducir MAX_TURNS de 10 a 4

**Observación:** Las cinco ejecuciones consumieron 5,852 tokens en total, con un promedio de aproximadamente 1,170 tokens por interacción.

**Decisión:** Evaluar reducir `MAX_TURNS` de 10 a 4 para limitar la cantidad de interacciones permitidas por conversación.

**Beneficio esperado:** Controlar el consumo acumulado de tokens y reducir el costo máximo potencial de las conversaciones extensas.

**Riesgo:** Algunas solicitudes complejas podrían necesitar más de cuatro turnos para resolverse correctamente.

**Validación pendiente:** Medir la cantidad real de turnos necesarios para completar las solicitudes y estimar el costo mensual con el tráfico esperado. El ahorro dependerá del comportamiento de los usuarios y no será necesariamente proporcional a la reducción de MAX_TURNS.

### Conclusión

Se recomienda priorizar la optimización del modelo utilizado para clasificar intenciones y evaluar límites de conversación según su complejidad, manteniendo controles adicionales para operaciones sensibles. Las decisiones deberán validarse con pruebas de calidad, latencia y costo antes de aplicarse en producción.