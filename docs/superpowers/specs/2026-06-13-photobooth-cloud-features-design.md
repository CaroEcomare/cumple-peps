# Photobooth: guardar en galería y subir a Google Drive

## Contexto

`photobooth.html` deja tomar una foto a la vez (preview + retake + descarga vía
`<a download>`). Varios invitados lo van a usar desde su propio celular durante
la fiesta. Se pidieron dos mejoras:

1. Que la foto se guarde directo en la galería del celular.
2. Un botón para subir la foto a una carpeta de Google Drive de Caro.

## Alcance

- Sigue siendo un sitio estático (GitHub Pages, `https://caroecomare.github.io/cumple-peps/`).
  No se agrega backend.
- Maneja UNA foto a la vez, igual que ahora (no se agrega galería de sesión).
- Subida a Drive: cada invitado autentica con SU propia cuenta de Google
  (login client-side, sin "client secret"), y el archivo se sube a una carpeta
  compartida por Caro.

## Componente 1: Guardar en galería

Modificar `savePh()` para usar la Web Share API cuando el navegador la soporte
con archivos:

```js
async function savePh() {
  if (!lastDataUrl) return;
  const blob = dataURLtoBlob(lastDataUrl);
  const file = new File([blob], 'peps60-foto.jpg', { type: 'image/jpeg' });

  if (navigator.canShare && navigator.canShare({ files: [file] })) {
    try {
      await navigator.share({ files: [file], title: 'Foto Peps 60' });
      return;
    } catch (e) {
      if (e.name === 'AbortError') return; // usuario canceló, no hacer nada más
      // si falla por otra razón, cae al método de descarga abajo
    }
  }

  const a = document.createElement('a');
  a.download = 'peps60-foto.jpg';
  a.href = lastDataUrl;
  a.click();
}
```

- En iOS/Android con soporte: abre el panel nativo de compartir, que incluye
  "Guardar imagen" → va directo a Fotos/Galería.
- Sin soporte (la mayoría de desktop): mantiene el comportamiento actual de
  descarga.
- Se agrega un helper `dataURLtoBlob(dataUrl)` (decodifica el base64 del
  `data:` URL a un `Blob`), compartido con el Componente 2.

## Componente 2: Subir a Google Drive

### Autenticación

Se carga el script de Google Identity Services:

```html
<script src="https://accounts.google.com/gsi/client" async defer></script>
```

Y se inicializa un token client (flujo OAuth implícito, sin backend, sin
"client secret"):

```js
const GOOGLE_CLIENT_ID = '__REEMPLAZAR_CON_CLIENT_ID__';
const DRIVE_FOLDER_ID  = '__REEMPLAZAR_CON_FOLDER_ID__';

let accessToken = null;
let tokenClient = null;

function initGoogleAuth() {
  tokenClient = google.accounts.oauth2.initTokenClient({
    client_id: GOOGLE_CLIENT_ID,
    scope: 'https://www.googleapis.com/auth/drive.file',
    callback: (resp) => {
      if (resp.error) { setUploadState('error'); return; }
      accessToken = resp.access_token;
      doUpload();
    },
  });
}
```

`GOOGLE_CLIENT_ID` y `DRIVE_FOLDER_ID` quedan como placeholders hasta que Caro
complete la configuración (sección siguiente).

### Subida del archivo

Drive API v3 requiere un body `multipart/related` (no `multipart/form-data`)
para subir metadata + contenido en una sola petición:

```js
async function doUpload() {
  setUploadState('uploading');
  try {
    const base64Data = lastDataUrl.split(',')[1];
    const metadata = {
      name: `peps60-foto-${Date.now()}.jpg`,
      parents: [DRIVE_FOLDER_ID],
      mimeType: 'image/jpeg',
    };
    const boundary = 'peps60_boundary';
    const body =
      `--${boundary}\r\n` +
      `Content-Type: application/json\r\n\r\n` +
      JSON.stringify(metadata) + `\r\n` +
      `--${boundary}\r\n` +
      `Content-Type: image/jpeg\r\n` +
      `Content-Transfer-Encoding: base64\r\n\r\n` +
      base64Data + `\r\n` +
      `--${boundary}--`;

    const res = await fetch(
      'https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart',
      {
        method: 'POST',
        headers: {
          Authorization: `Bearer ${accessToken}`,
          'Content-Type': `multipart/related; boundary=${boundary}`,
        },
        body,
      }
    );

    if (res.status === 401) {
      // token vencido/inválido: pedir uno nuevo y reintentar
      accessToken = null;
      tokenClient.requestAccessToken();
      return;
    }
    if (!res.ok) throw new Error('upload failed: ' + res.status);

    setUploadState('success');
  } catch (e) {
    setUploadState('error');
  }
}

function uploadToDrive() {
  if (!lastDataUrl) return;
  if (!accessToken) {
    tokenClient.requestAccessToken(); // dispara login/consentimiento si hace falta
    return; // el callback llama a doUpload()
  }
  doUpload();
}
```

### UI

Nuevo botón en `.btns`, junto a `#sb` ("Guardar"), visible solo cuando ya hay
foto tomada (`taken===true`, igual que `#sb`):

| Estado | Texto/ícono | Habilitado |
|---|---|---|
| Normal | "☁️ Subir" | sí |
| Subiendo | "Subiendo…" | no |
| Éxito | "✓ Subido" | sí (puede volver a "Normal" tras unos segundos) |
| Error | "⚠️ Reintentar" | sí |

`retake()` resetea este botón a "Normal".

### Manejo de errores

- **Usuario cierra/cancela el login de Google**: el callback recibe
  `resp.error` → estado "error" con texto breve ("No se pudo iniciar sesión").
- **Popup bloqueado por el navegador**: GIS muestra su propio aviso; no se
  agrega manejo adicional (caso raro en móvil).
- **Token expirado (401) o error de red**: estado "error" con botón de
  reintentar (`uploadToDrive()` de nuevo).

## Configuración previa requerida (Caro, antes de probar)

Estos pasos los hace Caro en su cuenta de Google — no se pueden automatizar
desde aquí:

1. [console.cloud.google.com](https://console.cloud.google.com) → crear/usar
   un proyecto → **APIs & Services → Library** → activar **Google Drive API**.
2. **APIs & Services → OAuth consent screen**:
   - Tipo de usuario: **External**.
   - Nombre de la app (ej. "Photobooth Cumple Peps"), correo de soporte.
   - Scopes: agregar `https://www.googleapis.com/auth/drive.file`.
   - Estado de publicación: **In production** (evita el límite de 100
     usuarios de prueba; cada invitado verá la pantalla de "app no verificada"
     y deberá darle a "Avanzado → Ir a [app] (no seguro)").
3. **APIs & Services → Credentials → Create Credentials → OAuth client ID**:
   - Tipo: **Web application**.
   - Authorized JavaScript origins: `https://caroecomare.github.io`.
   - Copiar el **Client ID** (termina en `.apps.googleusercontent.com`, no es
     secreto).
4. En Google Drive: crear una carpeta para las fotos de la fiesta → click
   derecho → **Compartir** → cambiar a **"Cualquiera con el enlace" → Editor**.
   Copiar el ID de la carpeta (segmento de la URL después de `/folders/`).
5. Pasar a Claude: el **Client ID** y el **ID de carpeta** para reemplazar los
   placeholders en `photobooth.html`.

### Nota de seguridad

Compartir la carpeta como "Cualquiera con el enlace → Editor" significa que
cualquiera con ese link puede ver/editar/borrar lo que haya en esa carpeta
(no solo subir). El scope `drive.file` limita lo que la *app* puede tocar
(solo archivos que ella misma sube), pero el acceso a la carpeta vía Drive
normal es más amplio. Recomendación: usar una carpeta dedicada solo para esto
(no una con archivos importantes), y mover las fotos a un lugar privado
después de la fiesta si se quiere restringir el acceso.

## Pruebas

- **Lógica pura** (sin navegador): script Node que valida
  `dataURLtoBlob` y la construcción del body `multipart/related` contra el
  formato esperado por Drive API.
- **Guardar en galería**: Playwright con `navigator.share`/`canShare` mockeado
  (soportado y no soportado) para confirmar que se toma cada rama.
- **Subida a Drive**: requiere credenciales reales de Caro — prueba manual en
  su celular una vez completada la configuración, antes de que lleguen los
  invitados.
