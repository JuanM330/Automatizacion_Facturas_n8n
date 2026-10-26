# Automatizacion_Facturas_n8n

# Procesamiento automatizado de facturas con n8n

Proyecto del curso **Automatización inteligente de procesos con n8n** (EAM, 2026 II, segundo corte).

## Nombre, descripción e integrantes

**Problema.** Las facturas llegan por correo, papel o desde el sistema de un proveedor y se revisan a mano: es lento, permite errores de digitación y facturas registradas dos veces.

**Solución.** Un conjunto de workflows de n8n que recibe una factura (formulario o Webhook), la valida, limpia sus datos, evita duplicados, la guarda en Baserow y responde al emisor según lo que realmente ocurrió. Si falla un servicio técnico, deja un incidente registrado y avisa por Discord.

**Integrantes**
- Juan Esteban Muñoz Díaz


## Alcance actual

**Disponible**
- Entradas: formulario de n8n, ejecución manual de prueba y Webhook `POST` autenticado.
- Validación de campos obligatorios (`numero_factura`, `cliente`, `monto`, `estado`) en el Webhook.
- Control de duplicados contra Baserow en el Webhook.
- Subworkflow que limpia estado y cliente.
- Guardado en Baserow con HTTP Request.
- Manejo de errores con respuesta 502, ejecución fallida, tabla de incidentes y aviso a Discord.

**Limitaciones principales**
- Dos envíos simultáneos de la misma factura podrían crear dos filas (falta unicidad en la base de datos).
- El formulario y la ejecución manual no validan ni revisan duplicados.
- No hay reintentos automáticos, ni lectura de facturas desde archivos, ni túnel público.

El detalle de lo implementado, lo simulado y lo propuesto está en la [bitácora unificada](documentacion/Bitacora_Unificada_Proyecto.md#8-estado-final).

## Arquitectura
`
```mermaid
flowchart LR
    W["Webhook POST"] --> V{"¿Clave y campos válidos?"}
    V -- No --> X["403 / 400"]
    V -- Sí --> D{"¿Factura nueva?"}
    D -- No --> Y["409"]
    D -- Sí --> S["Subworkflow: preparar factura"]
    F["Formulario / prueba manual"] --> S
    S --> E{"¿Completa?"}
    E --> G["Guardar en Baserow"]
    G -- Éxito --> OK["200"]
    G -- Error --> FL["502 + ejecución fallida"]
    FL --> EH["Manejador de errores"]
    EH --> I["Tabla Incidentes + aviso Discord"]
````

| Workflow | Archivo | Responsabilidad |
|---|---|---|
| `facturas-21/10` (principal) | [workflows/Principal_Facturas.json](workflows/Principal_Facturas.json) | Recibe, valida, evita duplicados, decide la ruta, guarda en Baserow y responde |
| `Subworkflow - Preparar Factura` | [workflows/Subworkflow_Preparar_Factura.json](workflows/Subworkflow_Preparar_Factura.json) | Limpia `estado` y `cliente`; devuelve `estado_limpio` y `cliente_limpio` |
| `Manejador de errores facturas` | [workflows/Manejador_Errores_Facturas.json](workflows/Manejador_Errores_Facturas.json) | Registra el incidente en Baserow y avisa por Discord |

## Requisitos

- **n8n:** [COMPLETAR: versión utilizada], ejecutado localmente.
- **Baserow:** cuenta con un workspace y permiso para crear un token de base de datos.
- **Discord:** un canal con un webhook creado (para los avisos).
- `curl` para probar el Webhook.

## Configuración

### 1. Baserow
1. Cree una base de datos con dos tablas.
2. **`Facturas`**: columnas `numero_factura`, `cliente`, `monto`, `estado`, `detalle`.
3. **`Incidentes`**: columnas `fecha`, `workflow`, `nodo`, `mensaje`, `ejecucion`, `estado_atencion`. Los nombres deben coincidir exactamente.
4. Anote el número de cada tabla (aparece en la URL de la tabla).
5. En *Settings → Database tokens* cree un token con permisos de **crear, leer y listar** sobre ese workspace.

### 2. Credenciales en n8n
Cree estas tres credenciales con **sus propios valores**. No las comparta ni las escriba dentro de los workflows.

| Credencial | Tipo | Configuración |
|---|---|---|
| Baserow | Header Auth | Name: `Authorization`; Value: `Token ` + su token de Baserow |
| Clave del Webhook | Header Auth | Name: `x-api-key` (o el que prefiera); Value: una clave propia |
| Discord | Discord Webhook | URL del webhook de su canal |

## Importación y conexiones

Importe en este orden (*Workflows → Import from file*):

1. **Subworkflow** `Subworkflow_Preparar_Factura.json` → **publíquelo**. Si no está publicado, el principal no se podrá publicar.
2. **Manejador** `Manejador_Errores_Facturas.json`:
   - En el nodo *Guardar Incidente en Baserow*: seleccione su credencial de Baserow y cambie el número de tabla de la URL por el de su tabla `Incidentes`.
   - En el nodo *Discord*: seleccione su credencial.
   - Publíquelo.
3. **Principal** `Principal_Facturas.json`:
   - En *Webhook*: seleccione su credencial de clave.
   - En *Guardar en Baserow* y *Buscar Duplicado*: seleccione su credencial de Baserow y cambie el número de tabla de la URL por el de su tabla `Facturas`.
   - En *Llamar preparar factura*: seleccione el subworkflow importado.
   - En *Settings → Error Workflow*: seleccione el manejador importado.
   - Publíquelo.

> Los IDs de credenciales, workflows y tablas que traen los JSON pertenecen a la instancia original. Hay que seleccionarlos de nuevo después de importar.

## Ejecución

**Funcionamiento habitual:** con el principal publicado, envíe las facturas por `POST` a
`http://localhost:5678/webhook/factura-c08` con la clave en el encabezado. Los resultados se ven en la tabla `Facturas` de Baserow y en *Executions* de n8n.

**Pruebas desde el editor:** use la URL `/webhook-test/factura-c08` y pulse *Listen for test event* en el nodo Webhook. Esa URL solo responde mientras el editor escucha. La de producción (`/webhook/`) solo responde con el workflow publicado.

**Formulario:** abra la URL del *Form Trigger* desde el editor y llene los 4 campos.

**Al terminar las pruebas:** despublique el principal para que el Webhook deje de recibir solicitudes. El proyecto no usa programación automática.

## Prueba de ejemplo

Datos ficticios. Reemplace `<SU_CLAVE>` y, si cambió el nombre del encabezado, `x-api-key`.

```bash
curl -i -X POST http://localhost:5678/webhook/factura-c08 \
  -H "Content-Type: application/json" \
  -H "x-api-key: <SU_CLAVE>" \
  -d '{"numero_factura":"C08-001","cliente":"Cliente Ficticio","monto":150000,"estado":" Completa "}'
```

En CMD de Windows:

```bat
curl -i -X POST http://localhost:5678/webhook/factura-c08 -H "Content-Type: application/json" -H "x-api-key: <SU_CLAVE>" -d "{\"numero_factura\":\"C08-001\",\"cliente\":\"Cliente Ficticio\",\"monto\":150000,\"estado\":\" Completa \"}"
```

| Resultado esperado | Dónde comprobarlo |
|---|---|
| Respuesta `200` con `{"ok":true,"mensaje":"Factura procesada y guardada en Baserow"}` | Salida de `curl` |
| Una fila nueva en `Facturas` con `cliente` y `estado` ya limpios (`completa`) | Tabla en Baserow |
| Al repetir el mismo comando: `409` y ninguna fila nueva | Salida de `curl` y Baserow |
| Sin el encabezado de clave: `403` | Salida de `curl` |
| Sin el campo `cliente`: `400` | Salida de `curl` |

Otras pruebas (fallo técnico, incidentes y recuperación) están descritas en la bitácora.

## Documentación y evidencias

- [Bitácora unificada](documentacion/Bitacora_Unificada_Proyecto.md): funcionamiento, integraciones, errores, pruebas y estado final.
- [Presentación en PDF](documentacion/Presentacion_Corte_2.pdf)
- Diagramas: [proceso manual](documentacion/diagramas/Proceso_Manual.png) y [proceso automatizado](documentacion/diagramas/Proceso_Automatizado.png)
- Evidencias de las pruebas: [documentacion/evidencias/](documentacion/evidencias/)
- Bitácoras por clase (historial): [documentacion/historial/](documentacion/historial/)

## Estructura del repositorio

```text
├── README.md
├── workflows/
│   ├── Principal_Facturas.json
│   ├── Subworkflow_Preparar_Factura.json
│   └── Manejador_Errores_Facturas.json
└── documentacion/
    ├── Bitacora_Unificada_Proyecto.md
    ├── Presentacion_Corte_2.pdf
    ├── diagramas/
    ├── evidencias/
    └── historial/
```
