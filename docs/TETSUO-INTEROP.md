# Viabilidad de implementar partes del backend en tetsuo

Estado verificado el 2026-09-06. Contraparte desde Videodrome del informe
`docs/BACKEND-READINESS.md` de `ser356/tetsuo`.

## Contrato verificado del compilador

La revisión fijada en `native/TETSUO_REV` es
`6e62b6f9d3178374b1f3cd43a47c7358ca2899a2`. Sus binarios bootstrap se
verifican con `native/TETSUO_SHA256` antes de instalarse.

| Invocación | Salida comprobada | Startup |
|---|---|---|
| `--emit=obj` | Ensamblador Mach-O ARM64 enlazable; `clang -c -arch arm64` produce un objeto Mach-O con ambos símbolos ABI | No |
| `--emit=obj --target=linux` | Ensamblador ELF ARM64 enlazable con pila no ejecutable | No |
| `--emit=obj --target=windows-x64` | Ensamblador COFF x64 enlazable | No |

`obj` nombra un modo de componente. Los tres modos escriben ensamblador privado
del build; Cargo lo ensambla y archiva dentro de `OUT_DIR`. Ningún backend
inyecta startup en este modo. Los componentes de Videodrome son autocontenidos.

La matriz habilitable con este contrato queda limitada a macOS ARM64, Linux
ARM64 y Windows x64. macOS x64, Windows ARM64 y mobile conservan Rust hasta que
Tetsuo produzca componentes enlazables para esos targets.

## Resumen

Dos núcleos del backend son componentes Tetsuo reales: el parser de nombres de
release y el cálculo OpenSubtitles moviehash. Cargo los compila desde `.tt` y
los enlaza dentro del proceso en macOS ARM64, Linux ARM64 GNU y Windows x64
MSVC. Rust queda como implementación transitoria de otros triples y como
oráculo diferencial en tests; no existe fallback silencioso en targets Tetsuo.

## Por qué el resto no

Tetsuo es un lenguaje de sistemas sin libc, sin heap general, sin async, sin
hilos, sin red y sin punto flotante. Sus backends actuales producen ejecutables
para macOS ARM64, Linux ARM64, Windows ARM64 y Windows x64; esto no implica que
todos produzcan objetos enlazables con Rust.

Cómo cae eso sobre este backend, contando LOC de todo fichero que toca cada
familia:

| Familia | LOC | % |
|---|---|---|
| `async` / `.await` | 23 908 | 82 % |
| Red (`reqwest`, `axum`, `hyper`) | 21 560 | 74 % |
| `f32` / `f64` | 23 805 | 82 % |
| `String` / `Vec` / `HashMap` / `Box` | 27 775 | 95 % |

Ficheros con cero dependencias de las cuatro: cinco (931 LOC), y cuatro de
ellos son pegamento (`main.rs`, `gui/commands/mod.rs`, `kiosk/mod.rs`,
`embed_player.rs`). Además de `release_name.rs`, se puede aislar el núcleo puro
de moviehash aunque el I/O asíncrono que obtiene sus bloques permanezca en Rust.

Videodrome también construye macOS x64 y targets móviles. No existe backend x64
SysV/macOS ni objeto para ARM64 Windows. Por eso cualquier módulo portado
mantiene una implementación Rust tras un `cfg` para esos targets.

## Lo que sí encaja: `src/torrents/release_name.rs`

Es tokenizar bytes y decidir sobre la estructura — justo lo que tetsuo lleva
haciendo en su propio lexer y parser. No usa async, red, flotantes, serde,
procesos ni reloj; sus 34 `String`/`Vec` son todos reubicables a arena.
`lib/string.tt` y `lib/parse.tt` de tetsuo ya cubren `string_eq`,
`string_has_prefix`, `string_find_byte` y `parse_u64`.

La ABI ya casa: tetsuo pasa parámetros en x0–x7, retorna por x0, usa x9–x15
como scratch (todos caller-saved en AAPCS64) y expande un `str` a dos
registros `(ptr, len)`. Un `fun f(p: *u8, n: u64) -> u64` es binariamente un
`extern "C" fn(*const u8, u64) -> u64`.

**Dos cosas nos afectan directamente:**

1. **`normalize_title` no es portable sin pérdida.** Usa
   `char::is_alphanumeric()` sobre codepoints Unicode, y el test
   `normalize_title_preserves_cjk` lo blinda con títulos chino, japonés y
   coreano. Tetsuo no decodifica UTF-8. La aproximación viable (todo codepoint
   ≥ U+0080 cuenta como alfanumérico) pasa esos tests pero **diverge en
   puntuación no-ASCII**: guiones tipográficos, comillas CJK y `·` seguirían
   siendo separadores en Rust y dejarían de serlo en tetsuo. Esa función
   alimenta el matching de títulos en `torrents/mod.rs:602`, `:676` y en
   `gui/mod.rs:1099`/`:1281` — una divergencia ahí degrada búsquedas en
   silencio.

2. **Nuestros 34 tests son el oráculo.** Cualquier port se valida ejecutando
   ambas implementaciones sobre el mismo corpus y comparando campo a campo.

## Segundo componente: OpenSubtitles moviehash

`subtitles::compute_moviehash` conserva validación y formato hexadecimal en
Rust. La suma wrapping de los dos bloques de 64 KiB se ejecuta en
`tt_moviehash_v1`, que solo recibe punteros ya validados y no hace I/O ni
asignaciones. Los vectores conocidos, overflow y comparación diferencial siguen
siendo parte de la suite Rust.

## Siguiente candidato

`src/keyframes.rs` (838 LOC): el parseo EBML/MKV (`SeekHead`, `Cues`, varints)
y las tablas `stss`/`stts` de MP4 son byte-crunching puro y encajarían bien.
Pero hoy están entretejidos con `reqwest` Range requests (10), `async` (23) y
`f64` para timestamps (19).

Extraer un núcleo puro `fn parse_cues(bytes: &[u8]) -> Vec<u64>` separado del
fetch HTTP **tiene valor por sí mismo**, con o sin tetsuo: haría testeable el
parser de contenedores sin red. Si se hace, queda además listo para portar.

## Integración actual

- `native/release_name.tt` y `native/moviehash.tt` son las únicas fuentes de
   producción de ambos núcleos en los tres triples habilitados.
- `build.rs` invoca el compilador fijado, escribe ensamblador y objetos en
   `OUT_DIR`, valida símbolos y ausencia de startup, enlaza una biblioteca
   estática y activa ambos `cfg` tras completar el proceso.
- `native/TETSUO_REV` y `native/TETSUO_SHA256` fijan revisión y bootstrap.
- `scripts/bootstrap-tetsuo.sh` instala el compilador sin generar fuentes ni
   artefactos dentro del repositorio.
- CI compila desde `.tt`, inspecciona símbolos, ejecuta el smoke ARM64 y corre
   tests diferenciales en los runners nativos.
- Un resultado ABI inválido aborta en targets Tetsuo; no cambia de
   implementación durante la ejecución.

El siguiente salto útil exige backend x64 SysV/macOS u objeto ARM64 Windows. No
se añade IPC lateral: para un parser llamado por cada candidato, el coste y la
superficie de fallo superarían el código sustituido.
