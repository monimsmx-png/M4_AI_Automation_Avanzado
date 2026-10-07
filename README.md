# M4. Expense Automation V2 - SugarCRM Integration

## 1. Descripción

Este workflow implementa una solución de automatización para la gestión
y conciliación de gastos utilizando **n8n como orquestador**, **IA para
interpretar y procesar solicitudes**, **Airtable como memoria
persistente del caso** y **SugarCRM como sistema externo de gestión de
gastos**.

La solución parte de una solicitud del usuario, identifica la intención
de negocio y deriva la operación al Worker especializado
correspondiente. Cuando se trata de una conciliación, el resultado se
utiliza para consultar SugarCRM y determinar si existe un gasto
relacionado. Dependiendo del resultado, el registro se actualiza como
conciliado o se crea un nuevo gasto.

La integración con SugarCRM se realiza mediante su **API REST utilizando
nodos HTTP Request**.

## 2. Objetivo

El objetivo principal es automatizar el flujo de gestión de gastos,
incorporando:

- Clasificación de intención mediante IA.
- Arquitectura Manager-Worker.
- Memoria persistente asociada a `Session_ID`.
- Extracción y conciliación de movimientos.
- Integración con SugarCRM mediante API REST.
- Actualización o creación de registros en SugarCRM.
- Notificación del resultado mediante Slack.
- Generación de un borrador de correo para revisión humana.
- Manejo de respuestas automáticas / Out of Office.
- Ruta de escalación a una persona cuando la solicitud no puede
  clasificarse de forma concluyente.

## 3. Arquitectura general

``` text
Chat / solicitud
      |
      v
Memoria Airtable por Session_ID
      |
      v
AI Manager
      |
      +----------+----------+
      |          |          |
      v          v          v
CONCILIACION EXTRACCION  ESCALAR
      |          |          |
      v          v          v
 Worker 1    Worker 2   Revisión humana
      |
      v
Autenticación API SugarCRM
      |
      v
Buscar gasto existente
      |
   +--+------+
   |         |
 Existe   No existe
   |         |
   v         v
 Update    Create
   |
   v
 Slack
   |
   v
Borrador de correo
```

## 4. Manager y clasificación de intención

El Manager utiliza un AI Agent para determinar qué operación corresponde
a la solicitud.

La taxonomía cerrada utilizada por el flujo es:

- `CONCILIACION`: movimiento bancario que debe compararse con gastos
  registrados.
- `EXTRACCION`: extracción de movimientos a partir de un estado de
  cuenta.
- `ESCALAR`: solicitud ambigua, información insuficiente o situación que
  requiere intervención humana.

El resultado se valida mediante un **Structured Output Parser**. El
`Switch1` utiliza la intención para dirigir la ejecución al Worker
correspondiente o a la ruta de revisión humana.

## 5. Memoria persistente

La memoria del caso se almacena en Airtable, en la tabla `Memoria`, y se
recupera mediante `Session_ID`.

El flujo distingue entre:

- **Caso existente:** recupera el contexto y continúa la conversación.
- **Caso nuevo:** crea un registro de memoria.

La memoria mantiene información como:

- Estado del caso.
- Resumen consolidado.
- Datos clave.
- Número de intercambios.

Cuando el número de intercambios supera 5, el flujo utiliza un LLM para
consolidar el contexto y reducir información repetitiva.

El contexto recuperado se entrega al Manager delimitado entre:

``` text
[INICIO DE CONTEXTO COMPARTIDO]
...
[FIN DEL CONTEXTO COMPARTIDO]
```

## 6. Workers

### Worker 1 - Expense Matching Agent

Recibe:

- `fecha_operacion`
- `fecha_cargo`
- `descripcion`
- `monto`

Su responsabilidad es identificar correspondencias entre el movimiento y
los gastos registrados y devolver un resultado estructurado.

El Manager lo invoca mediante `Execute Workflow` con la opción de
esperar la finalización del subworkflow.

### Worker 2 - Statement Extraction Agent

Recibe:

- `documento_texto`

Su responsabilidad es identificar y estructurar movimientos bancarios de
un estado de cuenta.

También es invocado mediante `Execute Workflow` con espera de
finalización.

## 7. Integración con SugarCRM

La integración utiliza la **API REST de SugarCRM** mediante nodos
`HTTP Request`.

### Autenticación

Se realiza una petición:

``` text
POST /rest/v11/oauth2/token
```

para obtener un `access_token`.

El token se utiliza posteriormente mediante el header:

``` text
OAuth-Token
```

### Consulta

El flujo consulta el módulo:

``` text
EXP_Gastos
```

utilizando principalmente:

- Importe.
- Fecha del gasto.

Se recuperan campos como ID, nombre, importe, fecha, facturable, estado
y cuenta relacionada.

### Actualización

Cuando se encuentra un gasto correspondiente:

``` text
PUT /rest/v11/EXP_Gastos/{id}
```

y se actualiza:

``` json
{
  "estado_c": "Conciliado"
}
```

### Creación

Cuando no se encuentra un registro correspondiente:

``` text
POST /rest/v11/EXP_Gastos
```

y se crea un nuevo gasto con la información del movimiento.

## 8. Notificaciones y salida operativa

Cuando un gasto es conciliado, el flujo prepara un payload para Slack
con:

- Descripción.
- Importe.
- Fecha.
- Resultado.

También se genera un **borrador de correo**, no un envío automático. El
borrador queda pendiente de revisión humana antes de enviarse al
cliente.

Este diseño evita comunicaciones externas automáticas sin validación.

## 9. Manejo de respuestas automáticas

Un segundo flujo utiliza `Gmail Trigger` para identificar respuestas
automáticas.

Se contemplan patrones como:

- `automatic reply`
- `auto-reply`
- `autoreply`
- `out of office`
- `respuesta automática`
- `respuesta automatica`

Cuando se detecta una respuesta automática:

1.  No continúa el procesamiento normal.
2.  Se envía una notificación a Slack indicando que el cliente no está
    disponible.
3.  Se informa el asunto y remitente.
4.  El flujo termina para esta entrega.

En un escenario productivo podría evolucionarse hacia un reintento o
reenvío programado.

## 10. Escalación humana

Cuando el Manager no puede clasificar de forma concluyente una
solicitud, utiliza:

``` text
ESCALAR
```

El `Switch1` dirige la solicitud a `Send message and wait for response`,
permitiendo intervención humana antes de continuar.

## 11. Credenciales y configuración

Las credenciales se administran mediante el sistema de credenciales de
n8n.

Para ejecutar el workflow en otra instancia será necesario configurar:

- OpenAI.
- Airtable.
- Gmail.
- Slack.
- Credenciales HTTP para SugarCRM.

También deben validarse la URL y los parámetros de la instancia de
SugarCRM de destino.

**No se incluyen contraseñas, tokens ni secretos en este README.**

## 12. Requisitos de ejecución

Antes de ejecutar:

1.  Importar el JSON en n8n.
2.  Configurar las credenciales.
3.  Verificar los Workflows de los Workers referenciados.
4.  Verificar el acceso a Airtable.
5.  Verificar el endpoint de SugarCRM.
6.  Verificar el módulo `EXP_Gastos`.
7.  Ejecutar una prueba con un gasto existente.
8.  Ejecutar una prueba con un gasto inexistente.
9.  Validar Slack.
10. Validar el borrador de Gmail.
11. Validar Out of Office.
12. Validar escalación humana.

## 13. Escenarios de prueba

### Prueba 1 - Gasto existente

``` text
Movimiento
  -> Worker de conciliación
  -> Consulta SugarCRM
  -> Gasto encontrado
  -> Estado = Conciliado
  -> Notificación
  -> Borrador
```

### Prueba 2 - Gasto no existente

``` text
Movimiento
  -> Worker de conciliación
  -> Consulta SugarCRM
  -> Sin coincidencia
  -> Crear Gasto
```

### Prueba 3 - Estado de cuenta

``` text
Solicitud
  -> Manager
  -> EXTRACCION
  -> Worker 2
  -> Movimientos estructurados
```

### Prueba 4 - Respuesta automática

``` text
Gmail Trigger
  -> Filtro de respuesta automática
  -> Notificación Slack
  -> Fin del flujo
```

### Prueba 5 - Solicitud ambigua

``` text
Manager
  -> ESCALAR
  -> Revisión humana
```

## 14. Evidencias recomendadas para la entrega

El PDF de evidencias puede incluir:

- Canvas general del workflow.
- Clasificación de intención.
- Memoria persistente.
- Configuración de `Execute Workflow`.
- Resultado de los Workers.
- Consulta a SugarCRM.
- Actualización de un gasto existente.
- Creación de un gasto nuevo.
- Registro actualizado en SugarCRM.
- Notificación en Slack.
- Borrador generado en Gmail.
- Manejo de Out of Office.
- Ruta de escalación humana.

## 15. Alcance y posibles evoluciones

La solución está planteada como una demostración funcional de
automatización agéntica e integración empresarial.

En un escenario productivo podrían incorporarse posteriormente:

- Reintentos automáticos.
- Manejo más granular de errores de API.
- Control de duplicados más robusto.
- Reenvíos programados para clientes no disponibles.
- Auditoría centralizada.
- Validaciones adicionales antes de actualizar o crear registros.
- Gestión de permisos y usuarios de SugarCRM.
- Procesamiento por lotes de múltiples movimientos.
- Monitoreo y alertas operativas.

Estas mejoras quedan fuera del alcance de la implementación actual.
