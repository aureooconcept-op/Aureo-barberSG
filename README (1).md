# Guía completa: Barbería en GitHub + Vercel + Turso + Google Sheets

**Cómo encaja todo**
```
Celular del barbero ──► Vercel (app + API) ──► Turso (base de datos real)
                                        └────► Google Sheets (copia en vivo de cada registro)
GitHub guarda el código; cada cambio que subes allí se publica solo en Vercel.
```
Todo se puede hacer desde el navegador, sin instalar nada. Tiempo estimado: 45–60 min.

---
## Paso 1 · GitHub (donde vive el código)
1. Crea cuenta en github.com.
2. **New repository** → nombre `barberia` → marca **Private** → Create.
3. En el repo: **Add file → Upload files**. Arrastra el *contenido* de esta carpeta (`api/`, `public/`, `package.json`, `vercel.json`, `README.md`, `.gitignore`) → **Commit changes**.
   - Verifica que en la raíz del repo veas `package.json` y las carpetas `api` y `public`.

## Paso 2 · Turso (base de datos)
1. Entra a turso.tech con tu cuenta de GitHub.
2. Crea una base de datos llamada `barberia`; elige la ubicación más cercana a Colombia disponible (p. ej. São Paulo, o la de EE. UU. este).
3. Abre la base y copia la **URL** (empieza con `libsql://`).
4. Genera un **token de acceso** para esa base (opción *Create token / Generate token*) y cópialo.
   - Guarda URL y token en un bloc de notas; el token no se vuelve a mostrar.
5. No crees tablas: la app las crea sola en el primer uso.

## Paso 3 · Google Sheets (copia en vivo)
**3.1 La hoja**
1. En Google Drive crea una hoja `Barbería – Registro`.
2. Crea 3 pestañas con estos nombres **exactos** y estos encabezados en la fila 1:

| Pestaña | A | B | C | D | E | F | G | H | I |
|---|---|---|---|---|---|---|---|---|---|
| `Cortes` | Fecha | ID | Barbero | Cliente | Teléfono | Servicio | Barba | Mascarilla | Precio |
| `Ventas` | Fecha | ID | Barbero | Producto | Cantidad | Precio unitario | Total | | |
| `Clientes` | Fecha alta | Nombre | Teléfono | | | | | | |

3. Copia el **ID de la hoja**: es el texto entre `/d/` y `/edit` en la URL.

**3.2 Cuenta de servicio (el "robot" que escribe en la hoja)**
1. Ve a console.cloud.google.com → crea un proyecto (p. ej. `barberia`).
2. Menú **APIs y servicios → Biblioteca** → busca **Google Sheets API** → **Habilitar**.
3. **APIs y servicios → Credenciales → Crear credenciales → Cuenta de servicio** → ponle un nombre → Crear → Listo.
4. Abre la cuenta creada → pestaña **Claves → Agregar clave → Crear clave nueva → JSON**. Se descarga un archivo.
5. Ábrelo con un editor de texto y copia dos valores: `client_email` y `private_key` (todo, desde `-----BEGIN` hasta `END PRIVATE KEY-----\n`).
6. Vuelve a tu hoja → **Compartir** → pega el `client_email` → permiso **Editor** → Enviar.
   - Este paso es el que más se olvida: sin él no escribe nada.

## Paso 4 · Vercel (publicar la app)
1. Entra a vercel.com con tu cuenta de GitHub (plan **Hobby**).
2. **Add New → Project** → importa el repo `barberia`.
3. Framework Preset: **Other**. Deja Build Command y Output Directory vacíos.
4. Abre **Environment Variables** y agrega (nombre → valor):

| Variable | Valor |
|---|---|
| `TURSO_DATABASE_URL` | URL de Turso (`libsql://...`) |
| `TURSO_AUTH_TOKEN` | token de Turso |
| `SECRET` | frase larga aleatoria (30+ caracteres) |
| `EXPORT_KEY` | otra frase aleatoria |
| `GOOGLE_SERVICE_ACCOUNT_EMAIL` | `client_email` del JSON |
| `GOOGLE_PRIVATE_KEY` | `private_key` del JSON, completa, tal como está (con los `\n`) |
| `GOOGLE_SHEET_ID` | ID de la hoja |

5. **Deploy**. En ~1 min tendrás una URL tipo `barberia-xxxx.vercel.app`.
6. Si cambias una variable después, hay que hacer **Deployments → ⋯ → Redeploy**.

## Paso 5 · Probar
1. Abre la URL → entra con `admin1` / `cambiar123`.
2. Abre `https://TU-URL/api/sheets-test` (con la sesión iniciada): debe responder `{"ok":true,"hoja":"Barbería – Registro"}`. Si dice error, ver "Problemas comunes".
3. En **Registrar** guarda un corte de prueba con nombre y teléfono → debe aparecer en las pestañas `Cortes` y `Clientes`.
4. En **Ajustes** cambia las claves de los 7 usuarios, los nombres reales de los barberos y tus precios.
5. En cada celular: abre la URL → menú del navegador → **Añadir a pantalla de inicio**.

## Paso 6 · Respaldo completo opcional (recomendado)
La copia en vivo agrega filas nuevas, pero no refleja correcciones. Para tener además una vista siempre completa, crea pestañas `Resp_Cortes`, `Resp_Ventas`, `Resp_Clientes`, `Resp_Resumen` y en la celda A1 de cada una:
```
=IMPORTDATA("https://TU-URL/api/export/cuts.csv?key=TU_EXPORT_KEY")
```
(cambia `cuts` por `sales`, `clients`, `resumen`). Se refresca sola cada cierto tiempo. `Resp_Clientes` incluye último corte y próximo corte.

---
## Editar después, fácilmente
| Qué quieres cambiar | Dónde |
|---|---|
| Precios, servicios, productos | App → **Ajustes** (sin código) |
| Barberos, claves, desactivar usuarios | App → **Ajustes → Usuarios** |
| Horarios de los barberos | App → **Agenda → Disponibilidad** |
| Textos, colores, mensaje de WhatsApp, días del recordatorio | Código en GitHub (abajo) |

**Editar código sin instalar nada:** en GitHub abre el archivo → ícono del lápiz → cambia → **Commit changes**. Vercel publica solo en ~1 min. Si algo sale mal, en Vercel → **Deployments** puedes volver a una versión anterior.
- Mensaje de WhatsApp y colores: `public/index.html` (busca `Hola ${c.name` y `:root{`).
- Días para recordar el próximo corte (21): columna `cycle_days` en la tabla `clients` (Turso → Shell/Editor de datos) o el valor por defecto en `api/index.js`.
- Zona horaria (UTC-5): constante `NOW` y `ts` en `api/index.js`.

## Problemas comunes
| Síntoma | Causa probable |
|---|---|
| "Base de datos: …" al entrar | `TURSO_DATABASE_URL` o `TURSO_AUTH_TOKEN` mal copiados; redeploy |
| Error 500 / "Falta la variable SECRET" | Falta `SECRET` en Vercel |
| `sheets-test` dice 403 / "permission" | No compartiste la hoja con el `client_email` como Editor |
| `sheets-test` dice 404 | `GOOGLE_SHEET_ID` incorrecto |
| `sheets-test` dice "invalid_grant" o error de clave | `GOOGLE_PRIVATE_KEY` incompleta; copia de nuevo desde `-----BEGIN` hasta `-----END PRIVATE KEY-----` |
| Se guarda el corte pero no aparece en la hoja | Nombre de pestaña distinto (`Cortes`, `Ventas`, `Clientes`) o Google lento (el registro sí queda en Turso; se ve en `Resp_*`) |
| Primer registro del día tarda unos segundos | Arranque en frío de Vercel; es normal |

## Seguridad y límites
- Repo **privado**; nunca pegues tokens ni claves en el código (van solo en las variables de Vercel).
- Cambia las 7 claves iniciales (`cambiar123`) el primer día.
- Los términos del plan Hobby de Vercel son para uso personal/no comercial; revísalos. Alternativa: el mismo código corre en otros hosts gratuitos de Node, porque los datos están en Turso.
- Revisa en turso.tech y en la consola de Google los límites vigentes de los planes gratuitos.

## Datos por defecto (editables en la app)
Clásico 25.000 · Degradado 30.000 · Niño 20.000 · Barba +12.000 · Mascarilla +10.000 · Cera, Aceite de barba, Shampoo · lun–sáb 9:00–19:00 · recordatorio a 21 días.

## Ejecutar en local (opcional)
`npm install && npm start` → http://localhost:3000 (usa `local.db`; el espejo a Sheets se activa si defines las variables `GOOGLE_*`).
