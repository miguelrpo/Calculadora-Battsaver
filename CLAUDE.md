# CLAUDE.md — Calculadora Battsaver

Instrucciones de contexto para trabajar en `Calculadora_Battsaver.html`, la calculadora interactiva de repago/ahorro/margen de Battsaver. Este archivo es distinto del `CLAUDE.md` de instrucciones de documentos (.docx/.pptx/.xlsx) del proyecto — este gobierna específicamente el desarrollo de esta pieza de software.

---

## 1. Qué es este archivo

Un **HTML autocontenido de una sola página** (~870 líneas: CSS inline en `<style>`, JS vanilla inline en `<script>`, sin build step, sin dependencias externas salvo la fuente de Google Fonts). Se abre directamente en el navegador o se sube como artifact. No hay backend: todo el cálculo ocurre en el cliente.

Uso previsto: **herramienta de venta en vivo**, operada por un vendedor de Battsaver frente a un cliente (Directo, Flotas, Concesionario o Gran distribuidor) para mostrar ROI/payback o margen de reventa en tiempo real, ajustando inputs mientras conversan.

No hay framework (no React/Vue), no hay `package.json`, no hay tests automatizados más allá de verificación manual con Playwright (capturas de pantalla + chequeo de errores de consola) antes de cada entrega.

---

## 2. Modelo de negocio (fuente de verdad)

### SKUs (3 referencias)
```js
var SKU = [
  {name:"12V · 10W", apps:"Motos, carros pequeños, boogies, cuatrimotos", price:399900},
  {name:"12V · 20W", apps:"Camionetas, camiones, maquinaria, botes",       price:499900},
  {name:"24V · 20W", apps:"Camiones europeos y chinos",                     price:549900}
];
```
Precios en COP, precio de lista (sin descuento de canal).

### Canales (4, con índices fijos)
```js
var C_DIRECTO=0, C_FLOTAS=1, C_CONCE=2, C_DISTRIB=3;
var CHANNEL = [
  {name:"Directo",       disc:0.00, canResell:false, sellsTo:[]},
  {name:"Flotas",        disc:0.25, canResell:false, sellsTo:[]},
  {name:"Concesionario", disc:0.35, canResell:true,  sellsTo:[C_FLOTAS, C_DIRECTO]},
  {name:"Gran distrib.", disc:0.55, canResell:true,  sellsTo:[C_CONCE, C_FLOTAS, C_DIRECTO]}
];
```
Descuentos sobre precio de lista, según Plan Comercial §6.1. `sellsTo` define a qué canales puede revenderle cada revendedor (un solo escalón de cadena, nunca a otro distribuidor).

### Modalidad de uso — **forzada por canal**, no elegible por el usuario
| Canal | Modalidad | Forzado por |
|---|---|---|
| Directo | Mantenedor fijo | `forcedMode()` |
| Flotas | Mantenedor fijo (+ mínimo 100 unidades) | `forcedMode()` + `MIN_FLOTAS` |
| Concesionario / uso propio | Mantenedor de patio | `forcedMode()` |
| Concesionario / reventa | — (vista de margen, no aplica modo) | `isResale()` |
| Gran distribuidor | — (vista de margen, siempre revende) | `isResale()` |

El selector visual de modo (`#mode`, botones Fijo/Patio) **nunca se muestra hoy** porque los 4 canales están forzados — pero el mecanismo de elección libre sigue en el código (`forcedMode()` devuelve `null`) por si algún canal se libera en el futuro. No borrar ese mecanismo sin pedir confirmación explícita.

- **Mantenedor fijo**: 1 equipo por vehículo, permanente. Ahorro = reposiciones de batería evitadas al año × precio de batería. La vida de la batería se extiende `EXT_FACTOR = 2×` con el mantenedor (manual dice 2–3×, se usa el conservador).
- **Mantenedor de patio**: 1 equipo por vehículo del **pico máximo simultáneo** detenido (no por vehículo total). El equipo solo se mueve cuando el vehículo se vende/reactiva. Ahorro = (pico × rotación de inventario/año) × Z% (baterías que se habrían cambiado sin Battsaver, slider 5–60%, default 20%).
- ~~Rotación activa~~ — **eliminada** por decisión del usuario (era un tercer modo con capacidad X÷Y y costo de mano de obra; no reintroducir sin que se pida explícitamente).

### Reventa / margen
- El precio de reventa **nunca se escribe a mano**: sale de `listPrice × (1 - disc del comprador elegido)`.
- Sin campo de cantidad: el margen se muestra **por equipo** (unitario), no total. Esto fue decisión explícita del usuario.
- Selector "¿A quién le revendes?" (`#resellTo`) se construye dinámicamente según `CHANNEL[canal].sellsTo`.
- KPI de "Rentabilidad sobre costo" (markup) fue **eliminado** por pedido del usuario — solo queda "Margen % sobre venta".

### Constantes clave
```js
var EXT_FACTOR = 2;     // extensión de vida útil de batería con mantenedor
var DEVICE_LIFE = 8;    // años de vida del equipo (horizonte de ROI en modo fijo)
var MIN_FLOTAS = 100;   // pedido mínimo Flotas, forzado en el cálculo Y en el input (min attr + blur)
```

---

## 3. Reglas de UI/UX establecidas (no revertir sin pedirlo)

- **Un solo precio visible por vista**, ubicado siempre **debajo del selector de tipo de vehículo/SKU** (`#priceField` → `#pricetag`), tanto en repago como en reventa. En reventa, justo debajo aparece `#resellHint` con lo que paga el comprador elegido — sin duplicar cifras.
- El tag de canal (`#chtag`) muestra solo el descuento ("35% de descuento sobre lista"), no repite el nombre del canal (ya está marcado en el botón activo).
- Cuando la modalidad está forzada, se muestra **una sola caja fusionada**: "Modalidad fija para [Canal] — [explicación del modo]" (`#modehelp`), no dos cajas separadas.
- Lenguaje: **tú/tu**, nunca "usted/su". Confirmado explícitamente contra el Manual de Marca ("tu batería siempre está protegida"). Revisar cualquier string nuevo contra este registro.
- "Battsaver" en prosa (no "BattSaver"); el wordmark del logo sí va en mayúsculas por ser logotipo.
- El medidor de "composición del precio" en reventa rellena en color de acento la porción de **margen** (lo positivo para el vendedor), no el costo — cuidado si se toca esa lógica, es fácil invertir la semántica sin querer.
- Cuando el repago **no ocurre** dentro de `DEVICE_LIFE`, el hero debe decirlo explícitamente ("No se repaga en la vida del equipo"), nunca mostrar solo un número de años grande sin ese matiz — fue un bug corregido.
- El ROI puede ser negativo; el signo "+" solo se antepone si `roiPct >= 0`. No volver a concatenar "+" incondicionalmente (causaba "+-52%").

---

## 4. Identidad visual

Fuente de verdad: `Manual_de_Marca__BATTSAVER.pdf` (17 páginas, en el proyecto). Puntos ya verificados página por página:

- **Colores oficiales de marca**: navy `#08344D`, carbón `#2E2E2E`, negro, blanco. El manual muestra explícitamente un logo en cian como ejemplo de "Nunca modifiques los colores institucionales" — cualquier verde/teal/mint usado en la app es una **decisión de acento comercial del usuario**, no un color de marca oficial. Actualmente el acento vigente es **teal `#0E8C7F` / mint `#C2E9D5`** (revertido desde un intento de ámbar que se probó y se descartó). El ámbar (`--amber`/`--amber-d`/`--amber-bg`) se conserva **solo** para el bloque de advertencia (`.warn`), donde es semánticamente correcto (ícono ⚠️ del manual).
- **Tipografía oficial**: **Cloud** (Cloud Bold para títulos, Cloud Light/Regular para texto corrido). No existe en Google Fonts ni tiene CDN pública confiable. Se usa **Plus Jakarta Sans** como sustituta temporal vía Google Fonts, con un `@font-face` placeholder ya declarado apuntando a `src:local("Cloud")` — **pendiente**: el usuario va a enviar el archivo de fuente oficial; cuando llegue, reemplazar el `src:local(...)` por `url()` al `.woff2` y no tocar nada más (toda la app hereda de `body{font-family:"Cloud","Plus Jakarta Sans",...}`).
- **Logo**: horizontal blanco oficial, incrustado como base64 en el `<header>` (no depender de una carpeta `assets/` externa — el archivo debe seguir siendo un único HTML portable). Original en `/mnt/user-data/uploads/1782936656692_LOGO_BATTSAVER_HORIZONTAL_-_WHITE_-_CUT.png`.
- **Voz de marca**: tú/tu (ver §3).

---

## 5. Layout responsivo

- Desktop: grid de 2 columnas (`Tus datos` | resultados).
- Móvil (`@media max-width:640px`): header apilado, botones de canal/SKU/reventa en grid 2×2, y una **barra sticky inferior** (`#stickyBar`) que muestra el resultado clave (ahorro/payback o margen) en todo momento y hace scroll a resultados al tocarla. Esto existe porque en móvil los resultados quedaban fuera de vista tras cada ajuste de input.
- Impresión (`@media print`): hoja de estilos dedicada que oculta la sticky bar y el botón de imprimir, quita sombras, evita cortes de tarjeta a media página. Se dispara con el botón "⤓ Guardar PDF" (`#printBtn` → `window.print()`).

---

## 6. Convenciones de desarrollo

- **Sin build step.** Cualquier cambio se edita directo en el HTML.
- **Formato de dinero**: `cop()` da formato completo ("$399.900"), `copShort()` abrevia a millones ("$1,8 mill."). El input de precio de batería (`#batt`) es `type="text"` con formateo en vivo de miles (`fmtMoneyInput`) y se lee con `moneyVal()` (parsea dígitos, ignora puntos) — no volver a `type="number"` ahí, se decidió así para evitar que el usuario vea "450000" sin separador.
- **Validación antes de entregar**: cada cambio se verifica con Playwright — render headless, recorrer los 4 canales + toggle de intención + modos, capturar pantallas, revisar `console.error`/`pageerror`, y contar llaves/paréntesis balanceados como chequeo rápido de sintaxis antes de guardar.
- **Archivo de salida**: siempre `Calculadora_Battsaver.html` en la raíz del repo. El versionado ya lo lleva git (historial de commits), no el nombre del archivo — no reintroducir sufijos `_v1`/`_v2`/etc. Versiones anteriores previas al repo (v1–v4, 3modos, v5) quedan solo en el historial de git, no como archivos sueltos.

---

## 7. Repositorio y publicación

- **Repo**: [github.com/miguelrpo/Calculadora-Battsaver](https://github.com/miguelrpo/Calculadora-Battsaver) — **público**. Se evaluó dejarlo privado, pero GitHub Pages en repos privados requiere un plan de pago (Pro/Team/Enterprise); la cuenta `miguelrpo` está en plan Free. El usuario aceptó hacerlo público sabiendo que, de todos modos, el HTML es 100% client-side: cualquiera con el link a la página publicada puede ver precios y descuentos por canal con "Ver código fuente", sin importar la visibilidad del repo.
- **GitHub Pages**: activo desde la rama `main`, carpeta `/`. **Dominio personalizado configurado por el usuario directamente en GitHub** (Settings → Pages), lo que creó un archivo `CNAME` en el repo con el contenido `finanzas.battsaver.com.co`. URL vigente: `https://finanzas.battsaver.com.co/Calculadora_Battsaver.html` (el dominio `miguelrpo.github.io/Calculadora-Battsaver/...` sigue funcionando pero redirige 301 al dominio personalizado). La raíz del dominio (`finanzas.battsaver.com.co/`) **da 404 a propósito** — no hay `index.html`, decisión explícita del usuario de no crear uno para no duplicar el archivo. No agregar `index.html` ni tocar/borrar el `CNAME` sin que lo pida explícitamente.
- **Remote via SSH**: el `~/.ssh/config` del usuario estuvo corrupto temporalmente (contenía la salida de un comando `brev` fallido en vez de config real), lo que rompía las operaciones git por SSH — por eso el remote `origin` se usó en HTTPS durante un tiempo. Ya fue arreglado por el usuario y el remote `origin` vuelve a apuntar a `git@github.com:miguelrpo/Calculadora-Battsaver.git`. No tocar `~/.ssh/config` (es un archivo fuera del proyecto, del sistema del usuario) sin que lo pida explícitamente.
- **push**: cambios en `Calculadora_Battsaver.html` y `CLAUDE.md` se commitean y suben a `main` (no hay ramas de feature ni PRs en este flujo por ahora); `main` se despliega automáticamente vía Pages en cada push.

---

## 8. Pendientes conocidos

1. **Fuente Cloud oficial** — el usuario la enviará; reemplazar el `@font-face` cuando llegue (ver §4).
2. **Aviso de parque pequeño en Flotas**: con el mínimo de 100 unidades forzado, un parque real mucho menor puede dar ROI negativo (matemáticamente correcto, pero vale la pena evaluar si merece una advertencia visual explícita cuando `veh` real < `MIN_FLOTAS` por un margen grande).
3. **Botón de reinicio** de todos los inputs a sus valores por defecto — mencionado como mejora, no implementado.
4. **Accesibilidad de segmented controls**: los selectores de botones (`.seg`) no tienen `role="radiogroup"`/`aria-checked`, y no hay estado de foco visible por teclado en los botones (sí en los inputs de texto). Impacto bajo para uso comercial en vivo, pendiente si se prioriza.
5. **Escenario comparativo Concesionario** (uso propio vs. reventa lado a lado) — sugerido como posible mejora de argumento de venta, no implementado.

---

## 9. Qué NO hacer sin confirmar explícitamente

- No reintroducir el modo "Rotación activa" (fue removido a propósito).
- No volver a mostrar cantidad/total en la vista de reventa (es intencionalmente unitaria).
- No cambiar el acento teal/mint por otro color sin que el usuario lo pida — ya hubo una iteración completa a ámbar que se revirtió.
- No usar colores fuera de navy/carbón/negro/blanco/teal-mint/ámbar(solo warnings) sin verificar contra el Manual de Marca.
- No cambiar "tú/tu" de vuelta a "usted/su".
- No volver el repo privado sin antes avisar que eso apaga GitHub Pages (plan Free no soporta Pages en repos privados) — ver §7.
- No tocar `~/.ssh/config` del usuario sin que lo pida explícitamente — ver §7.
