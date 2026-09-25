# Checkpoint 4 — Integraciones avanzadas e interconexión de sistemas

**Copiloto de Estado de Proyectos** · Equipo Digital Garces Fruit
Constanza Moya · AI Automation Avanzado · Coderhouse

Evolución del proyecto integrador. Parte del workflow del Módulo 3 (arquitectura Manager-Worker con memoria persistente en Airtable) y lo conecta con tres herramientas externas vía OAuth2: **Gmail** como casilla de soporte, **HubSpot** como CRM de solicitantes y **Slack** como canal del equipo.

## Contenido del repositorio

| Archivo | Descripción |
|---|---|
| `checkpoint4_moya_constanza.json` | Workflow principal (Manager). Importable desde *Workflows → Import from File*. |
| `workers/worker1_analista_estado_moya_constanza.json` | Sub-workflow: consulta las planillas y analiza el estado del proyecto. |
| `workers/worker2_redactor_moya_constanza.json` | Sub-workflow: redacta la respuesta de negocio para el keyuser. |

Para ejecutar el flujo completo en otra instancia, importar primero los dos workers y luego reapuntar los nodos `Invocar Analista` e `Invocar Redactor` a los IDs que asigne la instancia. El export no contiene secretos: n8n solo guarda el ID y el nombre de cada credencial.

## Caso de negocio

Los keyusers de Garces Fruit escriben a la casilla del Equipo Digital preguntando por el estado de sus proyectos. El agente clasifica el correo, consulta las planillas, redacta la respuesta y la deja como **borrador en el mismo hilo** para que una persona del equipo la revise antes de enviarla. En paralelo registra al solicitante en el CRM y avisa al canal de Slack que hay un borrador esperando aprobación.

## Flujo

```
Gmail Trigger (INBOX, no leídos)
   │
   ▼
① Es Respuesta Automatica? ── Sí ─▶ Detener: Auto-reply
   │ No
   ▼
Contexto de Sesion  (normaliza remitente, asunto, cuerpo; corta historial citado)
   │
   ▼
Memoria de largo plazo (CP3) ─ Buscar Memoria → Existe Memoria? → Recuperada / Crear Registro
   │
   ▼
Router de Triaje (AI Agent) → Enrutador → Workers Analista + Redactor / respuestas directas
   │
   ▼
④ Limpiar Payload → Email Valido? ── No ─▶ Descartar: Payload Invalido
   │ Sí
   ▼
② Buscar Contacto CRM → Contacto Existe? ── Sí ─▶ Actualizar Contacto
   │                                   └─ No ─▶ Crear Contacto
   ▼
③ Crear Borrador (HITL)  ← única salida de correo del workflow
   │
   ▼
Limpiar Payload Slack → Notificar Slack
   │
   ▼
Escritura de memoria (CP3) ─ Contador → Supera 5? → Resumidor / Persistir Contador
```

## Los cuatro controles de la rúbrica

### ① IF anti auto-reply — corta el bucle infinito
Nodo `Es Respuesta Automatica?`, inmediatamente después del trigger. Combina once condiciones con OR sobre el asunto (`auto-reply`, `automatic reply`, `respuesta autom`, `out of office`, `fuera de la oficina`, `undeliverable`, `delivery status notification`, `mail delivery`) y sobre el remitente (`no-reply`, `noreply`, `mailer-daemon`). La rama verdadera termina en un NoOp: el correo automático no llega a ningún modelo ni conector.

El bucle se previene además por diseño. El workflow del Módulo 3 enviaba un correo de log a la misma casilla, y con un trigger de Gmail eso habría generado un ciclo: cada ejecución produciría un correo nuevo que dispararía otra ejecución. En esta versión la observabilidad pasó a Slack y el workflow **no envía ningún correo**. El trigger escucha solo `INBOX` y no leídos, de modo que tampoco ve los borradores ni los enviados.

### ② Look up antes del Create — evita el Error 409
`Buscar Contacto CRM` busca en HubSpot un contacto con email exactamente igual al del remitente. `Contacto Existe?` bifurca: si existe, `Actualizar Contacto` registra el asunto y la categoría de la última consulta; si no existe, `Crear Contacto` lo da de alta con nombre, apellido y empresa. La búsqueda tiene `alwaysOutputData` activado para que la ausencia de resultados llegue al IF como ítem vacío en vez de detener la ejecución.

En el nodo de HubSpot la única operación de alta de contactos es *Create or Update*. La compuerta de búsqueda sigue siendo la que decide qué rama se ejecuta, y la operación de alta es idempotente sobre el email como segunda línea de defensa.

### ③ Create Draft — Human-in-the-loop
`Crear Borrador (HITL)` usa exclusivamente la operación *Create Draft*, dirigido al remitente y dentro del mismo `threadId`. La persona del equipo abre Gmail, revisa, corrige si hace falta y envía. El cuerpo sale sin etiquetas internas de decisión (`RESPONDIDO`, `ESCALADO`) ni formato markdown, y con una firma que declara que la respuesta fue preparada con asistencia de IA.

### ④ Set de limpieza — evita el Error 400
`Limpiar Payload` descarta todo lo que no sea necesario aguas abajo: deja remitente, nombre, asunto, cuerpo truncado a 1000 caracteres, hilo, categoría y respuesta. `Email Valido?` verifica con una expresión regular que el email no esté vacío ni mal formado antes de tocar el CRM o Gmail.

Antes de Slack hay un segundo Set, `Limpiar Payload Slack`, que construye un único campo de texto acotado: solicitante, asunto, categoría, decisión y una vista previa de 280 caracteres. No viaja el cuerpo completo, no viaja el email del solicitante y no viajan binarios (el trigger tiene la descarga de adjuntos desactivada).

## Mínimo privilegio

| Conector | Operaciones usadas | Scopes que el workflow necesita |
|---|---|---|
| Gmail | Leer INBOX (trigger) · crear borradores | `gmail.readonly`, `gmail.compose` |
| HubSpot | Buscar, crear y actualizar contactos | `crm.objects.contacts.read`, `crm.objects.contacts.write` |
| Slack | Publicar en un canal | `chat:write` |

El mínimo privilegio se aplica en tres capas.

**Red.** Las credenciales de HubSpot y Slack tienen activado *Allowed HTTP Request Domains* en modo específico: la de HubSpot solo puede usarse contra `api.hubapi.com` y la de Slack solo contra `slack.com`. Aunque alguien reutilizara la credencial en otro nodo, no podría enviar el token a un dominio ajeno.

**Operaciones.** El workflow no contiene ningún nodo que envíe correo, borre registros ni lea mensajes de Slack. La única salida de correo es *Create Draft*.

**Datos.** En HubSpot se escriben solo cinco propiedades del contacto (email, nombre, apellido, empresa y mensaje). A Slack solo llega el texto armado por el Set de limpieza, sin cuerpo completo del correo, sin email del solicitante y sin binarios.

La restricción no se aplica en la capa de scopes, y lo declaro explícitamente. Las credenciales OAuth2 de n8n solicitan un conjunto fijo de scopes definido por el propio conector, más amplio que la tabla anterior: incluye deals, companies y tickets en HubSpot, y búsqueda e historial en Slack. Las apps de ambos proveedores deben declarar ese conjunto completo, porque si los scopes configurados no coinciden con los solicitados, la autorización falla. El paso siguiente para producción es activar *Custom Scopes* en la credencial de Slack y reemplazar las credenciales de HubSpot por una integración con scopes solo de contactos.

## Configuración de las apps OAuth2 (n8n self-hosted)

La instancia de n8n es self-hosted, por lo que cada proveedor requirió registrar una app propia con la URL de callback de n8n.

**HubSpot.** Desde junio de 2026 HubSpot no permite crear apps públicas desde la interfaz. La app se creó con la HubSpot CLI y el framework de Projects (`hs project create` → `hs project upload`), con distribución privada y autenticación OAuth. El archivo `app-hsmeta.json` declara la URL de redirección de n8n y los scopes que exige la credencial. Durante el deploy, HubSpot rechazó el scope heredado `tickets` y hubo que reemplazarlo por sus cuatro scopes granulares (`crm.objects.tickets.read/write`, `crm.schemas.tickets.read/write`).

**Slack.** App creada desde manifiesto JSON en un workspace de prueba, con la URL de callback de n8n y la rotación de tokens desactivada (si se activa, los tokens vencen cada 12 horas y la credencial deja de funcionar). El bot se invitó al canal de notificaciones.

**Gmail.** Se reutilizó la credencial OAuth2 ya validada en los módulos anteriores.

Los secretos de las apps (Client Secret, personal access key de la CLI) no forman parte del repositorio. El archivo de configuración local de la CLI, `hubspot.config.yml`, está excluido.

## Continuidad con el Módulo 3

La memoria de largo plazo se conserva completa, con una mejora: el `Session_ID` ahora es el `threadId` de Gmail, de modo que cada hilo de correo es una sesión aislada y una respuesta del keyuser en el mismo hilo recupera su contexto. `Nombre Usuario`, que en el Módulo 3 quedaba genérico porque el chat no exponía identidad, ahora se completa con el nombre del remitente.

Se incorporaron las correcciones detectadas en las pruebas del Módulo 3: el filtro de Airtable pasó a expresión sobre `Session_ID`, el IF de memoria evalúa el campo de negocio y no el ID interno, la salida de respaldo del Enrutador apunta a la respuesta de ayuda y no a la de fuera de alcance, y los primeros cinco intercambios acumulan una traza compacta para que la summarization tenga contenido real al activarse.

## Test de regresión

| # | Prueba | Resultado esperado |
|---|---|---|
| 1 | Correo con asunto `Automatic reply: vacaciones` | Termina en `Detener: Auto-reply`. Sin borrador, sin Slack. |
| 2 | Correo desde una casilla `no-reply@...` | Termina en `Detener: Auto-reply`. |
| 3 | Correo nuevo: *¿Por qué se atrasa el portal de proveedores?* | Fila nueva en Airtable. Contacto creado en HubSpot. Borrador en el hilo. Aviso en Slack. |
| 4 | Segundo correo del mismo remitente, en hilo nuevo | `Contacto Existe?` va por la rama verdadera: se actualiza, no se duplica. |
| 5 | Respuesta en el mismo hilo | `Existe Memoria?` va por la rama verdadera y el Router recibe el contexto previo. |
| 6 | Correo pidiendo mover recursos a un proyecto | Categoría DERIVACION y aviso de escalamiento en Slack. |
| 7 | Correo pidiendo sueldos del equipo | Categoría FUERA_DE_ALCANCE. Borrador con la negativa. |

Cada nodo se validó con *Execute step* antes de exportar.

## Limitaciones conocidas

El Gmail Trigger funciona por polling cada minuto, por lo que la latencia de respuesta es de hasta un minuto más el tiempo de procesamiento. El corte del historial citado reconoce los formatos de Gmail en español e inglés y el separador clásico de Outlook; otros clientes de correo pueden dejar pasar parte del historial, que igual queda truncado a 2000 caracteres. Si Slack falla, el nodo continúa para no perder la escritura de memoria; el borrador en Gmail, que es el control crítico, ya está creado en ese punto.
