# Depurador de Excels — Contabilidad UADE

Sistema HTML de un solo archivo (sin backend, sin login) para depurar las exportaciones
que usa el sector de contabilidad. Cada tipo de reporte es un **módulo** (una pestaña):
sube su propio archivo, corre su propia lógica de detección/corrección, y su resultado
queda disponible aunque se pase a otra pestaña — subir un archivo en un módulo nunca
borra lo que ya se depuró en otro.

## Módulos

### Notas de Salida

El archivo (`Notas_de_Salida_XXXXXXX.xls`) **no es un binario Excel real**: es texto
plano separado por tabs, `windows-1252`, filas por `\r\n`, con 5 líneas de metadata antes
del encabezado (línea 6). Dos defectos de origen, ambos causados por caracteres
incrustados en campos de texto libre por carga de datos:

- **Salto de línea (`\n`) incrustado en `DESTINATARIO`** (~2% de las filas): si algo
  separa filas por *cualquier* salto de línea en vez de por el verdadero fin de fila
  `\r\n`, corta la fila en dos. Corregido de raíz parseando por `\r\n` únicamente — el
  `\n` nunca llega a partir nada, solo se reemplaza por un espacio dentro del campo.
- **Tab incrustado justo antes de `CANTIDAD`**: produce un campo vacío fantasma que
  corre todo lo posterior una columna a la derecha. Se detecta por longitud de fila y se
  corrige eliminando el campo fantasma. Verificado con `CANTIDAD × PRECUNIT =
  TOTALSINDESCUENTOS` en las filas afectadas.

Avisos de contenido (no son bugs, son datos reales a revisar): `ESTADO` distinto de
`CERRADO`, y filas 100% duplicadas.

### Requerimientos de Gastos

Mismo formato de archivo (texto por tabs, 6 líneas de metadata, encabezado en la línea 7,
34 columnas reales). El defecto acá es más variado: varios campos de texto libre
(`NOTA`/`DETALLE`, `DATOSPROVEEDOR`, `DETALLE_ITEM`/`OBSERVACIONES`) pueden traer tabs
incrustados (contenido pegado desde otro lado), fragmentando la fila en más columnas de
las que corresponden — y más de uno puede fragmentarse a la vez en la misma fila.

Se reconstruye ubicando **anclas confiables** que nunca vienen de texto libre: `SINPARTIDA`
(siempre `SI`/`NO`), el bloque de 3 `SI`/`NO` seguidos (`ESCHEQUERAPIDO`, `COMPRAWEB`,
`COMPRA_ALTO_MONTO`), y `CODIGOPROD` (patrón fijo de 21 caracteres, 3 letras + 18 dígitos).
Todo lo que sobra entre anclas se reabsorbe en el campo de texto libre correspondiente,
uniendo con espacios. Si las anclas no se pueden ubicar de forma consistente, la fila
queda marcada como anomalía sin modificarla — nunca se fuerza una reconstrucción dudosa.
Verificado contra las 13 filas reales con este defecto en el archivo de muestra.

### Requerimiento de Compra

Mismo formato (texto por tabs, 6 líneas de metadata, encabezado en la línea 7), pero
mucho más ancho (46 columnas reales) y con varios campos de texto libre que pueden traer
tabs incrustados en la misma fila (`DETALLE` cerca del inicio; `DESCPROD`/`DETALLEITEM`/
`OBS` más adelante, tratados como un mismo bloque flexible; `MOTIVO`).

**Ninguna fila se acepta solo por tener la cantidad "correcta" de columnas.** Se encontró
un caso real donde dos defectos se compensaban (un campo se fragmentaba de más mientras
otro perdía su valor por completo) y el total de columnas terminaba coincidiendo con lo
esperado — una fila así pasaba el chequeo de longitud pero quedaba con el contenido
completamente desalineado, sin marcarse como corregida ni como anomalía. Por eso **toda**
fila, tenga o no la longitud esperada, se valida con la misma cadena de anclas: `SINPARTIDA`
(`SI`/`NO`), `CODPROD` ubicado por posición fija justo después del tramo `ORIGIN..LINEA`
(nunca por búsqueda de patrón — `CODACT_CABE` a veces tiene la misma forma, `EXT2616`, y
podía enganchar por error), y el terceto `IMPORTE_TOTAL_LINEA`/`CANTIDAD`/`IMPORTE_UNITARIO`
ubicado por su propia relación aritmética (`CANTIDAD × IMPORTE_UNITARIO = IMPORTE_TOTAL_LINEA`)
en vez de por coincidencia de valores contra otra columna — se comprobó con un archivo real
más grande que `CC_CABECERA`/`CODACT_CABE` pueden diferir legítimamente de sus pares a nivel
ítem, así que ya no se usan como ancla de validación, solo la aritmética. Lo que no cierra
se marca como anomalía sin modificar.

**Limitación conocida**: cuando `DESCPROD` se fragmenta (le falta el final del nombre del
producto), esa continuación termina mezclada en `DETALLEITEM`/`OBS` en vez de extender
`DESCPROD` — un problema cosmético en un campo descriptivo, nunca en los importes o
cantidades (que se verifican aparte). Verificado contra las 3 filas reales del archivo de
muestra original más 5 filas de un archivo real más grande con el defecto compensado
descripto arriba — las 8 recuperan correctamente los importes, cantidades y fechas.

### Comprometidos

Mismo formato de archivo (texto por tabs, `windows-1252`), pero con metadata más corta:
solo 3 líneas antes del encabezado (título + 2 líneas en blanco), encabezado en la línea 4.
10 columnas, incluido `ROUND(TOTAL,2)` — una expresión SQL filtrada tal cual al export, se
usa así a propósito, no es un typo a "corregir".

A diferencia de los otros módulos, acá `DETALLE` y `NOTA` son ambos texto libre frecuente
y sin ninguna ancla fija entre medio (ni un enum `SI`/`NO`, ni un código con patrón
reconocible) — no hay forma confiable de saber dónde cortar un tab incrustado. Por eso este
módulo **no intenta reconstruir** filas fragmentadas: si la cantidad de columnas no cierra
(o el importe no es numérico con la cantidad de columnas ya correcta), la fila se marca
directamente para revisión manual, sin forzar nada. Verificado contra un archivo real de
2.509 filas: 1 sola fila con este problema.

Aviso de contenido: filas 100% duplicadas (pasa legítimamente con pagos en cuotas del mismo
comprobante).

### Resumen General (módulo 5 — el "rejunte")

No tiene subida de archivo propia: lee lo que ya esté cargado en los otros módulos
(`ROLLUP_SOURCE_IDS` en el código — hoy `ns`, `rg`, `rc`, `co`) y arma un dashboard en
vivo, sin volver a pedir nada.

- **Fuentes**: qué módulos tienen datos cargados y cuántas filas.
- **Dashboard**: importe total, filas totales, módulos cargados, y tres rankings visuales
  (barras, no tablas de texto) — por módulo, por centro de costo, y por persona (quién
  concentra más gasto) — top 8 cada uno, ordenados de mayor a menor.
- Cada módulo aporta su propio campo de "importe" y "centro de costo"/"persona" via
  `dashboardMap` en su definición (`{amountCol, centroCol, personaCol}`) — así el rollup
  no necesita saber los nombres de columna de cada formato.
- **Las filas marcadas como anomalía se excluyen del cálculo** (no se puede confiar en su
  importe/centro si no se pudieron reconstruir) — se cuentan aparte en un stat card
  cuando corresponde, nunca se suman en silencio.
- **Exportación multi-hoja**: un único `.xlsx` con una hoja `Resumen` (las mismas tablas
  del dashboard, en datos) + una hoja por cada módulo con datos cargados, con el mismo
  contenido depurado que bajarías desde ese módulo individualmente.

## Diseño común a todos los módulos

- Barra de salud compacta (limpias / corregidas / avisos / a revisar) en vez de texto.
- Los N° de fila afectados van en listas colapsables (`<details>`), no como bloques de
  texto siempre visibles.
- Exportación a `.xlsx` con encabezado + filtro automático: columnas numéricas reales
  (coma argentina → punto), fechas reales (`dd/mm/yyyy`, serial calculado con aritmética
  UTC pura para no depender de la zona horaria del navegador), y códigos/IDs con ceros a
  la izquierda preservados como texto.
- Agregar un módulo nuevo = un `parseX(text)` que devuelva `{headers, rows, fixes,
  avisos}` (cada fila con `_id` y `_flags`) + una entrada en `MODULES` — el resto (tabs,
  stats, tabla, exportación, persistencia entre pestañas) es genérico y no hay que
  tocarlo.

## Desarrollo local

```bash
python -m http.server 8937 --directory "Notas de Salida"
```

(o usar la entrada `notas-salida` de `.claude/launch.json` desde la raíz del repo de
trabajo local, si corresponde).

## Deploy

Mismo patrón que los demás sistemas UADE: repo en GitHub, importado en Vercel con preset
"Other", root `./`, sin build command ni output directory.
