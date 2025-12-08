# Guía de Integración - Sistema de Webhooks

## Tabla de Contenidos
1. [Resumen Ejecutivo](#resumen-ejecutivo)
2. [Arquitectura del Sistema](#arquitectura-del-sistema)
3. [Endpoint de Subida de Documentos](#endpoint-de-subida-de-documentos)
4. [Webhook 1: Notificación de Errores](#webhook-1-notificación-de-errores)
5. [Webhook 2: Resultado Final](#webhook-2-resultado-final)
6. [Troubleshooting](#troubleshooting)

---

## Resumen Ejecutivo

El sistema de extracción de documentos incluye un flujo completo de webhooks para integración con sistemas externos. Esta guía describe los 3 componentes principales que deben implementar:

### 1. Endpoint para enviar (cliente → sistema)
- Subir documentos para procesamiento

### 2. Webhooks para recibir (sistema → cliente)
- Webhook de Errores: Cuando falla una validación
- Webhook de Éxito: Cuando termina el procesamiento

---

## Arquitectura del Sistema

```
┌─────────────────────────────────────────────────────────────────────┐
│                         FLUJO COMPLETO                               │
└─────────────────────────────────────────────────────────────────────┘

    ┌──────────────┐
    │   CLIENTE    │
    │   (Ustedes)  │
    └──────┬───────┘
           │
           │ 1. POST /clients/:id/records
           │    {documents: [doc1, doc2, ...]}
           ↓
    ┌──────────────┐
    │  EXTRACTOR   │
    │   (Nuestro   │
    │    Sistema)  │
    └──────┬───────┘
           │
           │ 2. Procesamiento
           │    ├─ Extracción de datos
           │    ├─ Validaciones (reglas)
           │    └─ AI Transformations
           │
           ├────────┬────────┐
           │        │        │
      ¿Éxito?     NO       SÍ
           │        │        │
           │        ↓        ↓
           │   ┌─────────┐  ┌─────────┐
           │   │ Webhook │  │ Webhook │
           │   │ ERROR   │  │ SUCCESS │
           │   └────┬────┘  └────┬────┘
           │        │            │
           ↓        ↓            ↓
    ┌──────────────────────────────┐
    │    ENDPOINTS DEL CLIENTE     │
    │  POST /webhooks/error        │
    │  POST /webhooks/success      │
    └──────────────────────────────┘
```

---

## Endpoint de Subida de Documentos

### Endpoint Unificado

El sistema ofrece un endpoint único para crear records y subir documentos en una sola llamada atómica. El endpoint acepta dos formatos: `multipart/form-data` (recomendado) o `application/json` con archivos en base64.

#### **POST /clients/{client_id}/records**

**URL:** `https://api.extractor.com/clients/{client_id}/records`

---

### Opción 1: Multipart Form-Data (Recomendado)

**Headers:**
```http
X-API-Key: su-api-key-aqui
X-API-Secret: su-api-secret-aqui
Content-Type: multipart/form-data
```

**Límites:**
- Tamaño máximo total del request: 500MB
- Timeout: 2 minutos para completar la subida

**Form Fields:**
- `configuration_ref` (string, requerido): Referencia de la configuración
- `flow_id` (string, opcional): ID del flujo
- `rules_params` (string JSON, opcional): Parámetros adicionales de reglas. **Nota:** Las reglas ya están configuradas por el equipo de Nebuia, este campo solo se usa si se necesitan parámetros dinámicos adicionales.
- `files` (array de archivos, requerido): Archivos PDF a procesar
- `document_types` (array de strings, requerido): Tipo de cada documento (debe coincidir en cantidad con los archivos)

**Ejemplo con cURL:**
```bash
curl -X POST "https://api.extractor.com/clients/{client_id}/records" \
  -H "X-API-Key: su-api-key" \
  -H "X-API-Secret: su-api-secret" \
  -F "configuration_ref=dictaminacion" \
  -F "flow_id=flow-uuid-opcional" \
  -F 'rules_params={"razon_social":"ACME Corporation","rfc":"ACM950615ABC"}' \
  -F "files=@acta_constitutiva.pdf" \
  -F "document_types=acta_constitutiva" \
  -F "files=@asamblea1.pdf" \
  -F "document_types=acta_asamblea" \
  -F "files=@poder.pdf" \
  -F "document_types=poder_notarial"
```

---

### Opción 2: JSON con Base64

**Headers:**
```http
X-API-Key: su-api-key-aqui
X-API-Secret: su-api-secret-aqui
Content-Type: application/json
```

**Request Body (Ejemplo Simple - 1 Documento):**

```json
{
  "configuration_ref": "dictaminacion",
  "flow_id": "flow-uuid-opcional",
  "documents": [
    {
      "document_type": "acta_constitutiva",
      "file": "JVBERi0xLjQKJeLjz9MKMyAwIG9iago8PC9UeXBlIC9DYXRhbG9nCi9QYWdlcyAyIDAgUgo+PgplbmRvYmoKMiAwIG9iago8PC9UeXBlIC9QYWdlcwovS2lkcyBbMyAwIFJdCi9Db3VudCAxCj4+CmVuZG9iagozIDAgb2JqCjw8L1R5cGUgL1BhZ2UKL1BhcmVudCAyIDAgUgovTWVkaWFCb3ggWzAgMCA2MTIgNzkyXQovQ29udGVudHMgNCAwIFIKL1Jlc291cmNlcyA8PAovRm9udCA8PAovRjEgPDwKL1R5cGUgL0ZvbnQKL1N1YnR5cGUgL1R5cGUxCi9CYXNlRm9udCAvSGVsdmV0aWNhCj4+Cj4+Cj4+Cj4+CmVuZG9iago0IDAgb2JqCjw8L0xlbmd0aCA0ND4+CnN0cmVhbQpCVAovRjEgMTIgVGYKNzIgNzIwIFRkCihBY3RhIENvbnN0aXR1dGl2YSBkZSBBQ01FIENvcnBvcmF0aW9uKSBUagpFVAplbmRzdHJlYW0KZW5kb2JqCnhyZWYKMCA1CjAwMDAwMDAwMDAgNjU1MzUgZiAKMDAwMDAwMDAxOCAwMDAwMCBuIAowMDAwMDAwMDc3IDAwMDAwIG4gCjAwMDAwMDAxMzQgMDAwMDAgbiAKMDAwMDAwMDMxNyAwMDAwMCBuIAp0cmFpbGVyCjw8L1NpemUgNQovUm9vdCAxIDAgUgo+PgpzdGFydHhyZWYKNDEwCiUlRU9G",
      "file_name": "acta_constitutiva.pdf"
    }
  ],
  "rules_params": {
    "razon_social": "ACME Corporation S.A. de C.V.",
    "rfc": "ACM950615ABC"
  }
}
```

**Nota:** Los archivos se envían en base64 (sin el prefijo `data:application/pdf;base64,`)

**Request Body (Ejemplo Completo - Múltiples Documentos):**

```json
{
  "configuration_ref": "dictaminacion",
  "flow_id": "flow-uuid-opcional",
  "documents": [
    {
      "document_type": "acta_constitutiva",
      "file": "JVBERi0xLjQK...",
      "file_name": "acta_constitutiva.pdf"
    },
    {
      "document_type": "acta_asamblea",
      "file": "JVBERi0xLjQK...",
      "file_name": "asamblea1.pdf"
    },
    {
      "document_type": "acta_asamblea",
      "file": "JVBERi0xLjQK...",
      "file_name": "asamblea2.pdf"
    },
    {
      "document_type": "poder_notarial",
      "file": "JVBERi0xLjQK...",
      "file_name": "poder.pdf"
    }
  ],
  "rules_params": {
    "razon_social": "ACME Corporation S.A. de C.V.",
    "rfc": "ACM950615ABC"
  }
}
```

**Nota:** Los archivos se envían en base64 (sin el prefijo `data:application/pdf;base64,`)

---

### Response Exitosa:

```json
{
  "record_id": "550e8400-e29b-41d4-a716-446655440000",
  "job_id": "job-6ba7b810-9dad-11d1-80b4-00c04fd430c8",
  "status": "processing",
  "message": "Record created and processing started",
  "documents_uploaded": 4,
  "documents_by_type": {
    "acta_constitutiva": 1,
    "acta_asamblea": 2,
    "poder_notarial": 1
  }
}
```

### **Errores de Validación:**

```json
// Error: Falta documento requerido
{
  "error": "Required document type acta_constitutiva is missing (min_count: 1)"
}

// Error: Excede máximo
{
  "error": "Document type acta_asamblea allows maximum 10 documents, but 15 provided"
}

// Error: No cumple mínimo
{
  "error": "Document type acta_asamblea requires at least 3 documents, but 2 provided"
}

// Error: Tipo no permitido
{
  "error": "Document type pasaporte not allowed"
}
```

---

## Webhook 1: Notificación de Errores

### Cuándo se dispara:
- Una regla de validación detecta un problema
- El documento no cumple con los criterios configurados
- Fallo en validación de tipo de documento o información requerida

### Endpoint que deben crear:

```
POST https://su-sistema.com/webhooks/extractor/error
```

### Headers que recibirán:

```http
Content-Type: application/json
Authorization: Bearer su-token-secreto
```

### Body Example:

```json
{
  "event": "rule_execution_failed",
  "timestamp": "2025-01-15T10:30:45Z",
  "record": {
    "id": "rec-uuid-123",
    "client_id": "client-uuid-456",
    "configuration_ref": "dictaminacion",
    "flow_id": "flow-uuid-789",
    "status": "rejected",
    "created_at": "2025-01-15T10:30:00Z"
  },
  "rule": {
    "id": "rule-uuid-abc",
    "name": "Validar tipo de documento: Acta Constitutiva",
    "description": "Verificar que el documento sea un acta constitutiva válida",
    "processing_type": "pre_processing",
    "action": "reject",
    "severity": "high",
    "triggered": true,
    "reason": "El documento proporcionado no es un acta constitutiva. Se detectó que es un acta de asamblea."
  },
  "document": {
    "id": "doc-uuid-xyz"
  },
  "flow": {
    "id": "flow-uuid-789"
  },
  "fields_affected": [],
  "modifications": {},
  "webhook": {
    "triggered_at": "2025-01-15T10:30:45Z"
  }
}
```

### Response esperada:

```json
{
  "received": true,
  "message": "Error procesado correctamente"
}
```

**Status Code:** `200-299` (cualquier código 2xx se considera éxito)

### Diagrama de Flujo - Error:

```
┌─────────────────┐
│  Procesamiento  │
│   de Documento  │
└────────┬────────┘
         │
         ↓
   ┌──────────┐
   │  Regla:  │
   │ Max 10   │
   │ páginas  │
   └────┬─────┘
        │
        ↓
   ¿Pasa?
        │
       NO (15 páginas)
        │
        ↓
┌────────────────┐
│  on_fail_action│
│   = "webhook"  │
└────────┬───────┘
         │
         ↓
┌────────────────────────┐
│ POST su-sistema.com/   │
│    webhooks/error      │
│                        │
│ {event: "rule_...      │
│  reason: "15 páginas"} │
└────────────────────────┘
```

---

## Webhook 2: Resultado Final

### Cuándo se dispara:
- El procesamiento termina exitosamente
- Todos los documentos fueron procesados
- Todas las validaciones pasaron
- Las transformaciones de AI se completaron

### Endpoint que deben crear:

```
POST https://su-sistema.com/webhooks/extractor/success
```

### Headers que recibirán:

```http
Content-Type: application/json
Authorization: Bearer su-token-secreto
```

---

### Modo: TRANSFORMATIONS_ONLY

Este es el modo que usarán los clientes. Envía solo el resultado de las transformaciones de AI, que es la información procesada y consolidada lista para usar.

Body Example:

```json
{
  "event": "extraction_complete",
  "timestamp": "2025-01-15T10:35:22Z",
  "mode": "transformations_only",
  "record": {
    "id": "rec-uuid-123",
    "client_id": "client-uuid-456",
    "configuration_ref": "dictaminacion",
    "flow_id": "flow-uuid-789",
    "status": "complete",
    "created_at": "2025-01-15T10:30:00Z"
  },
  "data": {
    "empresa": {
      "razon_social": "ACME Corporation S.A. de C.V.",
      "rfc": "ACM950615ABC",
      "fecha_constitucion": "1995-06-15",
      "objeto_social_actual": "Desarrollo de tecnología blockchain y criptomonedas",
      "fecha_ultima_modificacion_objeto_social": "2024-12-01",
      "capital_social": "$500,000.00 MXN",
      "domicilio_fiscal": "Av. Insurgentes 123, CDMX"
    },
    "representacion_legal": {
      "representante_actual": "María López Sánchez",
      "tipo_representacion": "Apoderado General",
      "facultades": [
        "Actos de administración",
        "Representación legal",
        "Firma de contratos"
      ],
      "vigencia": "Indefinida",
      "fecha_otorgamiento": "2024-11-20"
    },
    "estructura_accionaria": [
      {
        "nombre": "Juan Pérez García",
        "porcentaje": 60,
        "tipo": "Persona Física"
      },
      {
        "nombre": "Inversiones XYZ S.A.",
        "porcentaje": 40,
        "tipo": "Persona Moral"
      }
    ],
    "historial_objeto_social": [
      {
        "objeto": "Desarrollo de software y consultoría tecnológica",
        "fecha": "1995-06-15",
        "vigencia": "1995-2024"
      },
      {
        "objeto": "Venta de productos digitales y software",
        "fecha": "2024-01-10",
        "vigencia": "Ene 2024 - Jun 2024"
      },
      {
        "objeto": "Servicios de consultoría empresarial y desarrollo tecnológico",
        "fecha": "2024-06-15",
        "vigencia": "Jun 2024 - Dic 2024"
      },
      {
        "objeto": "Desarrollo de tecnología blockchain y criptomonedas",
        "fecha": "2024-12-01",
        "vigencia": "Dic 2024 - Presente"
      }
    ],
    "ultimo_cambio_relevante": {
      "tipo": "Modificación de objeto social",
      "fecha": "2024-12-01",
      "descripcion": "Nueva línea de negocio: blockchain"
    }
  },
  "statistics": {
    "total_documents": 5,
    "processing_time_seconds": 145.3,
    "rules_executed": 8
  },
  "webhook": {
    "triggered_at": "2025-01-15T10:35:22Z"
  }
}
```

### Response esperada:

```json
{
  "received": true,
  "processed": true,
  "internal_id": "su-id-interno-123"
}
```

**Status Code:** `200-299`

### Diagrama de Flujo - Éxito:

```
┌─────────────────┐
│  Procesamiento  │
│    Completo     │
└────────┬────────┘
         │
         ↓
   ┌──────────────┐
   │ Todas las    │
   │ validaciones │
   │   pasaron    │
   └──────┬───────┘
          │
          ↓
   ┌──────────────┐
   │AI Transform  │
   │Consolida data│
   └──────┬───────┘
          │
          ↓
   ┌──────────────────┐
   │ success_webhook_ │
   │ mode configurado │
   └──────┬───────────┘
          │
    ┌─────┴─────┐
    │           │
 complete  transformations
            _only
    │           │
    ↓           ↓
┌────────┐  ┌────────┐
│documents│  │  data  │
│    +    │  │  (solo │
│consoli- │  │transf.)│
│dated    │  │        │
└────┬───┘  └────┬───┘
     │           │
     └─────┬─────┘
           ↓
┌──────────────────────┐
│ POST su-sistema.com/ │
│   webhooks/success   │
└──────────────────────┘
```

---

## Troubleshooting

### Problema: No recibo webhooks

Checklist:
- [ ] La URL del webhook es accesible públicamente
- [ ] El servidor responde con status 2xx
- [ ] No hay errores de autenticación (verificar Bearer token)
- [ ] El firewall permite conexiones entrantes
- [ ] Los certificados HTTPS son válidos

---

### Problema: Webhook falla constantemente

Posibles causas:

1. **Timeout:** El servidor tarda más de 30 segundos
   - Solución: Procesar webhook de forma asíncrona. Guardar el payload inmediatamente y responder 200, luego procesar en background.

2. **Error 401/403:** Problema de autenticación
   - Solución: Verificar que el Bearer token en el header `Authorization` es el correcto

3. **Error 5xx:** Problema en el servidor destino
   - Solución: Revisar logs del servidor para identificar la causa del error

Nota: El sistema reintentará automáticamente 3 veces si el webhook falla.

---

### Problema: Record rechazado por min/max count

Error típico:
```json
{
  "error": "Document type acta_asamblea requires at least 1 documents, but 0 provided"
}
```

Solución: Verificar que están enviando la cantidad correcta de documentos de cada tipo según la configuración. Contactar soporte para conocer los límites configurados para cada tipo de documento.

---

### Problema: Datos faltantes o incorrectos en webhook de éxito

Verificar:
1. Que el webhook está configurado en modo `transformations_only`
2. Los datos vienen en el campo `data`, no en `documents`
3. La estructura de `data` depende de la configuración específica

Ejemplo correcto:
```json
{
  "mode": "transformations_only",
  "data": {
    "empresa": { ... },
    "representacion_legal": { ... },
    ...
  }
}
```

Si los datos son incorrectos o están incompletos, contactar soporte con el `record_id` para revisión.

---

## Resumen de URLs

### Endpoints que deben crear:

| Endpoint | Método | Propósito |
|----------|--------|-----------|
| `POST https://su-sistema.com/webhooks/extractor/error` | POST | Recibir notificaciones de errores |
| `POST https://su-sistema.com/webhooks/extractor/success` | POST | Recibir datos extraídos (modo transformations_only) |

### Endpoints que deben llamar:

| Endpoint | Método | Propósito |
|----------|--------|-----------|
| `POST /clients/{client_id}/records` | POST | Subir documentos para procesamiento |

---

## Diagrama Completo del Sistema

```
┌──────────────────────────────────────────────────────────────────────┐
│                        SISTEMA COMPLETO                               │
└──────────────────────────────────────────────────────────────────────┘

    CLIENTE                          EXTRACTOR                      WEBHOOKS
┌─────────────┐                  ┌─────────────┐              ┌─────────────┐
│             │                  │             │              │             │
│   Portal    │   1. Upload      │  Processor  │   2. Error  │   Cliente   │
│   Web/App   │ ────────────────>│   Service   │ ───────────>│  Endpoint   │
│             │   POST /records  │             │  POST /error│             │
│             │                  │             │              │             │
│             │                  │      │      │              │             │
│             │                  │      ↓      │              │             │
│             │                  │  ┌───────┐  │              │             │
│             │                  │  │Extract│  │              │             │
│             │                  │  │  AI   │  │              │             │
│             │                  │  │ Rules │  │              │             │
│             │                  │  └───┬───┘  │              │             │
│             │                  │      │      │              │             │
│             │                  │      ↓      │              │             │
│             │   3. Success     │ Transform  │   4. Success │             │
│             │ <────────────────│   & Send   │ ───────────>│             │
│             │                  │  Webhook   │POST /success│             │
│             │                  │             │              │             │
└─────────────┘                  └─────────────┘              └─────────────┘
       ↑                                                              │
       │                                                              │
       └──────────────── 5. Actualiza UI/BD ────────────────────────┘
```

---

## Checklist de Implementación

### Preparar endpoints en su sistema:

- [ ] Implementar endpoint: `POST /webhooks/extractor/error`
  - Debe recibir notificaciones cuando falla una validación
  - Responder con status 2xx dentro de 30 segundos

- [ ] Implementar endpoint: `POST /webhooks/extractor/success`
  - Debe recibir el campo `data` con las transformaciones finales
  - El campo `mode` será `"transformations_only"`
  - Responder con status 2xx dentro de 30 segundos

- [ ] Configurar HTTPS (requerido)

- [ ] Configurar autenticación (Bearer token recomendado)

- [ ] Procesar webhooks de forma asíncrona (no bloquear respuesta)

- [ ] Guardar logs de webhooks recibidos para debugging

- [ ] Manejar reintentos (idempotencia) - el sistema reintentará 3 veces en caso de fallo

### Preparar integración con el API:

- [ ] Obtener su `client_id`, `API Key` y `API Secret`

- [ ] Implementar conversión de archivos PDF a base64

- [ ] Preparar lógica para llamar `POST /clients/{client_id}/records` con todos los documentos en una sola llamada

### Testing de integración:

- [ ] Enviar record de prueba con documento válido

- [ ] Verificar que webhook de éxito llegue correctamente con datos transformados

- [ ] Enviar record con documento inválido (tipo incorrecto, faltan documentos, etc)

- [ ] Verificar que webhook de error llegue correctamente

- [ ] Probar con múltiples documentos del mismo tipo

- [ ] Verificar que los datos en `data.empresa`, `data.representacion_legal`, etc. son correctos

---

*Versión del documento: 1.1*
*Última actualización: 8 de Diciembre, 2025*
