# Arquitectura — Salaverry Emprende

## Visión general

Aplicación web de directorio de emprendimientos sin framework ni base de datos: el frontend consume un archivo JSON plano que puede actualizarse desde un panel de administración a través de una API minimalista en PHP.

```
┌─────────────────────────┐
│       Visitante         │
└───────────┬─────────────┘
            │ HTTPS
            ▼
┌─────────────────────────┐      ┌──────────────────────────────┐
│      Sitio público      │      │        Panel admin           │
│  index.html             │      │  admin/index.html            │
│  perfil.html?id=...     │      │  login + CRUD + categorías   │
│  búsqueda / filtros     │      │  borrador en IndexedDB       │
└───────────┬─────────────┘      └──────────────┬───────────────┘
            │ fetch GET                         │ fetch POST (password)
            ▼                                   ▼
   data/emprendimientos.json ◀──── api/data.php (PHP)
        (fuente de verdad)          GET: leer · POST: guardar
```

## Estructura de archivos

```
SALAVERRY-EMPRENDE/
├── index.html              # Sitio público (SPA-like, secciones ancla)
├── perfil.html             # Perfil individual compartible (?id=N)
├── manifest.json           # Manifest PWA
├── sw.js                   # Service worker (caché offline)
├── css/
│   └── style.css           # Estilos globales (incluye dark mode)
├── js/
│   ├── shared.js           # Utilidades comunes (IndexedDB, dark mode, helpers)
│   └── app.js              # Motor del sitio: render, búsqueda, filtros,
│                           # orden, paginación, modal, slider, lightbox
├── admin/
│   ├── index.html          # Panel administrativo (login + vistas)
│   ├── css/admin.css       # Estilos del panel
│   └── js/admin.js         # CRUD, categorías, CSV, IndexedDB, sync API
├── api/
│   └── data.php            # API REST mínima sobre archivo JSON
├── data/
│   └── emprendimientos.json# Catálogo (fuente de verdad)
└── img/                    # Banners, logo e imágenes de emprendimientos
```

## Modelo de datos (`emprendimientos.json`)

Cada emprendimiento es un objeto:

| Campo | Tipo | Descripción |
|---|---|---|
| `id` | number | Identificador único |
| `nombre` | string | Nombre del emprendedor |
| `negocio` | string | Nombre comercial |
| `slug` | string | Slug URL-friendly |
| `categoria` | string | Clave de categoría (`gastronomia`, `moda`, …) |
| `categoriaLabel` | string | Etiqueta visible |
| `descripcionCorta` | string | Resumen para la tarjeta |
| `descripcionLarga` | string | Historia completa para el perfil |
| `imagen` | string | Imagen principal |
| `galeria` | string[] | Rutas de imágenes de galería |
| `telefono` / `whatsapp` / `correo` | string | Contacto |
| `direccion` | string | Dirección física |
| `mapaSrc` | string | URL del embed de Google Maps |
| `redes` | object | Enlaces a facebook / instagram / tiktok |

Las **categorías** personalizadas creadas en el panel se persisten aparte (IndexedDB en cliente + JSON servido por la API).

## Flujo de datos

1. **Lectura (público)**: `js/app.js` hace `fetch('data/emprendimientos.json')` directamente (estático, cacheable). Si el sitio se sirve con PHP, `api/data.php` ofrece la misma lectura.
2. **Escritura (admin)**: el panel trabaja sobre una copia local en IndexedDB (borrador instantáneo) y publica mediante `POST api/data.php` con la contraseña de administración; la API reescribe el JSON completo.
3. **Offline**: `sw.js` cachea assets y datos; el sitio sigue navegable sin conexión.

## API `api/data.php`

| Método | Auth | Descripción |
|---|---|---|
| `GET` | no | Devuelve el catálogo completo (`application/json`) |
| `POST` | sí (`{ password, data }`) | Valida contraseña y sobrescribe el JSON |
| `?test=1` | no | Diagnóstico: permisos de carpeta/archivo, versión PHP |

## Notas de despliegue

- Requiere cualquier hosting con **PHP 7+** (el sitio público funciona incluso solo estático; PHP es necesario para publicar cambios desde el panel).
- El servidor debe tener permiso de escritura sobre `data/`.
- No se requiere base de datos ni procesos en segundo plano.

## Seguridad conocida

- La contraseña del panel viaja en el cuerpo del `POST` y se valida en el servidor; conviene servirla siempre bajo HTTPS.
- Para un catálogo abierto a internet se recomienda migrar la autenticación a hashes/sesiones si el proyecto crece.
