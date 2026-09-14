# Propuesta de integración DOF → IA → ULPIANO

## 1. Objetivo y alcance

Integrar el monitoreo del **Diario Oficial de la Federación (DOF)** con **ULPIANO** para convertir publicaciones oficiales en información regulatoria estructurada: qué cambió, a quién afecta, qué impacto tiene y qué acciones conviene considerar.

La propuesta aprovecha el prototipo existente (POC) y plantea un proceso automatizado que extraiga publicaciones, las analice mediante inteligencia artificial y almacene los resultados en Supabase para su consulta personalizada desde ULPIANO.

Incluye el flujo técnico, la adaptación del prototipo, los datos de salida, los puntos de integración y la infraestructura inicial. Quedan fuera de esta etapa la implementación completa, el dimensionamiento definitivo y las modificaciones al frontend o a la base de datos de ULPIANO.

## 2. Flujo y responsabilidades

```text
DOF → Extracción → Normalización y deduplicación → Análisis con IA
    → Validación → Supabase → Consulta personalizada en ULPIANO
```

| Etapa | Función | Responsable |
| --- | --- | --- |
| Detección y extracción | Consultar las ediciones matutina y vespertina; recuperar título, dependencia, fecha, contenido y enlace oficial. | Automatización |
| Preparación | Unificar el formato, limpiar el contenido y comprobar si la publicación ya fue procesada. | Automatización |
| Análisis | Resumir, identificar cambios, clasificar temas y perfiles, proponer impacto y acciones, y extraer fechas relevantes. | IA |
| Validación | Comprobar estructura, campos obligatorios, tipos, catálogos y consistencia con la fuente. | Automatización; revisión humana según reglas pendientes de definir |
| Persistencia | Guardar registros válidos y registrar estados, errores y reintentos. | Worker autorizado |
| Personalización | Seleccionar publicaciones según organización, sector, rol, perfil y preferencias del usuario. | ULPIANO |

**La automatización controla el proceso; la inteligencia artificial interpreta el contenido.** Las operaciones deterministas se resuelven mediante código tradicional: la IA no administra credenciales, decide si un registro ya fue procesado, escribe directamente en Supabase ni controla reintentos o estados del pipeline. El worker ejecuta el procesamiento dentro de ese flujo.

## 3. Datos de salida y validación

Se propone **JSON como formato de intercambio**. Los siguientes campos constituyen un **contrato lógico propuesto**, distinto del esquema real de base de datos, que deberá revisarse con el equipo de ULPIANO. Se distinguen tres tipos de datos: extraídos del DOF, interpretados por IA y metadatos operativos del pipeline.

| Grupo | Campos propuestos | Origen |
| --- | --- | --- |
| Identificación | `source`, `source_id`, `publication_date`, `edition`, `title`, `agency`, `source_url` | Datos del DOF; el código asigna `source` y conserva o deriva `source_id` |
| Contenido fuente | `raw_content` | Texto extraído del DOF y normalizado para el análisis |
| Análisis | `summary`, `what_changed`, `why_it_matters`, `topic`, `key_points` | IA |
| Impacto y acciones | `impact_level`, `requires_action`, `effective_date`, `recommended_actions` | IA, con validación |
| Relevancia | `affected_profiles`, `affected_sectors`, `regulatory_topics`, `relevance_reason` | IA, alineada con los catálogos de ULPIANO |
| Trazabilidad | `processing_status`, `processed_at`, `ai_provider`, `ai_model`, `validation_status`, `error_message` | Metadatos operativos generados por el pipeline |

### Reglas mínimas

- Exigir identificador único, fecha de publicación, título y URL oficial válida; incluir la dependencia cuando la fuente la proporcione.
- Validar tipos: `requires_action` como booleano; puntos clave, acciones, perfiles, sectores y tópicos adicionales como listas.
- Limitar `impact_level` a `high`, `medium` o `low`, con criterios definidos por negocio.
- Usar identificadores de los catálogos compartidos de perfiles y tópicos.
- Validar las fechas y contrastar `effective_date` con la publicación original. Si la entrada en vigor no está explícita, usar `null`.
- Contrastar las afirmaciones generadas por IA con el contenido fuente; validar el JSON no garantiza su exactitud. Las interpretaciones y recomendaciones deben poder revisarse contra esa evidencia.
- Evitar duplicados y conservar la fuente, el modelo utilizado y el resultado de la validación para poder revisar el procesamiento.

Los estados iniciales propuestos son `detected`, `extracted`, `processing`, `processed`, `validation_failed`, `pending_review`, `published` y `error`. Su uso y transiciones definitivos deberán acordarse con el equipo de ULPIANO; `processed` no implica `published`.

**Un resultado incompleto o inválido no debe publicarse automáticamente.** Se propone reintentar errores temporales con límites por definir; reintentar o enviar a `pending_review` las respuestas inválidas del LLM; omitir duplicados ya procesados; y registrar y escalar fallos permanentes de la fuente sin reintentos indefinidos.

## 4. Integración con Supabase y ULPIANO

Se propone mantener un registro por publicación, vinculado con los perfiles, sectores y tópicos correspondientes. La deduplicación evitará repetir análisis por usuario, sin impedir reintentos de fallos o reprocesamientos controlados. ULPIANO determinará su relevancia para cada usuario sin repetir el análisis de IA.

| Componente | Responsabilidad |
| --- | --- |
| Worker de procesamiento | Generar y validar el JSON, evitar duplicados, insertar o actualizar registros y registrar el resultado. |
| Supabase | Almacenar publicaciones y relaciones, respetando organizaciones, membresías, roles y políticas de seguridad por fila (RLS). |
| ULPIANO | Consultar los datos y construir el feed personalizado según el contexto y las preferencias del usuario. |

La escritura se realizará desde el worker autorizado. Las credenciales administrativas de Supabase permanecerán del lado servidor. La integración deberá respetar RLS, organizaciones y la separación de datos del modelo multitenant existente; los permisos efectivos del worker se validarán con backend.

Antes de implementarla, se revisarán las tablas disponibles, relaciones, identificadores, campos obligatorios, catálogos, funciones o Edge Functions aplicables y permisos del worker. **El JSON propuesto no presupone que esos campos ya existan en Supabase.**

## 5. Infraestructura inicial

Se propone un **worker en AWS EC2**, independiente de la aplicación web. Ejecutará la extracción, las llamadas a la API de IA, la validación y la persistencia. El análisis del modelo ocurrirá en el proveedor externo, por lo que esta arquitectura no requiere GPU.

| Recurso | Propuesta para el piloto |
| --- | --- |
| Instancia | EC2 `t3.xlarge`, arquitectura x86_64 |
| Sistema operativo | Ubuntu Server 24.04 LTS |
| Capacidad | 4 vCPU y 16 GB de RAM |
| Almacenamiento | 80 GB EBS gp3 |
| Software | Git, Node.js, npm, Python 3 y pip; Chromium Headless cuando sea necesario |
| Conectividad | Salida HTTPS hacia DOF, Supabase y el proveedor de IA |

Esta configuración es un **dimensionamiento inicial para el piloto**, no un requerimiento definitivo: `t3.xlarge` es una opción propuesta. Todavía no se ha realizado una prueba de carga completa; el tamaño se ajustará con métricas reales de CPU, memoria, disco, duración y volumen de publicaciones.

### Operación

- **Ejecución:** programar corridas matutinas y vespertinas; seleccionar el scheduler durante la implementación.
- **Seguridad:** limitar el acceso administrativo, proporcionar API keys y credenciales al worker mediante variables de entorno o el mecanismo aprobado por infraestructura; nunca almacenarlas en Git.
- **Entorno:** fijar versiones de dependencias para reproducir el despliegue.
- **Almacenamiento:** usar el disco local para dependencias, logs, caché y archivos temporales, con políticas de limpieza. Los datos regulatorios persistentes se conservarán en Supabase u otros servicios definidos por el proyecto.
- **Monitoreo:** registrar inicio y fin de corrida, edición, publicaciones detectadas y procesadas, éxitos, errores, casos en revisión, duración y consumo de IA; habilitar alertas ante fallos.
- **Escalabilidad:** ajustar recursos o añadir workers según las métricas, manteniendo el procesamiento independiente de la aplicación web.

## 6. Adaptación del prototipo

El POC está orientado a generar reportes y distribuir documentos. La adaptación centrará su salida en datos estructurados para ULPIANO.

| Tratamiento | Componentes |
| --- | --- |
| Reutilizar como base | Consulta y extracción del DOF; lógica y conocimiento del análisis regulatorio: resumen, clasificación, perfiles afectados, impacto, fechas, puntos clave y recomendaciones. |
| Adaptar | Detección de ediciones, prompts para salida JSON, catálogos de perfiles y tópicos, criterios de impacto, validaciones y métricas. |
| Desarrollar o formalizar | Scheduler, estados persistentes, deduplicación, validación por esquema, integración con Supabase, reintentos, despliegue, secretos, monitoreo, alertas y pruebas de carga. |
| Definir con el equipo | Necesidad de una cola como alternativa futura y flujo de revisión editorial. |
| Separar del flujo principal | Generación de HTML/PDF y distribución por WhatsApp. Podrán funcionar después como canales que consuman publicaciones ya procesadas. |

Según lo documentado del POC, la identificación de ediciones se ha probado conceptualmente y las validaciones solo de forma parcial. Su reutilización requiere adaptación y pruebas antes de producción.

Los perfiles iniciales propuestos son agente aduanal, especialista en comercio exterior, abogado corporativo y responsable de compliance. Deberán alinearse con el catálogo de ULPIANO, al igual que los tópicos, sin mantener catálogos independientes.

## 7. Decisiones pendientes y responsables

| Decisión | Definición necesaria | Área sugerida |
| --- | --- | --- |
| Modelo de datos e integración | Tablas, relaciones, contrato JSON, estados, permisos y políticas RLS. | Equipo ULPIANO / Backend |
| Catálogos e impacto | Perfiles, tópicos, identificadores compartidos y criterios para impacto alto, medio o bajo. | Producto / Negocio |
| Publicación y revisión | Si toda salida válida se publica automáticamente, si el impacto alto requiere revisión y qué errores o inconsistencias pasan a `pending_review`. | Producto / Operación editorial |
| Proveedor de IA | Modelo inicial y alternativa ante indisponibilidad, falta de saldo, cuotas o cambios del servicio. | Equipo de procesamiento |
| Ejecución y reintentos | Scheduler, necesidad de cola y tratamiento de errores temporales, respuestas inválidas y fallos permanentes. | Procesamiento / Infraestructura |
| Despliegue y operación | Accesos, secretos, monitoreo, alertas y ajuste de capacidad. | Infraestructura |

Los responsables específicos se asignarán durante la planeación. La propuesta de infraestructura y la integración deberán validarse antes de la implementación definitiva.

## 8. Siguiente paso: piloto de extremo a extremo

1. Validar la propuesta y revisar el esquema actual de Supabase con los equipos técnico y de producto.
2. Acordar el contrato JSON, los catálogos, los estados y las reglas de validación y publicación.
3. Completar primero una prueba con **una publicación real**: DOF → extracción → análisis IA → JSON → validación → Supabase → ULPIANO. La automatización de todo el DOF queda para después de validar esta cadena.
4. Incorporar control persistente de estados, deduplicación y reintentos; automatizar las corridas.
5. Desplegar el worker con secretos, logs, métricas y alertas; ejecutar pruebas de carga y ajustar la infraestructura.

**Criterio de éxito del primer piloto:** una publicación oficial recorre toda la cadena y se muestra correctamente en ULPIANO, con clasificación, perfiles afectados, impacto, fechas disponibles y acciones recomendadas, sin correcciones manuales sobre los datos.


## Anexo A — Ejemplo de contrato JSON

**Ejemplo ficticio, no apto para publicación.** El contenido, la dependencia, los identificadores y los catálogos son ilustrativos. La URL usa un dominio reservado de ejemplo: en operación debe apuntar a la publicación oficial del DOF. Proveedor y modelo son marcadores pendientes de selección. Este contrato lógico no afirma que existan estas columnas en Supabase; `approved` ilustra una validación técnica, no una aprobación editorial.

```json
{
  "source": "DOF",
  "source_id": "ejemplo-dof-2026-09-14-001",
  "publication_date": "2026-09-14",
  "edition": "matutina",
  "title": "EJEMPLO FICTICIO: Acuerdo que actualiza el formato de un informe de operaciones",
  "agency": "Organismo de Comercio de Ejemplo (ficticio)",
  "source_url": "https://example.invalid/dof/publicacion-001",
  "raw_content": "Texto ficticio: Las empresas importadoras sujetas al informe de operaciones deberán utilizar el formato actualizado, que incorpora un campo de referencia interna. El acuerdo entrará en vigor el 1 de octubre de 2026.",
  "summary": "Se actualiza el formato del informe de operaciones para las empresas importadoras sujetas a su presentación.",
  "what_changed": "Se incorpora un campo de referencia interna al formato.",
  "why_it_matters": "Las empresas sujetas al informe deberán adaptar su preparación al nuevo formato.",
  "topic": "comercio_exterior",
  "impact_level": "medium",
  "requires_action": true,
  "effective_date": "2026-10-01",
  "key_points": [
    "El formato incorpora un campo de referencia interna.",
    "La entrada en vigor indicada es el 1 de octubre de 2026."
  ],
  "recommended_actions": [
    "Confirmar si la organización está sujeta al informe.",
    "Si aplica, actualizar la plantilla y revisar el procedimiento antes de la entrada en vigor."
  ],
  "affected_profiles": ["especialista_comercio_exterior", "responsable_compliance"],
  "affected_sectors": ["importacion"],
  "regulatory_topics": ["comercio_exterior"],
  "relevance_reason": "El cambio afecta la preparación del informe de las empresas importadoras sujetas a esa obligación.",
  "processing_status": "processed",
  "processed_at": "2026-09-14T08:32:00-06:00",
  "ai_provider": "proveedor_por_definir",
  "ai_model": "modelo_por_definir",
  "validation_status": "approved",
  "error_message": null
}
```
