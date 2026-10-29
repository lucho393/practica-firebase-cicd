# Reflexión — CI/CD con Firebase Hosting y hardening de headers HTTP

**Materia:** Programación Avanzada — 2026

**Repositorio:** https://github.com/lucho393/practica-firebase-cicd

**Producción:** https://practica-firebase-cicd-fd7ad.web.app

**Pull Request de seguridad:** https://github.com/lucho393/practica-firebase-cicd/pull/2

## 1. ¿Por qué el preview tiene vencimiento? ¿Qué riesgos tendría un preview que no vence nunca?

Los previews tienen vencimiento porque son entornos temporales destinados a revisar cambios antes de incorporarlos a producción. Si no vencieran, podrían acumularse versiones antiguas con errores o vulnerabilidades, aumentar el consumo de recursos y dejar accesibles versiones que ya no deberían utilizarse.

En esta práctica configuramos un vencimiento de 3 días.

## 2. ¿Qué diferencia hay entre desplegar y liberar? ¿Dónde aparece cada concepto en este flujo?

Desplegar significa publicar una versión de la aplicación en un entorno. Liberar significa poner esa versión a disposición de los usuarios finales.

En esta práctica, cada Pull Request genera un despliegue Preview para realizar pruebas. Al hacer Merge a `main`, GitHub Actions despliega automáticamente la versión en producción, donde queda disponible para los usuarios.

## 3. ¿Por qué la clave de la service account va en un secreto y no en el repositorio? ¿Qué podría hacer alguien que la obtenga? ¿Cómo limitarías el daño?

La clave de la service account se guarda en GitHub Secrets para evitar que quede expuesta en el código fuente o en el historial de Git.

Si alguien obtiene esa clave, podría realizar acciones sobre Firebase según los permisos asignados a esa cuenta, incluyendo despliegues no autorizados.

Para limitar el daño, se deben otorgar solamente los permisos necesarios, restringir el acceso a los secretos y revocar o reemplazar las credenciales comprometidas.

## 4. La CSP rompió la app: ¿por qué? ¿Qué opciones evaluaste y qué costo de seguridad tiene cada una?

La Content-Security-Policy inicial permitía ejecutar únicamente scripts provenientes del propio sitio. Sin embargo, Next.js necesitaba ejecutar código JavaScript inline para inicializar la aplicación. Como consecuencia, el buscador dejó de funcionar.

Para resolverlo, se incorporó `'unsafe-inline'` en `script-src` y `style-src`, y se permitieron los orígenes de las imágenes externas utilizadas por la aplicación.

Esta solución recuperó el funcionamiento del buscador y los filtros, pero reduce la protección frente a ataques de inyección de código.

Una alternativa más segura sería autorizar los scripts mediante hashes SHA-256 calculados durante el proceso de compilación. Esto permitiría evitar `'unsafe-inline'`, aunque requeriría una configuración más compleja del pipeline.

## 5. Para cada header de B2, ¿qué ataque mitiga?

- **Strict-Transport-Security (HSTS):** obliga al navegador a utilizar HTTPS en futuras conexiones y reduce el riesgo de ataques que intentan forzar conexiones HTTP inseguras.
- **X-Content-Type-Options:** con `nosniff`, impide que el navegador interprete archivos como un tipo de contenido diferente al declarado, reduciendo riesgos de ejecución de contenido malicioso.
- **X-Frame-Options:** con `DENY`, impide que el sitio se cargue dentro de un iframe, ayudando a prevenir ataques de clickjacking.
- **Referrer-Policy:** limita la información de la URL que se comparte al navegar hacia otros sitios, reduciendo la exposición de datos.
- **Permissions-Policy:** restringe el acceso a funcionalidades del navegador como cámara, micrófono y geolocalización.

## 6. ¿Por qué los assets de /_next/static/ pueden cachearse un año y el HTML no?

Los archivos estáticos de Next.js utilizan nombres asociados a su contenido. Cuando el contenido cambia, se genera un archivo con un nombre diferente. Por eso pueden almacenarse durante un año utilizando `immutable`, sin riesgo de reutilizar una versión incorrecta bajo el mismo nombre.

En cambio, el HTML debe revalidarse para que los usuarios reciban los cambios después de un despliegue o rollback. Si se almacena demasiado tiempo sin revalidación, podrían seguir viendo una versión anterior.

## 7. Si la app necesitara SSR, ¿qué cambiaría en la arquitectura, en la CSP y en el costo?

Si la aplicación necesitara Server-Side Rendering (SSR), ya no alcanzaría con exportar archivos estáticos mediante Next.js.

Sería necesario utilizar una infraestructura capaz de ejecutar código del lado del servidor, como Firebase App Hosting sobre Cloud Run.

La CSP podría generarse dinámicamente y utilizar nonces por respuesta para autorizar scripts sin depender de `'unsafe-inline'`.

Además, aumentaría la complejidad de la arquitectura y podrían aparecer costos por ejecución, memoria y solicitudes. En Firebase sería necesario utilizar un plan con facturación, como Blaze.

## 8. Resultados de las pruebas de seguridad

### Medición inicial — Antes de configurar los headers

Comando ejecutado:

`node scripts/check-headers.mjs https://practica-firebase-cicd-fd7ad.web.app`

Resultado:

```text
HTML: status 200

✘ content-security-policy (falta)
✔ strict-transport-security
✘ x-content-type-options (falta)
✘ x-frame-options (falta)
✘ referrer-policy (falta)
✘ permissions-policy (falta)

cache-control HTML: max-age=3600

Asset estático:
✘ cache-control: max-age=3600

❌ 6 control(es) pendiente(s).
```

### Medición final — Después de configurar los headers

Comando ejecutado:

`node scripts/check-headers.mjs https://practica-firebase-cicd-fd7ad.web.app`

Resultado:

```text
▶ HTML: https://practica-firebase-cicd-fd7ad.web.app/
  status 200
  ✔ content-security-policy: default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';img-src 'self' https:…
  ✔ strict-transport-security: max-age=31556926; includeSubDomains; preload
  ✔ x-content-type-options: nosniff
  ✔ x-frame-options: DENY
  ✔ referrer-policy: strict-origin-when-cross-origin
  ✔ permissions-policy: camera=(), microphone=(), geolocation=()
  · cache-control: public, max-age=0, must-revalidate

▶ Asset estático: /_next/static/chunks/1x6obpmlu1_c2.js
  ✔ cache-control: public, max-age=31536000, immutable

✅ Todos los controles pasaron.
```

La comparación demuestra que se incorporaron los encabezados de seguridad y se corrigió la política de caché para los archivos estáticos.

También se verificó que el buscador y los filtros de `/cursos` funcionan después de ajustar la CSP.
