# Dashboards ZB

Catálogo de dashboards de ZetaBé. Cada reporte tiene una ficha que explica qué responde, su estado de actualización a la vista, y un buscador que entiende preguntas en lenguaje normal. Los dashboards se abren embebidos, sin sacar a nadie del sitio.

Primer módulo del Hub de ZetaBé. Funciona de forma autónoma.

**En producción:** https://nicolasorjuela-zb.github.io/DashbordsZB/

---

## Cómo está armado

Sin framework y sin build. Tres archivos estáticos que hablan directo con Supabase.

```
index.html          la aplicación entera
zb.css              sistema visual del Hub — compartido entre módulos
dashboards.css      estilos propios de este módulo
supabase/
  functions/
    buscar/
      index.ts      Edge Function del buscador con IA
migrations/         el esquema, en orden
```

| Capa | Qué se usa |
|---|---|
| Frontend | HTML, CSS y JavaScript sin dependencias. `@supabase/supabase-js` por CDN |
| Base de datos | Postgres en Supabase, con Row Level Security |
| IA | API de Anthropic desde una Edge Function en Deno |
| Hosting | GitHub Pages |

## Levantarlo local

Los módulos de JavaScript no cargan desde `file://`, así que hace falta servirlo:

```bash
python3 -m http.server 8080
```

Y abrir `http://localhost:8080`.

La URL de Supabase y la `anon key` están en `index.html`. **La anon key es pública a propósito**: no otorga ningún permiso por sí sola, porque quién ve qué lo decide el RLS de la base. La que nunca debe aparecer en este repositorio es la `service_role`.

## Base de datos

Las migraciones se corren en orden en el SQL Editor de Supabase. Son acumulativas.

| | |
|---|---|
| `01_schema` | Tablas, ficha, frescura relativa, accesos, RLS, vista `catalogo` |
| `02_setup` | RLS de las tablas restantes y datos de prueba |
| `03_pruebas` | Verificación de permisos simulando sesiones (correr bloque por bloque) |
| `04_admin_y_pedidos` | Rol admin, apertura al equipo por defecto, pedidos de acceso |
| `05_edicion` | Edición y archivado desde la aplicación, normalización de etiquetas |
| `06_backlog` | Búsquedas sin resultado y tasa de aciertos |
| `07_etiquetas` | Vocabulario cerrado validado por trigger |
| `08_idioma` | Preferencia de idioma en el perfil |

### Cosas que conviene saber antes de tocar el esquema

**Los permisos viven en RLS, no en la aplicación.** Una pantalla nueva no necesita revalidar nada: si consulta la vista `catalogo`, ya está filtrada. Agregar lógica de permisos en el frontend no suma seguridad y sí agrega lugares donde equivocarse.

**Ver la ficha y ver el dashboard son permisos distintos.** La vista `catalogo` nunca incluye la URL. Se pide con `get_embed_url(id)`, que devuelve nulo si no hay acceso.

**El SQL Editor corre como `postgres` y saltea el RLS.** Para probar permisos de verdad hay que cambiar de rol y plantar el claim, como hace `03_pruebas`.

**`create or replace view` no renombra columnas.** Si cambia el nombre o el tipo de una columna, hay que hacer `drop view` primero.

**Roles:** `admin` ve y abre todo, otorga accesos y resuelve pedidos. `puede_cargar` sube dashboards. Ambos se asignan desde el SQL Editor, nunca desde la aplicación.

```sql
update perfiles set admin = true where email = 'persona@zetabe.com';
```

## Edge Function

```bash
supabase functions deploy buscar
supabase secrets set ANTHROPIC_API_KEY=sk-ant-...
```

También se puede desplegar desde el dashboard de Supabase, en Edge Functions → Deploy a new function → Via Editor.

La función construye su cliente de Supabase **con el token de quien pregunta**, no con `service_role`. Así el RLS sigue aplicando adentro y el modelo nunca ve una ficha que el usuario no podría ver. Si se cambia eso, la respuesta puede nombrar el reporte de otro cliente.

Las fichas van completas en el prompt: con un catálogo de este tamaño rinde mejor que la búsqueda vectorial y no hay embeddings que reindexar. Conviene revisarlo por encima de unos 500 reportes.

## Estilos

`zb.css` es el sistema visual del Hub: colores, tipografía, rótulos, botones, campos y formularios. `dashboards.css` es lo propio del catálogo: la tarjeta con su barra de frescura, el marco del embed y los paneles de administración.

Regla para ubicar una regla nueva: **si el próximo módulo del Hub la va a necesitar igual, va en `zb.css`.** Ante la duda, al archivo del módulo, porque promover después es trivial y sacar algo que ya usan tres módulos no lo es.

## Idiomas

Español y portugués. Las cadenas están en el objeto `TEXTOS` de `index.html`; las dos tablas deben tener exactamente las mismas claves. El contenido que escribe el equipo en las fichas no se traduce: queda en el idioma en que se escribió, y el buscador cruza los dos sin problema.

## Embeds

Cualquier cosa que viva en una URL y permita iframes: Power BI, Tableau, Looker Studio y HTML propio publicado en GitHub Pages.

Power BI necesita una URL de tipo `reportEmbed`, la que sale de Archivo → Insertar informe → Insertar en un sitio web o portal. La dirección que se copia de la barra del navegador (`app.powerbi.com/groups/.../reports/...`) **no se puede embeber**: el origen bloquea el iframe.

Cada ficha declara su proporción. `16:9` para Power BI, `4:3` para lienzos cuadrados, `alto` para HTML propio y reportes que scrollean. El embed escala su lienzo fijo para entrar en el marco, así que una proporción equivocada se ve como franjas blancas.
