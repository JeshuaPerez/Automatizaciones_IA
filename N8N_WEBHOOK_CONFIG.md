# Configuración del nodo Webhook en n8n

## 1. Crear el nodo Webhook

1. En tu flujo de n8n, agrega un nodo **Webhook** (nodo disparador).
2. Configura los siguientes campos:

| Campo | Valor |
|---|---|
| **HTTP Method** | `POST` |
| **Path** | `justificacion` (o el segmento que prefieras) |
| **Authentication** | None *(puedes agregar Header Auth después)* |
| **Response Mode** | `Respond to Webhook` |
| **Response Code** | `200` |

La URL resultante tendrá la forma:
```
https://TU_INSTANCIA_N8N/webhook/justificacion
```
Copia esa URL y pégala en la variable `WEBHOOK_URL` al inicio del script en `formulario/index.html`.

---

## 2. Activar la recepción de archivos binarios

El formulario envía `multipart/form-data`. Para que n8n procese correctamente el archivo adjunto:

1. En el nodo Webhook, despliega **Options**.
2. Activa la opción **"Binary Data"** → `true`.
3. En **"Binary Property"** escribe: `archivo`  
   (debe coincidir con el `name` del campo `<input type="file" name="archivo">` del formulario).

> Sin este paso n8n ignorará el archivo o lo recibirá corrupto.

---

## 3. Acceder a los campos en nodos siguientes

### Campos de texto (`$body`)

Todos los campos de texto del formulario llegan en el objeto `body` del ítem:

```javascript
// En un nodo "Set", "Code" o expresión de cualquier nodo:
{{ $body.nombre }}
{{ $body.id_estudiante }}
{{ $body.correo }}
{{ $body.telegram_id }}
{{ $body.motivo }}
{{ $body.descripcion }}    // puede estar vacío si el usuario no lo completó
```

### Archivo adjunto (`$binary`)

El archivo llega bajo la clave `archivo` (la que configuraste en Binary Property):

```javascript
{{ $binary.archivo.fileName }}    // nombre original del archivo
{{ $binary.archivo.mimeType }}    // "image/jpeg", "image/png" o "application/pdf"
{{ $binary.archivo.fileSize }}    // tamaño en bytes
{{ $binary.archivo.data }}        // contenido en base64 (para adjuntar a correo, Drive, etc.)
```

> El campo `archivo` sólo existirá en el ítem si el usuario adjuntó un archivo.  
> Protege los nodos posteriores con una condición `{{ $binary.archivo !== undefined }}`.

---

## 4. Flujo recomendado (resumen)

```
[Webhook POST /justificacion]
        │
        ▼
[IF: ¿tiene archivo?]  ──── No ───▶  (continuar sin subir nada)
        │ Sí
        ▼
[Google Drive / S3: subir $binary.archivo]
        │
        ▼
[Google Sheets: append fila]
   - Columna A: {{ $body.nombre }}
   - Columna B: {{ $body.id_estudiante }}
   - Columna C: {{ $body.correo }}
   - Columna D: {{ $body.telegram_id }}
   - Columna E: {{ $body.motivo }}
   - Columna F: {{ $body.descripcion }}
   - Columna G: URL del archivo subido (output del nodo anterior)
   - Columna H: {{ $now.toISO() }}   ← timestamp
        │
        ▼
[Telegram: sendMessage]
   chat_id : {{ $body.telegram_id }}
   text    : "✅ Hola {{ $body.nombre }}, tu justificación fue recibida."
        │
        ▼
[Webhook Response: 200 OK]
```

---

## 5. Respuesta que espera el formulario

El formulario solo verifica que el status HTTP sea **2xx** para mostrar el mensaje de éxito.  
Usa el nodo **"Respond to Webhook"** (o la respuesta automática del Webhook) con código `200`.  
El cuerpo de la respuesta puede ser vacío o un JSON simple: `{ "ok": true }`.

---

## 6. Prueba local del formulario

Mientras desarrollas, el formulario se puede abrir directamente en el navegador (`file:///…/formulario/index.html`).  
Para evitar errores CORS al enviar al webhook durante pruebas:

- Activa el flujo en n8n (cambiar de **Test** a **Production** o usar la URL de producción).
- O bien levanta un servidor HTTP local simple:  
  ```bash
  # Python 3
  python -m http.server 8080
  # luego abre http://localhost:8080/formulario/
  ```
- Asegúrate de que tu instancia n8n acepte peticiones desde `localhost` (revisa CORS en la configuración de n8n si usas self-hosted).
