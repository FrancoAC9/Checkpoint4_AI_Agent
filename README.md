# Asistente de Calificación de Leads — Triunfo Seguros

Proyecto integrador del curso de **Automatización Avanzada**. Un sistema multi-agente en n8n que atiende el primer contacto de personas interesadas en contratar un seguro, clasifica la consulta, extrae los datos de calificación y deriva el lead al asesor humano.

No es un ejercicio de laboratorio: corre sobre un caso real, una productora asesora de seguros de Rosario, Argentina.

El proyecto **crece módulo a módulo**. Cada checkpoint importa el `.json` del anterior y le suma una capa, sin rehacer nada desde cero.

| Módulo | Capa que suma | Estado |
|--------|---------------|--------|
| M1 | Agente base conversacional + observabilidad | Entregado |
| M2 | Orquestación multi-agente (Manager–Worker) | Entregado |
| M3 | Memoria híbrida y persistencia correlacionada | Entregado |
| **M4** | **Integraciones reales: Gmail · HubSpot · Slack (OAuth2)** | **Actual** |
| M5 | RAG | Pendiente |
| M6 | Voz | Pendiente |
| M7–M11 | Hacia el Proyecto Final | Pendiente |

---

## Estado actual — M4

Hasta el M3 el sistema pensaba, delegaba y recordaba, pero vivía encerrado en un chat de prueba. El M4 lo conecta con las herramientas reales del negocio: **la casilla de consultas entra por Gmail, el contacto queda en un CRM (HubSpot) y el equipo se entera por Slack**. Cada conexión tiene un control preventivo no-code que evita los fallos clásicos de un sistema que ahora actúa sobre el mundo real.

### El flujo, de punta a punta

```
[Gmail Trigger · casilla de consultas]
        │
        ▼
① [IF — ¿es auto-reply?] ── Sí ─▶ (Stop: bucle cortado)
        │ No
        ▼
[Memoria M3 → Manager IA → Workers W1/W2]      ← todo lo heredado, intacto
        │
        ▼
④ [Set — payload limpio + validación del email]  (evita el Error 400)
        │
        ▼
② [Look up en HubSpot — ¿el contacto ya existe?]
   ┌────┴─────┐
   Sí          No
   ▼            ▼
[Update]    [Create]                             (evita el Error 409)
   └────┬─────┘
        ▼
③ [Gmail · Create Draft — borrador en el hilo]   (HITL: nadie recibe nada sin aprobación)
        │
        ▼
[Set — payload mínimo] → [Slack #leads-triunfo]
```

### Los cuatro controles

**① IF anti auto-reply.** Pegado al trigger. Escanea asunto y remitente, y descarta `Auto-reply`, `Out of office`, `Undeliverable`, `Delivery Status Notification`, sus equivalentes en español, y casillas `no-reply@`, `noreply@`, `mailer-daemon@` y `postmaster@`. La comparación ignora mayúsculas. Si el mail es automático, el flujo muere en un nodo `Stop` antes de tocar la memoria, el modelo o el CRM: sin eso, dos sistemas que se responden solos entran en un bucle infinito. Como segunda barrera, el trigger ignora los mails que salen de la propia casilla (`-from:me`).

**② Look up antes del Create.** Antes de escribir en HubSpot se busca el contacto por email exacto. Si existe, se actualiza; si no, se crea. El `Look up` tiene `Always Output Data` activo para que el caso "no existe" también avance y el `IF` pueda decidir. El `Update` completa nombre, ciudad y teléfono con lo que extrajo el agente, y si un dato no vino en este mail conserva el que ya estaba en el CRM en lugar de pisarlo con un vacío.

**③ Create Draft como Human-in-the-loop.** La respuesta al cliente nunca se envía: queda como **borrador dentro del hilo original** del mail, con destinatario y asunto `Re:` ya cargados. La asesora la revisa, la ajusta si hace falta y recién ahí la envía. Es la única salida de correo hacia el cliente en todo el Manager.

**④ Set de limpieza de payload.** Antes del CRM, un `Set` en modo *solo campos definidos* deja `From`, `Subject` y `BodyText` más la clasificación del agente, saca HTML y URLs, recorta el cuerpo y valida el formato del email. Un `IF` posterior corta si el email no es válido. Un segundo `Set` arma para Slack un único texto corto con la opción *Strip Binary* activa: al canal no llegan adjuntos ni el mail crudo.

### Mínimo privilegio

| Herramienta | Credencial | Qué puede hacer el workflow |
|-------------|------------|------------------------------|
| Gmail | OAuth2 | Leer la bandeja de entrada (no leídos, sin promociones ni redes) y crear borradores. No envía nada al cliente |
| HubSpot | OAuth2 | Solo el objeto Contactos: lee 5 propiedades y escribe 6 (email, nombre, ciudad, teléfono, nota y, al crear, la etapa `lead`) |
| Slack | OAuth2 | Publicar en un único canal fijo, `#leads-triunfo` |

### Qué cambió respecto del M3

- **Entrada:** el Chat Trigger se reemplazó por un Gmail Trigger. El `Session_ID` de la memoria pasa a ser **el email del remitente**, así cada persona tiene su propia memoria de largo plazo sin configuración adicional.
- **Aviso al equipo:** el reporte de trazabilidad que el M2 mandaba por Gmail ahora sale por Slack. El Manager dejó de enviar correos propios: su única acción de mail es crear el borrador.
- **Respuesta al cliente:** la confirmación *"tu consulta ya quedó en manos de un asesor"* solo se agrega cuando ya hay teléfono o email, y va al final. Si faltan datos de contacto, el borrador los pide sin prometer un llamado.

---

## Estructura del repositorio

```
.
├── README.md
├── M1/
│   ├── checkpoint1_franco_canova.json
│   └── PreEntrega_Modulo1_CanovaFranco.pdf
├── M2/
│   ├── manager_multiagente.json
│   ├── w1_analista_de_leads.json
│   ├── w2_redactor_de_notificaciones.json
│   └── preentrega_modulo2_canova_franco.pdf
├── M3/
│   ├── preentrega_modulo3_canova_franco.json
│   └── PreEntrega_Modulo3_CanovaFranco.pdf
└── M4/
    ├── checkpoint4_franco_canova.json        ← entregable
    └── workers/
        ├── w1_analista_de_leads.json
        └── w2_redactor_de_notificaciones.json
```

El entregable del M4 es un único workflow: el Manager, con 50 nodos. Invoca los dos sub-workflows del M2 mediante `Execute Workflow`. Los workers no cambiaron desde el M2; están copiados en `M4/workers/` para que la carpeta alcance por sí sola para reconstruir el sistema.

---

## Stack

- **n8n Cloud**: orquestación
- **Claude API (Anthropic)**: `claude-sonnet-5` para el agente, `claude-haiku-4-5` para la summarization
- **Airtable**: memoria de largo plazo (heredada del M3)
- **Gmail**: entrada de consultas + borradores HITL + aviso al asesor (W2)
- **HubSpot** (CRM gratuito): registro de contactos
- **Slack**: canal del equipo de operaciones

---

## Puesta en marcha

### 1. Credenciales

En n8n → **Overview → Credentials**, creá las tres conexiones con **Connect my account**: Gmail OAuth2, HubSpot OAuth2 y Slack OAuth2. Cada una tiene que mostrar el cartel verde de conexión exitosa. Además, las del M3: Anthropic API y Airtable.

En Slack, creá el canal `#leads-triunfo` e invitá la app con `/invite @n8n`: sin eso, la API rechaza el mensaje.

### 2. Importar los workflows (en este orden)

1. **Workers primero.** Import from File → `M4/workers/w1_analista_de_leads.json` y después `w2_redactor_de_notificaciones.json`, **cada uno en un workflow nuevo y vacío**. En W2, cambiá el destinatario del nodo Gmail por la casilla del asesor.
2. **Después el Manager.** Import from File → `M4/checkpoint4_franco_canova.json`.
3. En los nodos `Ejecutar · W1 Analista` y `Ejecutar · W2 Redactor`, elegí W1 y W2 del selector. Los IDs del JSON son los de la instancia original.
4. Asigná las credenciales en los nodos de Gmail, HubSpot, Slack, Airtable y Anthropic.
5. En `Slack · #leads-triunfo`, elegí el canal del desplegable.
6. La base de Airtable es la del M3: el esquema está en el README de esa entrega.

---

## Cómo probarlo

El trigger es por *polling*: en modo test, mandá el mail y después hacé clic en **Execute workflow**, que levanta el último mail no leído. Mandá los mails **desde otra cuenta**: los de la propia casilla los filtra el trigger.

| # | Mail de prueba | Qué tiene que pasar |
|---|----------------|---------------------|
| 1 | *"Hola, quiero asegurar un Gol Trend 2013"* desde una casilla nueva | HubSpot **Create** · borrador que pide teléfono o email · aviso en Slack con "contacto nuevo creado" |
| 2 | Desde la misma casilla, con nombre, localidad y teléfono | HubSpot **Update**, sin duplicar · memoria **RECURRENTE** · Slack con "contacto existente actualizado" |
| 3 | Asunto *"Out of office - Vacaciones"* | Corta en **Stop** · ningún otro nodo se ejecuta |

Resultados reales en la instancia de desarrollo:

| Ejecución | Caso | Resultado |
|-----------|------|-----------|
| #115 | Contacto nuevo | ✅ Create + borrador en el hilo + Slack `ok: true` |
| #118 | Contacto existente | ✅ Update (mismo `vid`, sin duplicado) + memoria recurrente |
| #121 | Auto-reply | ✅ Stop en 0,5 s, cero nodos posteriores |

---

## Decisiones técnicas

**El email del remitente como `Session_ID`.** En el chat del M3 la sesión la generaba n8n y moría con la pestaña. En una casilla de correo la identidad natural de la persona es su dirección, así que la memoria de largo plazo, el contacto del CRM y el hilo de Gmail quedan correlacionados por la misma clave.

**El Look up es explícito aunque HubSpot ofrezca "Create or Update".** El nodo de HubSpot solo expone la operación combinada, que ya evita duplicados por email. Igual se agrega el `Look up` más un `IF` visible en el lienzo por dos razones: la rúbrica lo pide como compuerta, y separar las ramas permite tratar distinto cada caso (el Create asigna la etapa `lead`; el Update no la toca, porque HubSpot no permite retroceder etapas del ciclo de vida).

**El borrador va al hilo, no a un mail suelto.** El `Create Draft` recibe el `threadId` del mail original. La asesora ve la consulta y la respuesta propuesta juntas, como si fuera a contestar a mano.

**Fan-out después del log.** El nodo de observabilidad reparte en dos ramas independientes: la de memoria (M3) y la de integraciones (M4). Si el CRM o Slack fallan, la memoria de la sesión se guarda igual.

---

## Gotchas

**El system prompt heredado no evaluaba la memoria.** En el M3 el System Message del agente no tenía el prefijo `=`, así que n8n no lo trataba como expresión: las variables `{{ $json.user_name }}` y el resto le llegaban al modelo como texto literal. La memoria se guardaba bien, pero no se inyectaba. En el M4 está corregido. Regla general: en n8n, cualquier campo que mezcle texto con `{{ }}` tiene que empezar con `=`.

**El Gmail Trigger en modo test levanta un solo mail.** En manual toma el último no leído que cumple el filtro. Para probar varios casos, ejecutá uno por vez, en el orden de la tabla.

**Sin `Always Output Data` en el Look up, el Create nunca corre.** Cuando HubSpot no encuentra el contacto devuelve cero ítems, y n8n corta la rama ahí. Con la opción activa pasa un ítem vacío, y el `IF` lo manda a Create.

**La app de Slack tiene que estar en el canal.** La credencial puede estar en verde y el post fallar igual con `not_in_channel`: `/invite @n8n` en el canal lo resuelve.

---

## Pendiente

- Detectar auto-respuestas también por encabezados (`Auto-Submitted`, `X-Autoreply`, `Precedence: bulk`), no solo por asunto y remitente. Cubre los auto-reply que llegan con asunto normal.
- Registrar el borrador en HubSpot como actividad asociada al contacto, así el historial de la conversación queda también en el CRM.
- Activar el workflow en producción con polling cada 1 minuto, y marcar como leído el mail procesado para que no se vuelva a levantar.

---

**Franco Canova**, Automatización Avanzada, 2026
