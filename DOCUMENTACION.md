# Guía Completa de Desarrollo de Temas para CSS Loader (Big Picture & Desktop)
> **Propósito de este documento:** Manual de referencia técnico exhaustivo para cualquier asistente o desarrollador. Contiene toda la arquitectura, especificación de **CSS Loader / Millennium**, árbol DOM de menús, selectores resilientes y la guía paso a paso para crear, depurar y personalizar temas tanto para **Modo Big Picture / Steam Deck** como para **Modo Escritorio (Desktop)**.

---

## Índice General
1. [Arquitectura y Fundamentos de Steam UI](#1-arquitectura-y-fundamentos-de-steam-ui)
2. [Pestañas CEF y Métodos de Inyección](#2-pestañas-cef-y-métodos-de-inyección)
3. [Especificación de `theme.json`](#3-especificación-de-themejson)
4. [Temas Complejos: Parches y Componentes (`patches`)](#4-temas-complejos-parches-y-componentes-patches)
5. [Características Avanzadas y Banderas (`flags`)](#5-características-avanzadas-y-banderas-flags)
6. [Flujo de Pruebas y Depuración CEF (DevTools)](#6-flujo-de-pruebas-y-depuración-cef-devtools)
7. [Arquitectura y Estructura de Steam Desktop](#7-arquitectura-y-estructura-de-steam-desktop)
8. [Estructura DOM de los Menús y Vistas de Escritorio](#8-estructura-dom-de-los-menús-y-vistas-de-escritorio)
9. [Mapa de Selectores Resilientes de Escritorio](#9-mapa-de-selectores-resilientes-de-escritorio)
10. [Recetas y Snippets Prácticos para Escritorio](#10-recetas-y-snippets-prácticos-para-escritorio)
11. [Manual de Instrucciones para el Próximo Agente IA](#11-manual-de-instrucciones-para-el-próximo-agente-ia)

---

## 1. Arquitectura y Fundamentos de Steam UI

Steam (tanto la interfaz de Escritorio como el Modo Big Picture / Steam Deck) está construido como una aplicación web renderizada mediante **Chromium Embedded Framework (CEF)**.
* Toda la interfaz responde a reglas estándar de **HTML5, CSS3 y Flexbox/Grid**.
* La interfaz está dividida en múltiples ventanas y procesos virtuales llamados **pestañas CEF (CEF Tabs)**.
* **CSS Loader** (y herramientas como Millennium) funciona inyectando hojas de estilo CSS en pestañas CEF específicas según lo definido en un archivo maestro llamado `theme.json`.

### Ubicación de los temas:
* **Steam Deck (SteamOS):** `~/homebrew/themes/{NombreDelTema}/`
* **Windows:** Directorio de temas de CSSLoader Desktop (por defecto `C:\Users\{Usuario}\homebrew\themes\{NombreDelTema}\`).
* **Detección:** CSS Loader reconoce como tema cualquier subcarpeta que contenga un archivo `theme.json` (o `theme.css`) en su raíz.

---

## 2. Pestañas CEF y Métodos de Inyección

Para aplicar estilos a un elemento, el archivo CSS debe inyectarse en la pestaña CEF correspondiente donde se renderiza ese elemento.

### 2.1. Atajos de pestañas comunes (*Shorthands*)
CSS Loader proporciona alias para las pestañas más utilizadas:

| Shorthand | Descripción | Archivo típico |
| :--- | :--- | :--- |
| `"desktop"` | Ventana principal del cliente en modo escritorio (`Steam`). Incluye barra superior, biblioteca y estanterías. | `desktop.css` |
| `"desktopchat"` | Ventana y panel flotante de amigos y chats. | `desktopchat.css` |
| `"desktoppopup"` | Ventanas emergentes de escritorio (propiedades, ajustes modales, etc.). | `desktoppopup.css` |
| `"desktopcontextmenu"`| Menús contextuales de clic derecho en escritorio. | `desktopcontextmenu.css` |
| `"store"` | Pestañas web de la Tienda de Steam y Comunidad. | `store.css` |
| `"bigpicture"` | Ventana principal de Big Picture (`Steam Big Picture Mode`). | `bigpicture.css` |
| `"bigpictureoverlay"` | Menús superpuestos sobre Big Picture (menú guía, teclado). | `bigpictureoverlay.css` |
| `"QuickAccess"` | Menú lateral derecho de acceso rápido (botón `...` en Deck). | `quickaccess.css` |
| `"MainMenu"` | Menú lateral izquierdo (botón `STEAM` en Deck). | `mainmenu.css` |

### 2.2. Inyección por Nombre Exacto y Expresiones Regulares
* **Nombre exacto:** Se especifica el título de la pestaña (ej. `"Steam"` o `"Steam Big Picture Mode"`).
* **Regex (expresión regular):** Esencial para pestañas con identificadores variables en cada inicio (ej. `QuickAccess_uid82`, `notificationtoasts_uid14`):
  * `"QuickAccess.*"`
  * `"MainMenu.*"`
  * `"notificationtoasts.*"` (notificaciones emergentes).
* *Nota (Manifest v7+):* Una sola entrada puede inyectarse en múltiples pestañas que coincidan con el patrón al mismo tiempo.

### 2.3. Mapeo de Pestañas (*Tab Mappings*, Manifest v7+)
Para agrupar pestañas y evitar repetir listas en cada archivo CSS:
```json
{
  "tabs": {
    "all_desktop": ["desktop", "desktopchat", "desktoppopup", "desktopcontextmenu"],
    "all_bp": ["bigpicture", "QuickAccess.*", "MainMenu.*"]
  },
  "inject": {
    "shared_theme.css": ["all_desktop"],
    "bigpicture.css": ["all_bp"]
  }
}
```

---

## 3. Especificación de `theme.json`

El archivo `theme.json` es el manifiesto que define metadatos, inyecciones, parches y controles interactivos del tema.

### 3.1. Estructura estándar recomendada:
```json
{
  "name": "Better Big Screen",
  "display_name": "Better Big Screen",
  "author": "TuNombre",
  "version": "v1.0.0",
  "manifest_version": 9,
  "target": "Desktop",
  "description": "Tema personalizado para el cliente de Steam.",
  "flags": [
    "REQUIRE_NAV_PATCH"
  ],
  "inject": {
    "desktop.css": ["desktop"],
    "desktopchat.css": ["desktopchat"],
    "desktoppopup.css": ["desktoppopup"],
    "desktopcontextmenu.css": ["desktopcontextmenu"],
    "bigpicture.css": ["bigpicture"],
    "store.css": ["store"]
  }
}
```

### 3.2. Categorías de Destino (*Targets*):
* `Desktop`: Modificaciones amplias en el cliente de escritorio.
* `Desktop-Library`: Solo la biblioteca de juegos en escritorio.
* `Desktop-Friends`: Solo lista de amigos y chat de escritorio.
* `Desktop-Tweak`: Ajustes puntuales menores.
* `Home` / `Library` / `System-Wide` / `Other`: Para temas enfocados en Big Picture / Steam Deck.

---

## 4. Temas Complejos: Parches y Componentes (`patches`)

Los parches (`patches`) permiten al usuario activar/desactivar opciones desde el menú de CSS Loader:

### 4.1. Checkbox (Interruptor Sí/No)
```json
"patches": {
  "Barra Lateral Compacta": {
    "default": "No",
    "type": "checkbox",
    "values": {
      "Yes": {
        "tweaks/compact_sidebar.css": ["desktop"]
      },
      "No": {}
    }
  }
}
```

### 4.2. Dropdown (Desplegable con opciones)
```json
"patches": {
  "Color de Acento": {
    "default": "Azul",
    "type": "dropdown",
    "values": {
      "Azul": { "colors/blue.css": ["desktop"] },
      "Purpura": { "colors/purple.css": ["desktop"] },
      "Rojo": { "colors/red.css": ["desktop"] }
    }
  }
}
```

### 4.3. Color Picker interactivo (Manifest v3+)
Inyecta directamente una variable CSS sin archivos adicionales:
```json
"components": [
  {
    "name": "Color Primario",
    "type": "color-picker",
    "on": "_",
    "default": "#1a9fff",
    "css_variable": "custom-accent",
    "tabs": ["desktop"]
  }
]
```
En el CSS:
```css
.boton {
  background-color: var(--custom-accent) !important;
}
```

---

## 5. Características Avanzadas y Banderas (`flags`)

* `REQUIRE_NAV_PATCH` *(Manifest v9+)*: **Imprescindible si se usa `display: none;` en elementos enfocables**. Corrige la navegación para que el foco no se quede atrapado en elementos invisibles.
* `/themes_custom/`: Prefijo virtual para cargar recursos locales del tema (imágenes, fuentes):
  ```css
  @font-face {
    font-family: 'MiFuente';
    src: url('/themes_custom/NombreTema/assets/mifuente.woff2') format('woff2');
  }
  .miFondo {
    background-image: url('/themes_custom/NombreTema/images/fondo.png');
  }
  ```

---

## 6. Flujo de Pruebas y Depuración CEF (DevTools)

Steam ejecuta un servidor de depuración remota CEF en el puerto **`8080`**.

### 6.1. Habilitar depuración en Steam:
* En Windows, Steam debe ejecutarse con el argumento:
  ```powershell
  steam.exe -cef-enable-debugging
  ```
  *(O si Steam ya está corriendo, verificar si `steamwebhelper.exe` tiene `--remote-debugging-port=8080`).*

### 6.2. Conexión de Inspección:
1. Abrir Chrome, Edge o Brave y navegar a: `chrome://inspect` o `edge://inspect`.
2. En *Discover network targets*, añadir `127.0.0.1:8080`.
3. Aparecerán todas las pestañas de Steam (*Steam, QuickAccess, SharedJSContext, etc.*) con un enlace **inspect** para abrir DevTools completo.

> [!IMPORTANT]
> En scripts automatizados o PowerShell en Windows 11, usar siempre **`http://127.0.0.1:8080/json`** en lugar de `localhost:8080` para evitar retrasos de resolución de DNS/IPv6.

---

## 7. Arquitectura y Estructura de Steam Desktop

A diferencia de Big Picture (que es una interfaz de pantalla completa orientada a mando), **Steam Desktop** es una aplicación de ventanas para ratón y teclado dividida en varios subsistemas:

```
+--------------------------------------------------------------------------+
|  BARRA SUPERIOR (Titlebar & Navigation)                                  |
|  [Menu: Steam Ver Amigos...] [< >] [TIENDA | BIBLIOTECA | COMUNIDAD...]  |
+-------------------+------------------------------------------------------+
|                   |  ÁREA PRINCIPAL DE CONTENIDO                         |
|  BARRA LATERAL    |                                                      |
|  (Sidebar / List) |  A) Biblioteca Home: Estanterías, recientes, grid.   |
|  - Filtros        |  B) Detalles del Juego: Hero banner, logo, botón     |
|  - Colecciones    |     JUGAR, amigos, logros, actividad, noticias.     |
|  - Lista juegos   |  C) Tienda / Comunidad / Perfil (Webview).           |
|                   |                                                      |
+-------------------+------------------------------------------------------+
|  BARRA INFERIOR (Footer / Status Bar)                                    |
|  [+ Añadir juego]           [Descargas: 12 MB/s]       [Amigos y Chat]   |
+--------------------------------------------------------------------------+
```

### Separación de Pestañas CEF en Escritorio:
1. **`desktop.css` (Pestaña `"desktop"`):** Controla el 90% de la ventana principal: la barra de menú superior, la barra de navegación de pestañas, la barra lateral de juegos y el contenedor de la biblioteca.
2. **`desktopchat.css` (Pestaña `"desktopchat"`):** Controla la ventana de amigos, listas de chat y mensajes. Se ejecuta en un proceso/pestaña separado.
3. **`desktoppopup.css` (Pestaña `"desktoppopup"`):** Controla diálogos modales (Ajustes de Steam, propiedades del juego, gestor de capturas).
4. **`desktopcontextmenu.css` (Pestaña `"desktopcontextmenu"`):** Controla los menús desplegables de clic derecho (al hacer clic en un juego o en la barra de tareas).

---

## 8. Estructura DOM de los Menús y Vistas de Escritorio

Esta sección documenta la jerarquía real de contenedores que genera Valve en el cliente de escritorio:

### 8.1. Raíz de la Aplicación de Escritorio
```html
<body class="DesktopUI ...">
  <div id="popup_target">
    <div class="_1ENHEsrSLcTRtPQFl1F-wL Maximized"> <!-- Shell contenedor -->
      <div class="_3mz8wQ6Q44B8P7pzPP4Iyw">         <!-- Flex principal vertical -->
        <!-- HIJO 1: Barra Superior Completa -->
        <div class="_3Z7VQ1IMk4E3HsHvrkLNgo"> ... </div>
        <!-- HIJO 2: Cuerpo de Biblioteca (Sidebar + Main) -->
        <div class="_1rDh5rXSFZJOqCa4UpnI4z"> ... </div>
        <!-- HIJO 3: Barra Inferior / Notificaciones -->
        <div class="_1_yS5UP7el0aN4vntx3dx"> ... </div>
      </div>
    </div>
  </div>
</body>
```

### 8.2. Barra Superior (Titlebar & Supernav)
Dentro de `._3Z7VQ1IMk4E3HsHvrkLNgo`:
* **Menú clásico de texto (Fila 1):** `._39oUCO1OuizVPwcnnv88no`
  * Contiene los menús: *Steam*, *Ver*, *Amigos*, *Productos*, *Ayuda*.
  * Botones de control de ventana (minimizar, maximizar, cerrar) en la esquina derecha.
* **Navegación principal (Fila 2):** `._3Z3ohQ8-1NKnCZkbS6fvy`
  * Botones Atrás / Adelante (`< >`).
  * Enlaces de sección grande: **TIENDA**, **BIBLIOTECA**, **COMUNIDAD**, **{USUARIO}**.
  * Zona derecha: Botón de Notificaciones (campana), Saldo de cartera, Avatar de perfil.

### 8.3. Biblioteca: Barra Lateral (Sidebar / Game List)
Dentro de `._1rDh5rXSFZJOqCa4UpnI4z`:
* **Contenedor de la barra lateral izquierda:**
  ```css
  [class*="RGNMWtyj73_-WdhflrmuY"], [class*="_3WJCt_OkjPA6npxOtguSt5"]
  ```
* **Buscador y filtros de juegos:**
  * Caja de búsqueda: `[class*="_3xRRJfD2xy95i9NhJxLTp0"]`, input de filtro.
  * Botón de ordenación y filtro rápido: `[class*="_2CmrnGY-Amtd83ScJkFvx2"]`.
* **Lista de juegos (Virtual Scroll):**
  * Cada fila de juego: `[class*="_2TKEazUUS3TlniZfpc8OOe"]` o `[class*="Focusable"]`.
  * Estados del juego:
    * Seleccionado / Activo: `[class*="Selected_"]`.
    * No instalado: Texto atenuado (`opacity: 0.6` o clase de no instalado).
    * En ejecución: Texto verde característico de Steam.

### 8.4. Biblioteca: Vista de Detalles del Juego en Escritorio (App Details)
Cuando se selecciona un juego en la biblioteca:
* **Cabecera del juego (Hero Banner):**
  * Contenedor de imagen: `[class*="_2gZXhRmKUk68pA28"]` / `[class*="NZMJ6g2iVnFsOOp"]`.
  * Imagen de cabecera: `img[class*="HNbe3eZf6H7dtJ042x1vM"]`.
* **Logotipo del juego:**
  * Contenedor exterior: `[class*="A14yd24JRjhFI-Q1ae6tN"]`.
  * Contenedor posicional: `[class*="_2Eh7Soh97QONu_grMi2m66"]`.
  * Imagen del logo: `img[class*="_3NBxSLAZLbbbnul8KfDFjw"]`.
* **Barra de Acción del Juego (Action Bar):**
  * Contenedor de controles: `[class*="_3scbHORkYB7utTUGfkMCC_"]`.
  * **Botón JUGAR:** Botón verde nativo con clase `[class*="_1fHBRg7vFnKszK6EiOdIEY"]` o botón con texto "Jugar" / "Instalar".
  * **Información de horas / sesión:** `[class*="_1YbtIWcfkQJOysLXQbwzRf"]`.
  * **Botón de ajustes (engranaje) y mando:** `[class*="_1thLDT_28YIf6OkgIb6n-4"]`.
* **Pestañas inferiores de la biblioteca:**
  * Pestañas: Actividad, Tus cosas, Comunidad, Información sobre el juego (`[class*="_2-ns_GqXzymHk0jsXBBWJ1"]`).

---

## 9. Mapa de Selectores Resilientes de Escritorio

> [!TIP]
> **Regla de oro:** Valve compila CSS con nombres ofuscados (ej. `_3Z7VQ1IMk4E3HsHvrkLNgo`). Utiliza siempre el selector de atributo parcial `[class*="PREFIJO"]` para que tus estilos sigan funcionando tras actualizaciones menores de Steam.

| Componente | Selector Recomendado | Descripción |
| :--- | :--- | :--- |
| **Cuerpo General** | `.DesktopUI`, `body.DesktopUI` | Raíz de toda la ventana de escritorio. |
| **Barra Superior Completa** | `[class*="_3Z7VQ1IMk4E3HsHvrkLNgo"]` | Contenedor de la barra de título y pestañas. |
| **Menú de Texto (Steam, Ver...)** | `[class*="_39oUCO1OuizVPwcnnv88no"]` | Barra de menús desplegables clásica superior. |
| **Navegación (Tienda, Biblioteca...)** | `[class*="_3Z3ohQ8-1NKnCZkbS6fvy"]` | Enlaces principales de navegación superior. |
| **Barra Lateral de Juegos** | `[class*="RGNMWtyj73_-WdhflrmuY"]` | Panel izquierdo con la lista de juegos. |
| **Buscador de la Biblioteca** | `[class*="_3xRRJfD2xy95i9NhJxLTp0"]` | Input de búsqueda de títulos en la barra lateral. |
| **Fila de Juego en Lista** | `[class*="_2TKEazUUS3TlniZfpc8OOe"]` | Cada elemento individual de la lista de juegos. |
| **Hero / Banner del Juego** | `[class*="NZMJ6g2iVnFsOOp-lDmIP"]` | Contenedor de la ilustración superior del juego. |
| **Logotipo del Juego** | `[class*="_2Eh7Soh97QONu_grMi2m66"]` | Marco de posición del logo sobre el banner. |
| **Imagen del Logotipo** | `img[class*="_3NBxSLAZLbbbnul8KfDFjw"]` | Elemento `<img>` del logo oficial del juego. |
| **Barra del Botón Jugar** | `[class*="_3scbHORkYB7utTUGfkMCC_"]` | Fila que contiene el botón de jugar y estadísticas. |
| **Botón JUGAR** | `[class*="_1fHBRg7vFnKszK6EiOdIEY"]` | El botón principal de iniciar juego / jugar. |
| **Barra Inferior (Footer)** | `[class*="_1_yS5UP7el0aN4vntx3dx"]` | Barra inferior con enlaces de descargas y amigos. |

---

## 10. Recetas y Snippets Prácticos para Escritorio

Guarda estos fragmentos listos para usar en `desktop.css`:

### 10.1. Cambiar la Tipografía Global en Escritorio
```css
/* Aplicar tipografía moderna en todo el cliente de escritorio */
body.DesktopUI,
body.DesktopUI * {
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif !important;
}
```

### 10.2. Modo Oscuro Profundo (OLED / Slate)
```css
/* Fondo oscuro estilizado para la ventana de escritorio */
body.DesktopUI {
    background-color: #0b0e14 !important;
}

/* Barra lateral con transparencia y tono oscuro */
[class*="RGNMWtyj73_-WdhflrmuY"] {
    background-color: rgba(11, 14, 20, 0.95) !important;
    border-right: 1px solid rgba(255, 255, 255, 0.05) !important;
}

/* Barra superior limpia */
[class*="_3Z7VQ1IMk4E3HsHvrkLNgo"] {
    background-color: #080a0f !important;
    border-bottom: 1px solid rgba(255, 255, 255, 0.06) !important;
}
```

### 10.3. Barra Lateral de Juegos Compacta y Minimalista
```css
/* Reducir el padding de los juegos en la lista para ver más juegos a la vez */
[class*="_2TKEazUUS3TlniZfpc8OOe"] {
    padding-top: 4px !important;
    padding-bottom: 4px !important;
    font-size: 13px !important;
}

/* Efecto hover suave al pasar el ratón por encima de un juego */
[class*="_2TKEazUUS3TlniZfpc8OOe"]:hover {
    background-color: rgba(255, 255, 255, 0.06) !important;
    border-radius: 4px !important;
}
```

### 10.4. Logotipo del Juego Proporcional y Uniforme (Lección de Big Picture aplicada a Desktop)
```css
/* Forzar que el logo del juego no sea gigantesco ni se corte */
[class*="_2Eh7Soh97QONu_grMi2m66"] {
    position: absolute !important;
    left: 24px !important;
    bottom: 24px !important;
    top: auto !important;
    right: auto !important;
    width: 480px !important;
    height: 180px !important;
    max-width: 40vw !important;
    max-height: 25vh !important;
    display: flex !important;
    justify-content: flex-start !important;
    align-items: flex-end !important;
}

img[class*="_3NBxSLAZLbbbnul8KfDFjw"] {
    max-width: 100% !important;
    max-height: 100% !important;
    object-fit: contain !important;
    object-position: left bottom !important;
}
```

### 10.5. Botón de Jugar Moderno con Gradiente
```css
/* Estilizar el botón principal de JUGAR */
[class*="_1fHBRg7vFnKszK6EiOdIEY"] button,
[class*="_1fHBRg7vFnKszK6EiOdIEY"] [role="button"] {
    background: linear-gradient(135deg, #1fa2ff 0%, #12d8fa 100%) !important;
    border: none !important;
    border-radius: 8px !important;
    font-weight: 700 !important;
    color: #ffffff !important;
    box-shadow: 0 4px 15px rgba(18, 216, 250, 0.35) !important;
    transition: transform 0.15s ease, box-shadow 0.15s ease !important;
}

[class*="_1fHBRg7vFnKszK6EiOdIEY"] button:hover,
[class*="_1fHBRg7vFnKszK6EiOdIEY"] [role="button"]:hover {
    transform: translateY(-2px) !important;
    box-shadow: 0 6px 20px rgba(18, 216, 250, 0.5) !important;
}
```

---

## 11. Manual de Instrucciones para el Próximo Agente IA

Si eres un asistente de IA que retoma el desarrollo para crear o modificar un tema de **Escritorio**:

### Paso 1: Configurar el manifiesto (`theme.json`)
Asegúrate de que `theme.json` apunte a `desktop.css` con el target `desktop`:
```json
{
  "inject": {
    "desktop.css": ["desktop"]
  }
}
```

### Paso 2: Conectar y Verificar Steam CEF vía WebSocket
Usa este script en PowerShell para verificar qué pestañas de escritorio están activas:
```powershell
$targets = Invoke-RestMethod 'http://127.0.0.1:8080/json'
$desktop = $targets | Where-Object { $_.title -eq 'Steam' } | Select-Object -First 1
Write-Host "Target encontrado: $($desktop.title) - $($desktop.webSocketDebuggerUrl)"
```

### Paso 3: Script de Inyección en Vivo (Desarrollo Rápido)
Para probar estilos al instante sin obligar al usuario a reiniciar Steam cada vez, usa una plantilla como esta:
```powershell
$targets = Invoke-RestMethod 'http://127.0.0.1:8080/json'
$desktop = $targets | Where-Object { $_.title -eq 'Steam' } | Select-Object -First 1
$ws = New-Object System.Net.WebSockets.ClientWebSocket
$cts = New-Object System.Threading.CancellationTokenSource(5000)
$ws.ConnectAsync([Uri]$desktop.webSocketDebuggerUrl, $cts.Token).Wait()

$css = Get-Content '.\desktop.css' -Raw -Encoding UTF8
$b64 = [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($css))

$code = @"
(() => {
    let s = document.getElementById('my-live-desktop-theme');
    if (!s) {
        s = document.createElement('style');
        s.id = 'my-live-desktop-theme';
        document.head.appendChild(s);
    }
    s.textContent = decodeURIComponent(escape(atob('$b64')));
    return 'Desktop CSS inyectado con exito';
})()
"@

$payload = @{ id = 1; method = 'Runtime.evaluate'; params = @{ expression = $code; returnByValue = $true } } | ConvertTo-Json -Compress
$bytes = [System.Text.Encoding]::UTF8.GetBytes($payload)
$ws.SendAsync((New-Object System.ArraySegment[byte] @(,$bytes)), [System.Net.WebSockets.WebSocketMessageType]::Text, $true, $cts.Token).Wait()
$ws.CloseAsync([System.Net.WebSockets.WebSocketCloseStatus]::NormalClosure, 'done', $cts.Token).Wait()
```

### Paso 4: Capturar Pantalla del Cliente de Escritorio
Puedes tomar una captura directamente del canvas de CEF con `Page.captureScreenshot`:
```powershell
$payload = @{ id = 2; method = 'Page.captureScreenshot'; params = @{ format = 'png' } } | ConvertTo-Json -Compress
# Recibir datos base64 y guardar como png para verificar visualmente con view_file.
```

### Reglas Clave a Recordar:
1. **Usa `!important`**: Los estilos internos de Valve a menudo usan selectores anidados profundos con especificidad alta.
2. **Cuidado con `overflow: hidden`**: Modificar alturas sin comprobar los scrollbars nativos puede bloquear el scroll de la lista de juegos.
3. **No toques archivos vacíos sin necesidad**: Si un tema solo modifica escritorio, escribe únicamente en `desktop.css`. Deja `bigpicture.css` intacto para preservar el tema de Big Picture existente.
4. **Codificación de Archivos (ASCII Seguro en Windows)**: `CssLoader-Standalone-Headless.exe` en Windows ejecuta scripts de Python que leen los archivos `.css` y `.json` usando la codificación del sistema local (`cp1252`). Evita caracteres fuera de 7-bit ASCII (tildes, eñes, etc.) en comentarios o strings de los archivos del tema para evitar excepciones de `charmap codec can't decode byte`.
5. **Limpieza de Inyecciones de Prueba**: Al realizar pruebas mediante DevTools o scripts WebSocket, no dejes etiquetas `<style>` manuales fijas en el DOM de Steam. CSS Loader solo controla y sobreescribe sus propios tags gestionados; las etiquetas manuales persistirán y anularán las opciones elegidas por el usuario en la interfaz de CSS Loader.

