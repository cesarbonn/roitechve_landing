# Sitio web de ROI (Astro, estático)

Sitio público mínimo en español para ROI y **ROI Calendar Automation**,
construido con [Astro](https://astro.build) en modo estático (sin runtime de
servidor). Publica tres páginas: Inicio, Política de privacidad y Términos y
condiciones, requeridas para la configuración de marca de Google OAuth
(Gmail y Google Calendar).

> **Estado actual: borrador general.** Las páginas de Inicio, Privacidad y
> Condiciones contienen un borrador de trabajo (no la copia profesional ni
> legal final), señalado con un aviso visible en cada página. Deben
> revisarse y reemplazarse por la versión definitiva —idealmente con
> asesoría legal para Privacidad y Condiciones— antes de publicar el sitio.
> No se debe desplegar en producción con este borrador.

## Estructura

```
website/
├── src/
│   ├── layouts/Layout.astro     # Plantilla compartida (head, nav, footer)
│   ├── components/Nav.astro     # Navegación
│   ├── components/Footer.astro  # Pie de página
│   ├── pages/index.astro        # Inicio            → /
│   ├── pages/privacidad/        # Política de privacidad → /privacidad/
│   ├── pages/condiciones/       # Términos y condiciones → /condiciones/
│   └── pages/404.astro          # Página 404
├── deploy/
│   └── nginx.roitechve.com.example.conf  # Ejemplo de configuración nginx
└── astro.config.mjs             # site: https://roitechve.com, output estático
```

## Cómo editar el contenido

1. El texto de cada página vive directamente en su archivo `.astro`:
   - Inicio: `src/pages/index.astro`
   - Privacidad: `src/pages/privacidad/index.astro`
   - Condiciones: `src/pages/condiciones/index.astro`
2. Reemplazar los bloques marcados con `[Pendiente: ...]` por la copia
   aprobada, preservando su significado. No se deben inventar prácticas de
   privacidad, plazos de retención de datos, compromisos legales, datos de
   la empresa ni información de contacto.
3. Quitar el aviso `<div class="placeholder-notice">...</div>` de cada
   página solo cuando su contenido ya sea el aprobado y definitivo.
4. Los títulos de página y las meta-descripciones se definen al llamar al
   layout compartido, por ejemplo:

   ```astro
   <Layout
     title="Política de privacidad | ROI Calendar Automation"
     description="..."
   >
   ```

5. El idioma del documento (`<html lang="es">`) se define una sola vez en
   `src/layouts/Layout.astro` y aplica a todas las páginas.

## Cómo compilar y previsualizar el sitio

Requiere Node.js (probado con Node 22+) y npm.

```bash
cd website
npm install        # solo la primera vez, o si cambian las dependencias
npm run dev        # servidor de desarrollo en http://localhost:4321
npm run build      # genera el sitio estático en dist/
npm run preview    # sirve dist/ localmente para revisar el build final
```

`npm run build` no requiere conexión a internet salvo para instalar
dependencias la primera vez, y no inicia ningún proceso de servidor en
producción: el resultado es HTML, CSS y assets estáticos en `dist/`.

## Cómo desplegar `dist/` a través del nginx existente

Este sitio **no** introduce un nuevo proceso de servidor ni un segundo
proxy. Se sirve como archivos estáticos a través del nginx que ya administra
HTTPS en el VPS, sin tocar n8n ni su configuración.

Pasos generales (confirmar cada detalle contra el servidor real antes de
ejecutar nada):

1. **Compilar localmente:**
   ```bash
   npm run build
   ```
2. **Inspeccionar el nginx activo en el VPS** antes de cambiar nada:
   - `sudo nginx -T` para ver toda la configuración activa.
   - Confirmar si ya existe un server block para `roitechve.com`.
   - Confirmar rutas de certificados TLS vigentes y su método de renovación.
   - Verificar que no haya conflicto con el server block de
     `n8n.roitechve.com`.
3. **Hacer una copia de respaldo** de cualquier archivo de nginx que se vaya
   a modificar, por ejemplo:
   ```bash
   sudo cp /etc/nginx/sites-available/roitechve.com \
           /etc/nginx/sites-available/roitechve.com.bak-$(date +%Y%m%d%H%M%S)
   ```
4. **Copiar `dist/` al servidor**, a una ruta dedicada que no se mezcle con
   otros sitios (ajustar la ruta real según lo que exista en el VPS):
   ```bash
   rsync -avz --delete ./dist/ deploy@<host>:/var/www/roitechve.com/html/
   ```
5. **Adaptar y aplicar la configuración de nginx** usando
   `deploy/nginx.roitechve.com.example.conf` únicamente como referencia, no
   como archivo final. El bloque real debe ajustarse a las rutas y
   certificados verificados en el paso 2.
6. **Verificar la sintaxis antes de recargar:**
   ```bash
   sudo nginx -t
   ```
7. **Recargar nginx** (no reiniciar, para evitar cortar conexiones
   innecesariamente):
   ```bash
   sudo systemctl reload nginx
   ```

Ninguno de estos pasos debe ejecutarse sin autorización explícita, y ninguno
debe detener, reinstalar ni reconfigurar n8n, PostgreSQL o Redis.

## Cómo verificar las tres URLs públicas

Después de desplegar:

```bash
curl -I https://roitechve.com/
curl -I https://roitechve.com/privacidad/
curl -I https://roitechve.com/condiciones/
```

Cada una debe responder `HTTP/2 200`. También conviene:

- Revisar visualmente cada página en el navegador.
- Confirmar que `https://n8n.roitechve.com` sigue respondiendo con
  normalidad y que sus rutas de webhook no cambiaron.
- Revisar `sudo nginx -t` y los logs de error de nginx
  (`/var/log/nginx/roitechve.com.error.log` o el que corresponda) por si
  hay advertencias.

## Cómo actualizar el sitio

1. Editar el contenido en `src/`.
2. `npm run build` localmente y revisar con `npm run preview`.
3. Repetir el paso de copia (`rsync`) hacia la misma ruta en el VPS.
4. No es necesario tocar la configuración de nginx para actualizaciones de
   contenido; solo se vuelve a sincronizar `dist/`.

## Cómo revertir (rollback)

- **Contenido:** mantener cada despliegue en una carpeta con marca de
  tiempo en el servidor (por ejemplo
  `/var/www/roitechve.com/releases/<fecha>/`) y apuntar `root` de nginx a un
  enlace simbólico `current` que se actualiza en cada despliegue. Así, revertir
  es solo redirigir el enlace simbólico a la carpeta anterior y recargar
  nginx. Si no se usa ese esquema, conservar una copia de la carpeta
  `dist/` anterior antes de sobrescribirla.
- **Configuración de nginx:** restaurar el archivo de respaldo creado en el
  paso 3 del despliegue y ejecutar `sudo nginx -t && sudo systemctl reload
  nginx`.
- **Certificados TLS:** no se gestionan desde este proyecto; se reutiliza la
  configuración de certificados ya existente en el VPS.

## Alcance explícitamente fuera de este proyecto

- Publicación, verificación y revisión de la app en Google OAuth.
- Reconexión de credenciales de Gmail/Google Calendar.
- Pruebas end-to-end de automatizaciones.
- Analítica, cookies, seguimiento, formularios o integraciones de terceros:
  este sitio no incluye ninguno de forma intencional.
