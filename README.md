# Prueba Técnica — Abrir enlace en navegador externo desde Gmail (Android)

## El problema

El caso es el siguiente: un usuario recibe un correo en Gmail para Android, toca el enlace y la web se abre... pero dentro de Gmail. No en Chrome, no en el navegador del teléfono, sino en una especie de visor integrado que Gmail monta por su cuenta.

Esto es un problema porque ese visor no es un navegador completo. Puede tener restricciones con cookies, con ciertas APIs del dispositivo, con el comportamiento de algunas funcionalidades web. El objetivo de esta prueba es conseguir que, al pulsar un botón en esa página, la web se cargue en el navegador real del teléfono, escapando del entorno de Gmail.

---

## Por dónde empecé a buscar

Lo primero que hice fue intentar entender qué tipo de "navegador" usa Gmail exactamente, porque eso condiciona completamente qué soluciones son viables.

Resultó que hay dos casos distintos según la versión de Gmail:

- **WebView clásico**: versiones antiguas de Gmail abren los enlaces en un WebView de Android, que es básicamente un componente de Chrome recortado y embebido en la app. Tiene bastantes restricciones.
- **Custom Chrome Tab (CCT)**: versiones modernas de Gmail usan esto. Un Custom Chrome Tab es Chrome de verdad, pero lanzado en modo "adjunto" a la app que lo abre. Visualmente parece Chrome, tiene sus mismas capacidades, pero sigue corriendo dentro del contexto de Gmail.

El segundo caso es el que aplica hoy en día, y es el más complicado de detectar, porque la User-Agent que reporta un CCT es exactamente la misma que la de Chrome normal. No hay ningún flag ni marca que lo diferencie programáticamente.

---

## Cosas que probé y no funcionaron (o no eran suficientes)

**`window.open(url, '_blank')`**: Lo primero que se me ocurrió. Abre una nueva pestaña, pero dentro del mismo CCT. No escapa a ningún lado.

**`window.location.href = 'googlechrome://...'`**: Chrome tiene (o tenía) un scheme propietario para abrirse a sí mismo como app independiente. Está deprecado y bloqueado en versiones modernas. Descartado.

**Firebase Dynamic Links / Branch.io**: Servicios de deep linking que permiten abrir apps y navegar fuera de entornos integrados. Funcionan bien, pero requieren configuración de dominio (`/.well-known/assetlinks.json`), una cuenta en esos servicios y una app nativa. Demasiado para una página estática.

**Android App Links**: Misma idea pero nativo de Android. Requiere publicar una app que registre el dominio. No aplicable.

---

## La solución que funciona: Android Intent URI

Android tiene un sistema de comunicación entre apps llamado **Intent**. Básicamente es una forma de decirle al sistema operativo "quiero hacer esta acción, búscame una app que la gestione". Una de las formas de lanzar un Intent desde el navegador es usando URIs con el esquema `intent://`.

La estructura es:
```
intent://HOST/PATH#Intent;scheme=https;package=com.android.chrome;S.browser_fallback_url=URL;end
```

Cuando el navegador (en este caso el CCT de Gmail) intenta navegar a una URL con ese esquema, en vez de cargarla como una página web, Android la intercepta y la trata como una orden para lanzar otra aplicación. El resultado: Chrome se abre como actividad independiente, fuera del stack de Gmail.

El motivo por el que esto funciona en un CCT y no en un WebView estricto es que los Custom Chrome Tabs están diseñados explícitamente para pasar los `intent://` al sistema. Es un comportamiento documentado y esperado.

### Soporte de múltiples navegadores

Una mejora importante sobre el enfoque inicial (que solo apuntaba a Chrome) es no especificar ningún paquete en el primer intento:

```
intent://HOST/PATH#Intent;scheme=https;S.browser_fallback_url=URL;end
```

Sin `package`, Android usa el navegador por defecto del usuario. Si tiene Brave como navegador principal, lo abrirá en Brave. Si tiene Firefox, en Firefox. Esto hace la solución agnóstica al navegador instalado.

Si ese primer intento falla (el sistema no resuelve el intent), la página reintenta con paquetes específicos en orden: Chrome, Brave, Firefox, Edge, Opera, Samsung Internet.

---

## Limitaciones conocidas

- **La detección de CCT no es fiable**: Como la UA es idéntica a Chrome, no podemos saber con certeza si estamos en Gmail o en Chrome normal. La heurística es: si el dispositivo es Android y no detectamos otros in-app browsers conocidos, asumimos que podría ser Gmail. En la práctica esto no causa problemas, el botón simplemente no hace nada dañino en Chrome real.

- **iOS no tiene equivalente**: Apple no expone ningún API para forzar apertura en Safari desde un WKWebView. No hay intent system, no hay nada programático. La única opción sería instrucciones manuales al usuario.

- **Si ningún intent funciona**: el botón vuelve a su estado inicial. El usuario puede reintentar.

---

## Estructura

```
├── index.html
└── README.md
```

Sin dependencias. HTML, CSS y JS vanilla.

---

## Despliegue

Repositorio público en GitHub + GitHub Pages (`Settings → Pages → branch main`).

URL pública: `https://valentintic.github.io/prueba-tecnica/`

