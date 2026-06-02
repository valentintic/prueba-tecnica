# Prueba Técnica — Escapar del navegador integrado de Gmail (Android)

## Contexto del problema

Cuando un usuario abre un enlace desde la aplicación Gmail en Android, este se carga dentro de un **Custom Chrome Tab (CCT)**: un navegador integrado gestionado por la propia app de correo. Aunque visualmente se parece a Chrome, sigue "dentro" de Gmail, lo que puede limitar funcionalidades (cookies de terceros, APIs de dispositivo, etc.).

El objetivo: al pulsar un botón en la web, **forzar que la página se abra en Chrome (o el navegador por defecto) como aplicación independiente**, escapando completamente del entorno de Gmail.

---

## Enfoque adoptado: Android Intent URI

### Por qué esta técnica

Android expone un mecanismo de IPC llamado **Intent** para comunicar actividades entre apps. Un Intent URI tiene la forma:

```
intent://HOST/PATH#Intent;scheme=https;package=com.android.chrome;S.browser_fallback_url=ENCODED_URL;end
```

Cuando el navegador navega a esta URL, en lugar de cargarla como una página web, el sistema operativo Android la interpreta como una orden para **lanzar otra aplicación** (en este caso Chrome) con la URL indicada. Esto hace que Chrome se abra como actividad independiente, completamente fuera del stack de Gmail.

**¿Por qué funciona desde un Custom Chrome Tab?**  
Los CCT están diseñados deliberadamente para pasar los `intent://` URIs al sistema Android — es un comportamiento documentado que permite la comunicación inter-app. A diferencia de un WebView estricto, el CCT no bloquea este tipo de navegación.

### Flujo de la solución

```
Usuario abre enlace en Gmail
        ↓
  Página carga en CCT
        ↓
Usuario pulsa el botón
        ↓
JS navega a intent:// URI
        ↓
Android lanza Chrome como app independiente
        ↓
  Chrome carga la misma URL
```

Si tras 1,5 s el intent no funcionó (bloqueado o Chrome no instalado), se muestran instrucciones manuales ("Abrir en Chrome" desde el menú de tres puntos).

---

## Alternativas evaluadas

| Opción | Descripción | Por qué se descartó |
|---|---|---|
| `window.open(url, '_blank')` | Abrir en nueva pestaña | En CCT solo abre otra CCT; no escapa del entorno de Gmail |
| `window.location.href = 'googlechrome://...'` | Scheme propietario de Chrome | No es estándar; fue deprecado/bloqueado en versiones modernas |
| `x-safari-https://...` (iOS) | Abrir en Safari | Bloqueado por WKWebView desde iOS 14; no aplica a Android |
| Firebase Dynamic Links / Branch.io | Servicios de deep linking | Funcionan muy bien, pero añaden dependencia de terceros y configuración DNS (`/.well-known/assetlinks.json`). Overkill para una página estática. |
| Android App Links | Asociar dominio a una app nativa | Requiere publicar una app; imposible sin ella |
| Instrucciones manuales únicamente | Decirle al usuario qué hacer | Funciona siempre, pero es la peor UX |

---

## Limitaciones conocidas

1. **Detección de CCT no es fiable**: La User-Agent de un Custom Chrome Tab es idéntica a la de Chrome en modo normal (no incluye el flag `wv` que sí aparece en WebViews clásicos). Por ello, la detección es heurística: si el dispositivo es Android y no se reconocen otros in-app browsers conocidos (Facebook, Instagram…), se asume que "podría estar en Gmail". No es un falso positivo grave, ya que el botón simplemente no hace nada perjudicial en un Chrome normal.

2. **No todas las versiones de Gmail se comportan igual**: Versiones antiguas de Gmail usaban un WebView clásico (no CCT). En ese caso el `intent://` también funciona, pero puede ser necesario incluir la categoría `BROWSABLE` explícitamente.

3. **iOS**: No existe ningún mecanismo programático estándar para forzar apertura en Safari desde un WKWebView. La única opción viable es mostrar instrucciones al usuario (`Compartir → Abrir en Safari`). Apple no expone un API equivalente al Intent de Android.

4. **Desktop**: En escritorio el problema no existe; los enlaces de correo abren directamente en el navegador por defecto.

5. **Chrome no instalado en Android**: Si el usuario no tiene Chrome (`com.android.chrome`), el intent falla y se activa `S.browser_fallback_url`, que redirige a la misma página. En ese momento se muestran las instrucciones manuales. Una mejora sería añadir un segundo intento con `package=com.google.android.browser` para el navegador AOSP genérico.

---

## Estructura del proyecto

```
├── index.html   # Página principal con botón y lógica JS
└── README.md    # Este archivo
```

Sin dependencias externas. HTML/CSS/JS vanilla puro.

---

## Despliegue sugerido

La forma más rápida para obtener una URL pública con un repositorio público es **GitHub Pages**:

1. Crear un repositorio público en GitHub y subir los ficheros.
2. En `Settings → Pages`, seleccionar rama `main` y carpeta `/ (root)`.
3. GitHub publica la página en `https://<usuario>.github.io/<repo>/`.

Alternativas igualmente válidas: **Netlify** (drag & drop de la carpeta) o **Vercel** (conectando el repo de GitHub).
