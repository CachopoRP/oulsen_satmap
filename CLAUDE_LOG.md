# CLAUDE_LOG — oulsen_satmap

> Cambios realizados por agentes Claude. No borrar entradas.

---

## 2026-09-07 — Reconvertidas las 102 texturas a Enhanced de verdad (crash real en producción) · Claude

**Crash real reportado por Oscar** tras activar el `ensure` de este resource por primera vez en
producción (ver `qbx_phone/CLAUDE_LOG.md` entrada 47/48 y issue `CachopoRP/trabajos#1`): el cliente
crasheaba (`GTA5_Enhanced.exe`, `BREAKPOINT_80000003`) poco después de conectar. Aislado en vivo por
Oscar (activar/desactivar recursos uno a uno): con `oulsen_satmap` desactivado el server arranca
bien, con él activo crashea — confirmado, no era `burgershot-map` (otro crash distinto y ya
conocido, ver `RSCoroner/docs/METHODOLOGY.md`, que también salió en los mismos `.dmp` pero es un bug
aparte).

### Diagnóstico con `RSCoroner`

Este resource es un fork literal del repo público de `Oulsen` — su propio README confirma que es un
mod de **2021, exclusivamente Legacy**, sin ninguna mención a Enhanced/gen9, pensado para editarse
con OpenIV (herramienta de la era Legacy). El commit `e42b745` ("Aplana estructura del resource y
activa variante NoGrid") solo reorganizó carpetas, nunca convirtió nada a Enhanced.

`CwToolkit.exe check` decía que los 78 ficheros de `stream/` cargaban bien — pero eso solo prueba
que la estructura RPF es parseable bajo el flag `IsGen9=true`, no que el contenido esté realmente
convertido (la misma lección que ya costó un falso positivo en el caso `burgershot-map`). La prueba
real, `ytd-diff` entre el fichero desplegado y una reconversión de referencia:

```
minimap_0_0.ytd: 1 texturas -- minimap_0_0.ytd: 1 texturas
=== Diferencias ===
  solo en A: tex=minimap_0_0 4096x4096 levels=1 format=D3DFMT_DXT5 g9format=UNKNOWN g9tile=Auto
  solo en B: tex=minimap_0_0 4096x4096 levels=1 format=D3DFMT_DXT5 g9format=BC3_UNORM g9tile=Auto
```

**`g9format=UNKNOWN` en las 102 texturas** (`stream/`, `grid/`, `nogrid/`) — nunca tuvieron el
formato Enhanced puesto de verdad. Esto es casi con toda seguridad lo que hacía saltar el
`BREAKPOINT_80000003`: el motor se encuentra un enum de formato de textura gen9 sin valor válido en
vez de fallar con gracia.

### `CwToolkit` no soportaba `.ytd` — extendido hoy

`gen9-convert` de `RSCoroner` solo convertía `.ydd`/`.ydr` (el caso `burgershot-map` era un MLO con
shaders). Este resource es puramente texturas (`.ytd`) — `CodeWalker.Core` sí expone
`TextureDictionary.EnsureGen9()`, solo hacía falta cablearlo. Añadido en
`RSCoroner/src/CwToolkit/Program.cs` (`CmdGen9Convert`), mismo patrón que `.ydd`/`.ydr`: leer,
`EnsureGen9()`, guardar, releer para verificar. Detalle en el propio repo `RSCoroner`.

### Arreglado

Reconvertidas las 102 texturas (`stream/` 78, `grid/` 12, `nogrid/` 12) con
`CwToolkit.exe gen9-convert` usando la lógica real de CodeWalker (no `AlchemistCli`, que tampoco
soporta `.ytd` para este flujo). Los 78 `.ydd` diminutos (461 bytes, probablemente placeholders del
propio mod) se reconvirtieron también de paso — pasan de 461 a 506 bytes, mismo indicio de que
tampoco estaban en formato gen9. Todos los ficheros resultantes cargan sin excepción
(`CwToolkit check`, 102/102 OK).

### `stream_enhanced/` — copia de referencia, explícitamente marcada

Además de sustituir en sitio `stream/`, `grid/` y `nogrid/` por sus versiones reconvertidas, se
añadió `stream_enhanced/` como copia exacta de la variante activa ya convertida (mismo criterio que
`grid/`/`nogrid/`, que ya eran variantes con nombre propio en este mismo repo) — para que quede
explícito y a mano cuál es la versión Enhanced de verdad, sin depender de mirar el historial de git
o fiarse de que nadie vuelva a pisar `stream/` con un backup viejo sin darse cuenta. **No sustituye
a `stream/`** como carpeta activa — FXServer solo auto-detecta streaming en una carpeta llamada
literalmente `stream/`, así que esa sigue siendo la que de verdad carga el juego; `stream_enhanced/`
es la copia de referencia/rollback.

### Sin probar en vivo todavía

Reconversión estática verificada (parseo + comparación de formato de textura), pero **no se ha
confirmado en el juego que esto elimina el crash real** — eso solo lo puede confirmar quien tenga
el cliente abierto. Antes de dar esto por cerrado: volver a activar `ensure oulsen_satmap` en
producción (ya estaba activo desde hoy, alguien lo desactivó a mano para aislar el bug) y probar
varias conexiones/desconexiones reales.
