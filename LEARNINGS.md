# LEARNINGS — GIMP MCP

Bitácora de cosas aprendidas trabajando con este proyecto (plugin + server MCP de GIMP).
Ir agregando arriba de todo lo nuevo. Fechas en formato AAAA-MM-DD.

---

## Plugin / server MCP

### Recargar código del plugin = reiniciar GIMP ENTERO
- El proceso del plugin es persistente (mantiene contexto Python y el socket). *Tools > MCP > Restart MCP Server* **solo reinicia el hilo del socket, NO reimporta el `.py` desde disco**.
- Para cargar cambios de `gimp-mcp-plugin.py`: **cerrar GIMP por completo → reabrir → Tools > MCP > Start MCP Server.**

### Hay DOS copias del plugin — sincronizar siempre
- El que corre está en `~/.config/GIMP/3.2/plug-ins/gimp-mcp-plugin/gimp-mcp-plugin.py` (ruta depende de la versión: `3.2`, `3.4`, …). Es una **copia separada** del repo.
- Al editar: parchear el repo y `cp` a esa ruta (+ `chmod +x`). Verificar con `diff` antes de reiniciar GIMP.

### Loops pesados por el socket MATAN el server (2026-07-07)
- Dibujar miles de shapes con `call_api` en un solo request (cada `select_*` + `edit_fill` es un round-trip) satura el socket → **timeout del cliente y el hilo de accept muere**. El puerto 9877 queda sin escuchar.
- Mitigaciones:
  - **`IMG.undo_disable()` antes de loops pesados y `IMG.undo_enable()` después** → mucho más rápido y menos memoria (evita que GIMP guarde cada paso en el historial).
  - **Partir el dibujo en tandas chicas** (varias `call_api`), no miles de fills en un request.
  - Bajar densidad (spacing mayor, shapes más grandes). Preferir menos operaciones grandes que muchas chiquitas.
  - No llamar `Gimp.displays_flush()` dentro del loop; una sola vez al final.

### Bugs de arranque del socket (arreglados 2026-07-07)
- Cuando el loop de `accept()` moría por `OSError`, cortaba pero **dejaba `self.running = True`** → *Start MCP Server* quedaba como no-op permanente ("already running") con el puerto cerrado.
- `_restart_server` re-bindeaba el socket pero **nunca arrancaba un hilo nuevo de `accept()`** → el *Restart* tampoco servía.
- Fix aplicado: `_start_server_thread` ahora setea `running=False` al salir del loop; `_restart_server` frena el loop viejo, espera y **lanza un hilo fresco** de `_start_server_thread`. Con esto, un socket caído se recupera con *Restart* (sin reabrir GIMP), siempre que GIMP siga vivo.
- Diagnóstico útil sin el socket: `ss -ltnp | grep 9877` (¿escucha?), `ps -eo pid,stat,cmd | grep gimp` (STAT `Sl+` = idle/responde, no congelado).

### export_image ignoraba format/quality (arreglado 2026-07-07)
- El bug estaba en el **plugin** (`_export_to_path`), no en el server (el server solo reenvía).
- En GIMP 3.2 los procedimientos son `file-<fmt>-export`, **no** `file-<fmt>-save` (esos no existen → caía al fallback PNG y escribía PNG con la extensión pedida).
- Escala de `quality` por procedimiento: `file-jpeg-export` es **0.0–1.0** (pasar `quality/100`); `file-webp-export` es **0.0–100.0** (pasar tal cual); `file-png-export` y `file-tiff-export` no tienen `quality`.

---

## Tips de dibujo / API GIMP 3.0 vía call_api

- El **contexto Python del plugin es persistente**: definir helpers/globals (`newlayer`, `fillrect`, colores) una vez y reusarlos en `call_api` siguientes.
- `Gimp.Layer.new(img, name, w, h, Gimp.ImageType.RGBA_IMAGE, opacity, mode)` + `img.insert_layer(layer, None, pos)` (pos `0` = arriba del stack).
- Color: `c = Gegl.Color.new('black'); c.set_rgba(r,g,b,a)` (valores 0–1). Rellenar: `img.select_rectangle(op,x,y,w,h)` + `Gimp.context_set_foreground(c)` + `drawable.edit_fill(Gimp.FillType.FOREGROUND)`.
- **`select_polygon(operation, segs)`** — firma de 2 args (sin `num_segs`); `segs` = lista plana `[x1,y1,x2,y2,...]`.
- **`select_by_color` (tool MCP) es GLOBAL**, no contiguo: selecciona TODOS los píxeles del color en la imagen. Para "varita mágica" contigua usar `call_api`: `img.select_contiguous_color(op, drawable, x, y)` con `Gimp.context_set_sample_threshold(0..1)`.
- Glow real = capa con la forma brillante en modo **Screen/Addition** + **Gaussian blur** fuerte (a veces varios pases). Un blur chico en una imagen 1920px casi no se nota; para halos difusos usar radios grandes (100–300).
- **Viñeta no destructiva**: capa blanca + `apply_vignette` + modo **Multiply** + opacidad moderada.
- `get_image_bitmap` renderiza la transparencia como **negro** en el preview (sirve para ver halos claros; para evaluar sobre fondo claro, componer aparte).

### Gotchas de dibujo por capas (2026-07-08, sesión cyberpunk)
- **`edit_fill` sobre una selección VACÍA rellena TODO el drawable.** Si `select_rectangle` recibe un rect **totalmente fuera del canvas** (ej. `x` negativo porque un edificio arranca en `x=-20`), la selección queda vacía y el fill pinta la capa entera → "lavado" de color. **Solución:** clampear/skipear en el helper de fill (si `x<0`: `w+=x; x=0`; etc.; si queda `w<=0` return). Pasó también con coords fuera por derecha/abajo.
- **`args[1]` de `call_api` DEBE ser una lista anidada** de strings de código: `["exec-console", ["linea1","linea2"]]`. Si mandás las líneas como elementos sueltos de `args` (`["exec-console","linea1","linea2"]`), el plugin itera el string **carácter por carácter** y tira `name 'p' is not defined` (la 'p' de `print`). Síntoma inconfundible: errores de nombres de 1 letra.
- **`IMG.undo_disable()` rompe las selecciones** en esta versión: con el undo desactivado, `select_rectangle` no se aplica y cada `edit_fill` pinta toda la capa. NO usar `undo_disable` alrededor de loops que dependen de selecciones. (Para acelerar/no colgar, mejor **partir en tandas chicas**.)
- **El tool `get_pixel_color` NO es confiable para muestrear una capa puntual** (parece muestrear el composite/otra capa): reportó "relleno" donde la capa estaba transparente. Para verificar píxeles de una capa usar `call_api`: `layer.get_pixel(x,y).get_rgba()`. Para verificar el resultado final, **renderizar** con `get_image_bitmap`, no fiarse de tools de muestreo.
- **Cargar mucho el socket en UN request lo cuelga / lo mata.** Miles de `fillrect` en una sola `call_api` → timeout del cliente (y a veces muere el hilo del socket). Mantener cada request en ~300–500 fills y **partir en varias llamadas**. El cuello de botella es GIMP ejecutando los PDB in-process, no el round-trip.
- **`reorder_layer`**: en la práctica `new_position` se comportó como índice **desde arriba** (0 = tope), aunque el docstring diga "0 = bottom". Verificar con `list_layers` (index 0 = capa superior).

---

## Recorte de fondo / rembg (2026-07-07)

- `rembg` acá: el CLI tiene deps opcionales rotas (`filetype`, `watchdog`). **Usar la API Python** (`from rembg import remove, new_session`) evita esos imports.
- Comparación de modelos sobre la foto del gatito (gato + florcita):
  - **u2net**: bueno, algo liso, pierde algún mechón.
  - **isnet-general-use**: **el mejor balance** — más pelo fino, bigotes crispos, poco halo, y **conserva la flor**. Ganador para escenas con varios elementos.
  - **birefnet-general**: mejor pelo puro, pero **descarta objetos separados** (borró la planta/flor) y deja halo azul. OK solo si hay UN sujeto claro.
- La **selección por color/fuzzy de GIMP no sirve** cuando el sujeto comparte tonos con el fondo (gato azulado sobre pared azulada → la varita se filtra y se come al gato). Para recortes finos de pelo, rembg gana lejos.
- **Defringe / decontaminación de color** (script propio, `scratchpad/defringe.py`): propagar el color del foreground opaco hacia el anillo de borde **sin tocar el alpha** (preserva la forma del pelo) + **despill de azul** (capar canal B ≤ max(R,G) en la banda de borde). Verificar con métrica objetiva "azul de más = B - max(R,G)" antes/después. Ojo: si la foto tiene cast frío, propagar color solo NO baja el azul (el sujeto ya es azul) — hace falta el despill.

---

## Entorno / misc

- El **Bash tool corre en sandbox** con vista de FS que a veces no ve `~/Downloads`. Si "desaparecen" archivos que GIMP sí tiene abiertos, es el sandbox o que el usuario los movió. Usar `dangerouslyDisableSandbox: true` para operar sobre el FS real, y `find` para relocalizar.
- El **timeout del Bash tool es 120s por default** aunque uses `timeout 900` en el comando (el tool corta antes). Para descargas largas (modelos rembg ~170MB–1GB) correr en background (`run_in_background`).
- `Date.now()`/`Math.random()` no existen en scripts de Workflow, pero en `call_api` (Python real dentro de GIMP) `random` sí está disponible — usarlo para estrellas, ventanas, jitter, etc.
