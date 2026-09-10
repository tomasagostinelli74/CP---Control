# Depurador de Notas de Salida — Contabilidad UADE

Herramienta HTML de un solo archivo (sin backend, sin login) para depurar la exportación
de "Notas de Salida" antes de que el sector de contabilidad la use.

## El problema que resuelve

El archivo que exporta el sistema (`Notas_de_Salida_XXXXXXX.xls`) **no es un binario Excel
real**: es texto plano separado por tabs (`\t`), codificado en `windows-1252`, con filas
terminadas en `\r\n`. Tiene 5 líneas de metadata (título + rango de fechas) antes del
encabezado real, que está en la línea 6.

Algunas filas (en el archivo de muestra, 51 de 2441 = ~2%) tienen un **salto de línea
(`\n`) incrustado dentro del campo `DESTINATARIO`** — un defecto de carga de datos en el
sistema de origen (alguien escribió un enter de más en un nombre). Cuando esa fila se abre
o procesa con una herramienta que separa filas por *cualquier* salto de línea en vez de por
el verdadero fin de fila `\r\n`, el `\n` incrustado corta la fila en dos: la primera mitad
se queda con las columnas A–D (de `CODIGOPRODUCTO` en adelante, vacío), y el resto de los
datos (en verdad las columnas E–N) terminan como una fila "fantasma" con la columna A vacía
y los datos corridos a B–K. Es exactamente el patrón de filas rotas que reportó contabilidad.

**La corrección de raíz**: en vez de detectar y "reconstruir" filas ya rotas (heurística
propensa a error), esta herramienta parsea el archivo original correctamente — separando
filas *solo* por `\r\n` y campos por `\t` — con lo cual el `\n` incrustado nunca llega a
partir una fila. El salto de línea se reemplaza por un espacio dentro del campo, se recorta,
y la fila queda íntegra desde el primer parseo. No hay ambigüedad ni adivinación.

## Qué hace

1. **Sube el archivo** (`.xls`, tal cual lo entrega el sistema).
2. Elimina las primeras 5 filas de metadata; la fila 6 pasa a ser el encabezado real.
3. Detecta y corrige los campos con salto de línea incrustado (marcados como "corregida"
   en la vista previa, con un resumen de qué N° de Nota de Salida fueron afectados).
4. Muestra la tabla depurada en el navegador (con buscador y filtro "solo corregidas").
5. Permite **exportar** el resultado como `.xlsx` (encabezado + filtro automático), con:
   - `NOTASALIDA`, `CODIGO_CUENTA`, `CODIGOPRODUCTO` como texto (se preservan ceros a la
     izquierda).
   - `FECHA_TRANSACCION` como fecha real de Excel (`dd/mm/yyyy`).
   - `CANTIDAD`, `PRECUNIT`, `TOTALSINDESCUENTOS` como números reales (coma decimal
     argentina convertida a punto), para poder sumarlos/analizarlos directamente en Excel.

## Alcance actual / próximos pasos

Esta es la primera versión: resuelve específicamente el defecto de salto de línea
incrustado en `DESTINATARIO`. El usuario mencionó un segundo patrón de corrimiento de
columnas que se abordará en una sesión futura — queda pendiente, no implementado todavía.

Si en algún momento el archivo de entrada cambia (otro layout, otra codificación), avisar
para ajustar el parser — está escrito para este layout específico, no es un parser
genérico de Excel.

## Desarrollo local

```bash
python -m http.server 8937 --directory "Notas de Salida"
```

(o usar la entrada `notas-salida` de `.claude/launch.json` desde la raíz del repo de
trabajo local, si corresponde).

## Deploy

Mismo patrón que los demás sistemas UADE: repo en GitHub, importado en Vercel con preset
"Other", root `./`, sin build command ni output directory.
