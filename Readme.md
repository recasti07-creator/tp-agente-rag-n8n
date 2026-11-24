1. Caso de Negocio y Cálculo de ROI
Descripción del Problema y Solución
TechFix Computación es un taller de servicio técnico y venta de repuestos que recibe un alto volumen de consultas diarias por WhatsApp sobre precios de repuestos, compatibilidad de componentes, estado de reparaciones y políticas de garantía. 
Se implementó un Asistente Virtual Autónomo en n8n con arquitectura RAG, capaz de consultar en tiempo real el inventario y estado de tickets (Google Sheets) y los manuales/políticas del taller (Supabase Vector Store), procesando las respuestas mediante un modelo local con Ollama.
Cálculo de ROI (Retorno de la Inversión)
A. Costos de Implementación (Inversión Inicial)
Licencias de Software (n8n, Ollama, Supabase Free Tier):$0 USD (herramientas open-source y planes gratuitos).
Hardware (Servidor Local):$0 USD (se reutiliza una PC existente del taller con GPU dedicada para correr Ollama). (Nota: en caso de adquirir un equipo nuevo dedicado con RTX 3060/4060, la inversión única sería de $1.000 USD).
Costo Operativo Mensual:$10 USD/mes correspondientes al consumo eléctrico estimado del equipo local (sin costos por tokens de APIs externas).
Desarrollo y Puesta en Marcha: 30 horas de implementación, carga de documentos y pruebas. Valorando la hora técnica en $15 USD/hora, el costo equivalente de desarrollo es de $450 USD (inversión única).
 B. Ahorro Operativo (Antes vs. Después)
Antes del Agente: Un técnico dedicaba en promedio 10 minutos por consulta (buscando precios, revisando planillas de tickets y redactando respuestas). Con 20 consultas diarias, se invertían 200 minutos/día (3,3 horas diarias), equivalentes a 73 horas al mes (22 días hábiles).
Después del Agente: El bot resuelve automáticamente el 80% de las consultas en segundos. El técnico solo interviene en casos complejos, reduciendo el tiempo de atención a 20 minutos diarios (7,3 horas al mes).
Ahorro Neto Mensual: Se liberan 65,7 horas de trabajo técnico al mes. Valorando esa hora técnica en $15 USD, representa un ahorro operativo de $985 USD/mes (o $975 USD/mes des contando los $10 USD de electricidad).
C. Conclusión del ROI
Recupero de la inversión: Tomando el costo de desarrollo ($450 USD) frente al ahorro mensual ($975 USD), la inversión se recupera en menos de 2 semanas (14 días hábiles).
ROI a 3 meses:((2.925 - 450) / 450) * 100 = 550% de retorno, liberando además al personal técnico para enfocarse en reparaciones facturables.

2-Elección del Modelo de IA (Local con Ollama vs. Nube)
Para el procesamiento del lenguaje natural se optó por ejecución local mediante Ollama (utilizando el modelo  qwen2.5:7b) en lugar de APIs comerciales en la nube (como OpenAI o Anthropic) por tres motivos principales:
Costo operativo nulo por token: Permite escalar el volumen de consultas diarias por WhatsApp sin preocuparse por facturas variables ni límites de cuota.
Privacidad: La información sensible de clientes (nombres, números de ticket y estados de reparación) y las listas de precios internas se procesan íntegramente en el servidor del taller sin enviarse a terceros.
Estrategia de Vector Store y Embeddings (RAG)
Almacenamiento Vectorial (Supabase): Se utilizó Supabase en su capa gratuita para almacenar las políticas del local, tiempos de garantía y manuales técnicos. 
Se eligió por su facilidad de integración nativa con n8n, persistencia en la nube sin consumo extra de RAM local y rapidez en búsquedas por similitud semántica.
Embeddings Locales (Embeddings Ollama): Para transformar los documentos en vectores ( nomic-embed-text:latest), se utilizó el mismo motor local de Ollama, garantizando coherencia dimensional entre la fase de ingesta.

 3- ¿Por qué un AI Agent con RAG y no un nodo de IA simple?
Un nodo de LLM simple (completado de texto básico) no alcanza para resolver este caso de negocio por tres limitaciones críticas:
1. Conocimiento estático y alucinaciones: Un modelo base desconoce los precios actuales, el stock real del taller y las políticas internas de TechFix; sin RAG inventaría respuestas.
2. Necesidad de datos dinámicos en tiempo real: El stock y el estado de las reparaciones cambian minuto a minuto. Un AI Agent tiene la capacidad de razonar y decidir qué herramienta ejecutar según la intención del usuario (consultar tickets  en Sheets, buscar en Inventario  en Sheets o hacer búsqueda semántica en Supabase).
3. Memoria conversacional: Gracias al nodo Simple Memory, el agente mantiene el hilo de la conversación por sessionId (por ejemplo, si primero pide el número de ticket y el usuario lo responde en el siguiente mensaje).

4. Manejo de Excepciones y Control de Errores
Para garantizar que el cliente nunca quede sin respuesta ante un fallo técnico en producción, se implementaron dos puntos de control de errores en el flujo:
1. Control de fallos de ejecución en el AI Agent1 (On Error Output):
   Qué puede fallar: Caída del servidor local de Ollama, error de autenticación con Google Sheets o latencia/desconexión con Supabase Vector Store.
   Cómo se gestiona: Se configuró la salida de error del nodo AI Agent1 (Continue using error output) redirigiendo el flujo hacia el nodo Respond to Webhook1, el cual devuelve inmediatamente un mensaje de contingencia al usuario informando que el sistema está momentáneamente ocupado y derivando la atención a un técnico humano.
2.Validación lógica de respuesta mediante nodo If:
   Qué puede fallar: Respuestas vacías/nulas del modelo local o casos donde el agente determina que no posee la información suficiente y requiere intervención humana ([completar con tu condición exacta del nodo If, ej: detectar respuesta vacía o palabra clave de derivación).
Cómo se gestiona: El nodo If evalúa la salida generada por el agente antes de enviarla. Si cumple la condición de respuesta válida (true), responde normalmente vía Respond to Webhook; en caso contrario (false), enruta hacia Respond to Webhook1 para activar el flujo alternativo de derivación sin exponer errores crudos al cliente.
5. System Prompt del Agente
Sos el asistente virtual de atención al cliente y soporte técnico de "TechFix Computación". Tu objetivo es responder consultas de clientes por WhatsApp sobre estado de reparaciones, stock de repuestos, precios y manuales técnicos, usando únicamente la información de tus herramientas.
REGLAS DE OPERACIÓN Y HERRAMIENTAS:
1.ESTADO DE REPARACIONES (Google Sheets - tickets): 
   - Si el cliente consulta por el estado de su equipo o brinda un número de orden/ticket, utiliza la herramienta Consulta estado de reparación-soporte técnico.
   - Si no brindó el número de orden (id_ticket), pedilo cordialmente antes de buscar. Nunca inventes estados ni fechas de entrega.
2.STOCK, PRECIOS Y COMPATIBILIDAD (Google Sheets - Inventario):
   - Si consultan por disponibilidad de repuestos (pantallas, discos, memorias RAM, fuentes), precios o costos, utiliza la herramienta Inventario - stock disponible y precios.
   - Verifica siempre la columna modelo compatible antes de confirmar que un repuesto le sirve al equipo del cliente.
   - Si la columna Stock es “0” o está vacía, aclarale al cliente que actualmente no hay stock disponible.
3. MANUALES Y ASISTENCIA TÉCNICA (RAG - Supabase): 
   - Si consultan por procedimientos técnicos, políticas de garantía, consulta la base de conocimiento vectorial (Supabase Vector Store).

4. DERIVACIÓN A TÉCNICO HUMANO: 
   - Si el problema requiere revisión física urgente, no encontrás el ticket en la planilla o el cliente pide hablar con una persona, indicá claramente que derivás la consulta con un técnico humano en el taller.

 TONO Y FORMATO (OPTIMIZADO PARA WHATSAPP):
- Mantené un tono profesional, cercano y resolutivo (usando "vos").
- Sé conciso y claro: usa viñetas cortas y negritas para resaltar datos clave (precios, estados de reparación o modelos).
- Regla anti-alucinación: Si un dato no figura en ninguna herramienta, no lo supongas ni lo inventes.
 5. Evidencia de Funcionamiento.

en la carpeta de capturas están los archivos donde se ve la ejecución y el chat con las respuestas.

