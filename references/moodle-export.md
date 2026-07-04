# Método técnico — exportar el calificador, no scrapear HTML

A diferencia de `informe-pendientes-curso` (que lee `mod/assign` página por
página porque necesita el estado exacto de entrega), acá necesitamos notas
finales consolidadas por alumno — y Moodle ya tiene una función nativa para
eso: el **Exportador del calificador**. Usarla en vez de scrapear la tabla
HTML del Informe del calificador ahorra el 90% del trabajo y es mucho más
confiable (sin paginación, sin rowspan/colspan raros).

## Login (Fase 0 — igual que informe-pendientes-curso)

No pidas DNI ni contraseña. El usuario se loguea solo; confirmá sesión activa
navegando a `/my/` y leyendo el nombre logueado, y confirmá que la cuenta tiene
**visión global** de la materia (ve todas las comisiones).

## Exportar a Hoja de cálculo Excel (Fase 3)

URL: `grade/export/xls/index.php?id=<course_id>`

En esa pantalla:

1. **"Grupos separados"** → dejar en **"Todos los participantes"**. Exportar
   por comisión (27 descargas para Prog I, por ejemplo) es innecesariamente
   caro — un solo export trae a todos los alumnos de la materia, y el mapeo a
   comisión/tutor se hace después cruzando con el padrón (Fase 5, igual que la
   skill vieja).
2. **"Ítems de calificación a incluir"** — tildar:
   - Todas las `Actividad de cierre unidad N` (los TPs)
   - Todas las `Cuestionario de Autoevaluación - Unidad N` (o variantes de
     nombre, ver abajo)
   - **NO tildar** los `Cuestionario de Actividad 1/2/3` (miniquizzes, se
     ignoran — ver `reglas-cierre.md`)
   - **NO tildar** el cuestionario diagnóstico `Antes de comenzar...`
   - Todas las instancias de Parcial 1 (entrega + recuperatorio +
     extraordinaria + cualquier variante con fecha en el nombre)
   - Todas las instancias de Parcial 2 (mismas variantes)
   - Todas las instancias del TPI (`Entrega del Trabajo Integrador`,
     `Recuperatorio Entrega del Trabajo Integrador`, etc.)
3. Click **Descargar** → pedile permiso explícito al usuario antes de
   descargar (regla general de la herramienta de browsing), decile nombre de
   archivo y que va a un solo export por materia.

**Antes de tildar nada**, mirá los nombres reales de los ítems en esa pantalla
para esa cohorte — no asumas que se llaman exactamente igual que en la
auditoría de referencia (04/07/2026, Prog I id=38). Los nombres varían por
materia y a veces por cohorte (ver `reglas-cierre.md`, sección
"Particularidades por materia"). Clasificá por **regex sobre el texto**, nunca
por posición de columna — el orden de columnas en el export NO sigue el orden
numérico de unidades (ej.: en la auditoría de referencia, la Unidad 8 apareció
al final de todo, después de todas las instancias de parciales).

Patrones de regex que funcionaron en la auditoría de referencia:

```
TP (cierre de unidad):     /Actividad de cierre.*unidad\s*(\d+)/i
Autoevaluación:            /(Cuestionario de )?Autoevaluaci[oó]n.*Unidad\s*(\d+)/i
Parcial 1 (instancia):     /Primer(a)?\s+(Examen\s+)?Parcial/i  y variantes con
                            "Recuperatorio", "Extraordinaria", "Extensión"
Parcial 2 (instancia):     /Segundo\s+(Examen\s+)?Parcial/i     (mismas variantes)
TPI (instancia):           /Entrega.*Trabajo Integrador/i       (con o sin
                            "Recuperatorio" antepuesto)
Miniquiz (ignorar):        /Cuestionario de Actividad \d/i
Diagnóstico (ignorar):     /Antes de comenzar/i
```

Ignorá también cualquier instancia con fecha claramente vieja (otra cohorte) —
mismo criterio que `informe-pendientes-curso` para excluir legacy.

## Normalización de escala

Los ítems de Moodle no comparten escala: en la auditoría de referencia las
autoevaluaciones estaban sobre 10 (ej. `9,50`) mientras que los parciales
estaban sobre 100 (ej. `80,00`). **Siempre convertí a porcentaje del máximo
configurado de ESE ítem** antes de compararlo contra 60/90/40 — nunca
compares el número crudo. El máximo de cada ítem se puede leer desde la misma
pantalla de exportación (columna de puntaje máximo) o desde
`grade/edit/tree/index.php?id=<course_id>` si hace falta confirmarlo.

## Padrón y mapeo comisión→tutor (Fase 5 — reutilizado tal cual de informe-pendientes-curso)

```
user/index.php?id=<course_id>&perpage=2000
```

Trae nombre + email + columna "Grupos" (comisión). Cruzá por nombre/email
contra las filas del export para asignar cada alumno a su comisión. Docente
por comisión:

```
user/index.php?id=<course_id>&group=<GROUP_ID>&perpage=200
```

Filtrar rol "Profesor" (no "sin permiso de edición"); si hay más de uno,
listarlos separados por " / " — mismo criterio que la skill vieja.

## De la clasificación al script

Una vez clasificadas las columnas y cruzado el padrón, armá el JSON de entrada
de `scripts/generar_informe_cierre.py` (formato documentado en el docstring del
script, con un ejemplo completo en `assets/ejemplo_input.json`) y corré el
script — calcula el Estado Alumno y escribe el Excel final, una hoja por
materia, sin que tengas que reescribir la fórmula de cierre cada vez.
