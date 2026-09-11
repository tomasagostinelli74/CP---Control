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
mucho más ancho (46 columnas reales) y con **hasta cuatro** campos de texto libre que
pueden traer tabs incrustados en la misma fila (`DETALLE` cerca del inicio; `DETALLEITEM`
y `OBS` más adelante, tratados como un mismo bloque; `MOTIVO`; `OBSERFASTTRACK`). Mismo
enfoque de anclas que en Requerimientos de Gastos, pero encadenando más de ellas: además
de `SINPARTIDA` (`SI`/`NO`) y el patrón fijo de `CODPROD`, se usan dos **anclas por
valor** — `CC_CABECERA`/`DESC_CC` tienen que reaparecer literalmente, en ese orden, como
`COD_CCITEM`/`NOM_CCITEM` más adelante en la misma fila, y lo mismo para
`CODACT_CABE`/`DESCACT_CABE` contra `CODIGO_CODACTITEM`/`NOM_CODACTITEM` justo después —
si esos pares no coinciden exactamente, la fila se descarta como anomalía en vez de
reconstruirse a ciegas. Verificado contra las 3 filas reales con este defecto (0
anomalías) y con `CANTIDAD × IMPORTE_UNITARIO = IMPORTE_TOTAL_LINEA` exacto en las 150
filas del archivo de muestra.

### Resumen General (módulo 5 — el "rejunte")

No tiene subida de archivo propia: lee lo que ya esté cargado en los otros módulos
(`ROLLUP_SOURCE_IDS` en el código — hoy `ns`, `rg`, `rc`; `comprometidos` se suma ahí el
día que exista) y arma un dashboard en vivo, sin volver a pedir nada.

- **Fuentes**: qué módulos tienen datos cargados y cuántas filas, con los que todavía
  faltan marcados aparte (hoy: Comprometidos).
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
