# Análisis de asistentes de voz existentes

Fecha: 12-oct-2026 · Método: análisis estático con **REA 6.4.0** (`rea analyze-javascript-application`) + lectura del código. No se ejecutó código de terceros.

## 1. Qué se analizó

| Producto | Tipo | Versión analizada | Por qué es relevante |
|---|---|---|---|
| **Leon AI** | Asistente personal open source (Node + Python) | commit `21baf11` | Lo más parecido a un "asistente personal" completo: wake word, STT, skills, TTS, memoria |
| **ElevenLabs Conversational AI** | SDK cliente comercial (`@elevenlabs/client`) | 1.27.0 | Captura de micrófono, protocolo, interrupciones en el cliente |
| **Vapi** | SDK cliente comercial (`@vapi-ai/web`) | 2.7.1 | Cliente "delgado" sobre WebRTC (Daily) |
| **LiveKit Agents** | Framework de agentes de voz (servidor) | 1.9.2 | Pipeline STT→LLM→TTS con turnos, interrupciones y latencia muy trabajados |
| **OpenAI Agents Realtime** | SDK speech-to-speech | 0.20.0 | Modelo único voz→voz, herramientas, handoffs, guardrails |

## 2. Arquitectura de referencia (lo que todos tienen en común)

```
 ┌──────────── CLIENTE ────────────┐          ┌──────────────── SERVIDOR / AGENTE ────────────────┐
 Mic → AEC/NS/AGC → (wake word)     │  WebRTC  │ VAD → STT streaming → detector de fin de turno     │
     → trozos PCM16 20–25 ms ───────┼────────▶ │        │ (transcripción provisional)                │
                                    │          │        ▼                                            │
 Altavoz ← cola de reproducción ◀───┼──────────│ LLM (streaming, herramientas, memoria, contexto)   │
   (worklet, avisa "terminé")       │          │        ▼ frases                                     │
 Estado: listening/thinking/speaking│◀─eventos─│ TTS streaming → audio con event_id                  │
 Interrupción: vaciar cola + fade   │          │ Barge-in: cancelar LLM+TTS, truncar historial       │
 └──────────────────────────────────┘          └─────────────────────────────────────────────────────┘
```

Dos variantes:
- **Pipeline (STT + LLM + TTS)**: LiveKit, Leon. Más control y proveedores intercambiables, pero hay que trabajar la latencia.
- **Speech-to-speech (realtime)**: OpenAI Realtime. Un solo salto y prosodia natural, pero el VAD y los turnos son una caja negra y dependes de un solo proveedor.

## 3. Hallazgos por producto

### Leon AI
- La voz es un **proceso Python aparte** (openWakeWord + faster-whisper + TTS local) que se comunica con el orquestador Node por TCP con mensajes `{topic: ...}`.
- **Wake word → escucha activa**: tras responder, la ventana de escucha se alarga con la duración de la respuesta, así puedes seguir hablando sin repetir "Hey Leon" (`tcp_server.py:239-247`).
- **Barge-in**: si detecta voz (umbral RMS) mientras Leon habla, emite `tts-interruption` y vacía la cola de frases (`brain.ts:114-144`).
- **Enrutado híbrido**: skills deterministas declarados en `skill.json` con parámetros en JSON-schema; un LLM elige skill y acción; si nada encaja, cae a un agente ReAct con herramientas.
- **Texto distinto para pantalla y para voz** (`answer.text` frente a `answer.speech`), y limpieza de emojis y HTML antes del TTS.
- **Memoria** en SQLite con ámbitos (persistente, diario, conversación), importancia, caducidad y deduplicación.
- **Debilidades**:
  - TCP sin delimitar mensajes: los separa con una regex.
  - Micrófono en el servidor, no en el cliente.
  - STT y TTS sin streaming: se espera 1 s de silencio y se genera el WAV completo.
  - Idioma `'en'` fijado en el código (`asr/api.py:153`).
  - `process.exit(1)` si falla la voz.

### ElevenLabs (cliente)
- Usa WebRTC/LiveKit por defecto y WebSocket como alternativa. En WebSocket hay un handshake (`conversation_initiation_client_data` → `conversation_initiation_metadata`) que fija el formato de audio (`pcm_16000`) antes de mandar audio.
- **Captura**: `getUserMedia` con echoCancellation, noiseSuppression, autoGainControl y mono, más un AudioWorklet que trocea en 25 ms, convierte a PCM16 o µ-law y re-muestrea.
- **Reproducción**: un worklet con cola avisa cuando termina, y el estado `speaking/listening` se deriva de la **reproducción real**.
- **Interrupción**: el servidor decide y manda `interruption{event_id}`. El cliente vacía la cola y hace un fundido de ganancia (2 s por defecto, `output.ts:205-230`) y descarta los eventos con `event_id` anterior.
- **Herramientas del cliente**: `client_tool_call` → `client_tool_result{tool_call_id, result, is_error}`. Siempre responde, también con error. La aprobación de herramientas MCP falla cerrada.
- `ping`/`pong` de aplicación con la latencia medida.

### Vapi (cliente)
- Es un cliente delgado: `POST /call/web` → sala Daily (WebRTC). Todo el audio, el VAD, los turnos y las herramientas están en el servidor.
- Solo 4 mensajes de cliente a servidor: `add-message`, `control`, `say` y `end-call`.
- Mide el arranque por etapas (`call-start-progress` con su duración), lo que sirve para encontrar cuellos de botella.
- **A evitar**: deduce "el asistente habla" por el nivel de audio con 1 s de margen, lo que da turnos imprecisos.

### LiveKit Agents (servidor)
- **Fin de turno adaptativo**: espera 500 ms por defecto, que sube hasta 3000 ms si un modelo de turn detection (sobre las últimas 6 entradas) cree que la frase está incompleta. Descuenta el silencio que ya pasó.
- **Falsa interrupción**: con voz ≥ 500 ms **pausa** el TTS; si en 2 s no hay transcripción, lo **reanuda**. No interrumpe durante el calentamiento del AEC (3 s).
- **Historial veraz**: si se interrumpe, guarda solo lo que se llegó a reproducir, marcado `interrupted: true`.
- **Generación preventiva**: lanza el LLM con la transcripción provisional y solo reutiliza la respuesta si transcripción, contexto y herramientas no cambiaron.
- LLM → TTS en streaming por frases, con filtro de markdown y emojis.
- Herramientas: máximo 3 pasos encadenados y después `toolChoice='none'`. Si una herramienta devuelve otro agente, hace handoff.
- **Robustez**: 3 reintentos (el primero a los 100 ms, luego cada 2 s; timeout de 10 s), adaptadores de fallback entre proveedores y cierre tras 3 errores irrecuperables.
- **Métricas**: retraso de EOU, retraso de transcripción, TTFT, TTFB y latencia end-to-end.

### OpenAI Agents Realtime
- Modelo voz→voz con `semantic_vad` y PCM a 24 kHz. Funciona sobre WebSocket, WebRTC o SIP.
- **Interrupción**: `conversation.item.truncate` con `audio_end_ms = min(tiempo transcurrido, duración generada)`, como entero, para que el modelo "sepa" lo que el usuario oyó.
- Herramientas con aprobación (`needsApproval`) y guardrails antes y después; las "herramientas en segundo plano" no generan respuesta.
- **Handoffs entre agentes** en la misma conexión con `session.update`.
- Guardrails de salida cada 100 caracteres: si saltan, interrumpen el audio y piden al modelo que se corrija.
- Solo mide tokens: no trae métricas de latencia ni reconexión automática.

## 4. Comparativa rápida

| | Leon | ElevenLabs | Vapi | LiveKit | OpenAI RT |
|---|---|---|---|---|---|
| Transporte de audio | TCP local (mic en el servidor) | WebRTC / WS | WebRTC | WebRTC | WS / WebRTC / SIP |
| Wake word | ✅ openWakeWord | ❌ | ❌ | ❌ | ❌ |
| VAD / fin de turno | RMS + 1 s de silencio | Servidor | Servidor | VAD + modelo EOU adaptativo | `semantic_vad` (servidor) |
| STT/TTS en streaming | ❌ | ✅ | ✅ | ✅ | ✅ (nativo) |
| Barge-in | Vacía la cola | Vacía la cola + fade | Servidor | Pausa / reanuda / cancela | Truncate |
| Historial truncado a lo oído | ❌ | Servidor | Servidor | ✅ | ✅ |
| Herramientas | Skills + agente ReAct | Cliente + MCP | Servidor | Paralelas, máx. 3 pasos | Aprobación + guardrails |
| Memoria a largo plazo | ✅ SQLite | ❌ | ❌ | Hook `onUserTurnCompleted` | ❌ |
| Métricas de latencia | ❌ | ping | Arranque por etapas | ✅ completas | Solo tokens |

## 5. Valores de partida recomendados (tomados del código)

| Parámetro | Valor | Origen |
|---|---|---|
| Trozo de audio de entrada | 20–25 ms, PCM16 mono 16 kHz | ElevenLabs |
| Espera mínima de fin de turno | 500 ms (300 ms con un detector de audio en streaming) | LiveKit |
| Espera máxima si la frase parece incompleta | 3000 ms (2500 ms) | LiveKit |
| Duración mínima de voz para interrumpir | 500 ms | LiveKit |
| Timeout de falsa interrupción | 2000 ms → reanudar | LiveKit |
| Ignorar interrupciones tras arrancar el AEC | 3000 ms | LiveKit |
| Máximo de pasos de herramientas por turno | 3 | LiveKit |
| Reintentos de proveedor | 3 (100 ms, luego 2 s; timeout de 10 s) | LiveKit |
| Fundido de audio al interrumpir | 100–300 ms (ElevenLabs usa 2 s, demasiado) | Recomendación |

## 6. Recomendaciones para nuestro asistente (por prioridad)

**P0: base sólida**
1. **Transporte WebRTC** para el audio, o WebSocket con frames binarios (no JSON en base64, que añade un 33 %). Mensajes de control en JSON con un `type` y un protocolo versionado.
2. **Captura con AEC, NS y AGC** del navegador o del sistema operativo. Sin cancelación de eco, el asistente se oye a sí mismo y se interrumpe.
3. **Streaming de punta a punta**: STT en streaming, LLM en streaming, TTS por frases. Es la mayor ganancia de latencia (el gran fallo de Leon).
4. **Máquina de estados explícita**: `idle → listening → thinking → speaking`, con `speaking` derivado de la reproducción real, no del envío.
5. **Barge-in completo**: al detectar voz del usuario, cancelar LLM y TTS, vaciar la cola con un fundido corto, descartar el audio con `event_id` antiguo y **truncar el historial a lo que realmente se oyó**.

**P1: que se sienta natural**

6. **Fin de turno adaptativo**: 500 ms base, hasta 3 s si la frase parece incompleta. Se puede empezar con una heurística (por ejemplo, termina en "y", "pero", "que"…) y pasar después a un modelo.
7. **Falsa interrupción**: pausar en vez de cancelar, y reanudar si en 2 s no hay palabras. Ignorar backchannels ("ajá", "sí").
8. **Texto para pantalla ≠ texto para voz**: filtrar markdown, emojis y URLs antes del TTS.
9. **Wake word + ventana de escucha activa** (como Leon), si quieres un asistente siempre escuchando.

**P2: capacidades de asistente personal**

10. **Herramientas con `tool_call_id`**, respuesta obligatoria (también con error), límite de pasos y **aprobación humana que falle cerrada** para acciones sensibles (enviar correos, pagos, borrar).
11. **Enrutado híbrido**: intenciones frecuentes y deterministas (temporizadores, recordatorios, música) vía skills declarativos, y el resto con un agente LLM.
12. **Memoria a largo plazo** con ámbitos e importancia (modelo de Leon), inyectada en el contexto al cerrar cada turno.

**P3: operación**

13. **Métricas por etapa** desde el primer día: EOU, STT, TTFT, TTFB y end-to-end (objetivo: menos de ~800 ms del fin de la voz del usuario al primer audio).
14. **Reintentos y fallback de proveedores**; degradar a modo texto en vez de cerrar el proceso.
15. **Configuración validada por esquema** y secretos en `.env`. Idioma configurable, nunca fijado en el código.

## 7. Sobre REA en este análisis

- **Lo que aportó**: inventario rápido (de 30 s a 2 min por paquete) de módulos, eventos y listeners (por ejemplo los 120 eventos de OpenAI Realtime y los 30 de Vapi), variables de entorno, `fetch`/`WebSocket` reales y timers. Sirvió de mapa para saber qué leer.
- **Limitaciones vistas**:
  - Muchos "endpoints" eran falsos positivos y no detectó URLs construidas con template literals.
  - No analiza Python (la voz de Leon).
  - `recover-javascript-sources` necesita Wakaru, así que los source maps se extrajeron a mano.
  - `trace-application-feature` no encontró coincidencias con semillas de texto.
- Toda la lógica de turnos, interrupciones y herramientas se entendió **leyendo el código**, con REA como guía.
