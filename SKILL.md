---
name: informe-cierre-cursada
description: >-
  Genera el reporte de cierre de cursada (Excel con una hoja por materia: quien PROMOCIONA, quien REGULARIZA y quien RECURSA) para Programacion I, II y III en el campus Moodle TUP, exportando el calificador completo de cada materia y aplicando las reglas de aprobacion directa / cursado aprobado / recursada. Usar cuando el usuario pida cerrar el cursado, saber quien aprobo/regularizo/recursa, un informe de cierre de curso, el estado final de los alumnos, o una planilla de notas final por comision/tutor. NO dispara para auditar correcciones pendientes de calificar (esa es informe-pendientes-curso) ni para consultar la situacion de un alumno puntual sin necesidad de un reporte completo.
license: Apache-2.0
---

# Informe de Cierre de Cursada

Calcula, a nivel **curso completo** (todas las comisiones, todas de una vez),
quién PROMOCIONA, quién REGULARIZA y quién RECURSA cada materia — y arma un
Excel con una hoja por materia, agrupado por comisión/tutor, listo para
repartir o archivar. La regla que ordena todo: **el cálculo de Estado Alumno
nunca lo hace el agente "a ojo" leyendo notas** — siempre pasa por
`scripts/generar_informe_cierre.py`, que aplica el mismo árbol de decisión de
forma determinística todas las veces. Un cierre de cursada tiene consecuencia
académica real; la fórmula tiene que dar el mismo resultado hoy que en la
próxima corrida.

## Cuándo aplica

El usuario (tutor o coordinador con cuenta que ve todas las comisiones de la
materia) quiere cerrar la cursada de Programación I, II y/o III: saber, por
alumno, si promocionó, si regularizó (puede rendir final) o si recursa. Ya
tiene o puede conseguir acceso a `tup.sied.utn.edu.ar`. El resultado esperado
es un Excel, con una hoja por materia, agrupado por comisión y tutor, con el
resumen de conteo arriba (Promocionados/Regulares/Recursantes) y el detalle
alumno por alumno abajo.

Esta skill es prima de `informe-pendientes-curso` (comparten padrón y mapeo
comisión→tutor) pero resuelven preguntas distintas: esa audita **qué falta
corregir**, esta calcula **el veredicto final del cursado**. No las confundas.

## Las 6 fases

```
0. Bootstrap        → confirmar login + cuenta con visión global (SIN pedir credenciales)
1. Preguntar alcance → materia(s) + umbral de TPs + confirmar reglas (defaults precargados)
2. Verificar ítems   → inventario real de ítems del calificador (nombres varían por cohorte)
3. Exportar y armar  → exportar calificador (Todos los participantes) + clasificar columnas
4. Mapear comisión   → padrón + docente por comisión (reutilizado de informe-pendientes-curso)
5. Calcular y generar → JSON → scripts/generar_informe_cierre.py → Excel final
```

Es de **solo lectura** — no modifica ninguna nota ni configuración en Moodle,
así que no hace falta compuerta de aprobación para el barrido en sí. Sí hay
que pedir permiso explícito antes de **descargar** el archivo del exportador
(regla general de la herramienta de navegador: toda descarga se confirma con
el usuario primero, con nombre de archivo y origen).

## Fase 0 — Bootstrap (sin pedir credenciales)

Igual que `informe-pendientes-curso`: nunca pidas DNI ni contraseña. Pedile al
usuario que se loguee solo en `https://tup.sied.utn.edu.ar` (o confirmá sesión
activa navegando a `/my/` y leyendo el nombre logueado), y confirmá
explícitamente que la cuenta tiene **visión global** de la materia elegida. Si
no la tiene, avisá ANTES de arrancar — parar a mitad de un cierre de cursada es
carísimo en tiempo y en confusión sobre qué quedó a medio calcular.

## Fase 1 — Preguntar alcance (con defaults precargados)

Usá `AskUserQuestion` — son decisiones del usuario, no algo a inferir. Por
cada materia que el usuario nombre:

1. **¿Qué materia(s)?** Cada materia = un `course_id` distinto → una hoja
   separada en el Excel final.
2. **¿Qué % mínimo de TPs aprobados exige el umbral de "no recursa"?** Esta es
   la única perilla de la normativa que NO tiene un número fijo acordado — el
   usuario pidió explícitamente que se pregunte en cada corrida (ver
   `references/reglas-cierre.md`). Default sugerido: **100%** (todos los TPs
   aprobados), pero dejá que el usuario lo cambie.
3. **¿Las reglas de autoevaluación/parciales/TPI son las de siempre?** Default
   precargado: autoeval ≥90% en TODAS las unidades (no promedio), parciales
   ≥60% para promoción / ≥40% para regularizar, TPI ≥60% — la normativa
   completa está en `references/reglas-cierre.md`, no la repitas de memoria,
   leela de ahí. Si el usuario dice que para alguna materia cambió algún
   número, se sobreescribe solo para esa materia (campo `reglas` en el JSON de
   entrada del script).
4. **¿Dónde guardar el Excel final?** Default: misma carpeta usada en la
   última corrida de `informe-pendientes-curso`, si el usuario no tiene
   preferencia distinta.

No preguntes por el agrupamiento (comisión→tutor) ni por el formato de salida
(Excel con una hoja por materia): son el propósito fijo de esta skill.

## Fase 2 — Verificar ítems del calificador

**No asumas los nombres de ítems de una auditoría anterior.** Navegá a
`grade/export/xls/index.php?id=<course_id>` y mirá la lista real de "Ítems de
calificación a incluir" para la cohorte actual — los nombres varían por
materia y a veces entre cohortes de la misma materia. Clasificá cada ítem
usando los patrones de `references/moodle-export.md` (TP, Autoevaluación,
Parcial 1/2, TPI) y las particularidades ya conocidas de
`references/reglas-cierre.md` (unidades opcionales de Prog III, TP1/TP2 de
Prog I que SÍ cuentan acá aunque no cuenten en `informe-pendientes-curso`,
etc.). Los miniquizzes `Cuestionario de Actividad N` y el cuestionario
diagnóstico de entrada se descartan siempre — no forman parte de ninguna
regla de cierre.

## Fase 3 — Exportar y clasificar

Seguí `references/moodle-export.md` para el método completo. En resumen:
exportar con "Grupos separados = Todos los participantes" (un solo archivo por
materia, no uno por comisión), tildando únicamente los ítems clasificados como
TP/Autoevaluación/Parcial/TPI en la Fase 2. Pedile permiso al usuario antes de
descargar (nombre de archivo + qué contiene). Al procesar el archivo
descargado, normalizá cada nota a **porcentaje del máximo configurado de ese
ítem** — nunca compares números crudos entre ítems de distinta escala (ver
"Normalización de escala" en `moodle-export.md`).

## Fase 4 — Mapear a comisión/tutor

Reutilizá el método de `informe-pendientes-curso` (Fase 5 de esa skill,
snippets en su `references/moodle-scraping.md` si necesitás el detalle):
padrón completo una sola vez (`user/index.php?id=<course_id>&perpage=2000`,
columna "Grupos") para asignar cada alumno a su comisión, y
`user/index.php?id=<course_id>&group=<GID>&perpage=200` filtrando rol
"Profesor" para el tutor de cada comisión involucrada. Si hay más de un
docente con rol "Profesor" en la misma comisión, listalos separados por " / ".

## Fase 5 — Calcular y generar el Excel

Armá el JSON de entrada de `scripts/generar_informe_cierre.py` (formato
documentado en el docstring del script, ejemplo completo en
`assets/ejemplo_input.json`) con las columnas ya clasificadas y el padrón ya
cruzado, y corré el script:

```bash
python scripts/generar_informe_cierre.py --input datos.json --output Cierre_Cursada.xlsx
```

El script calcula el Estado Alumno con el árbol de decisión de
`references/reglas-cierre.md` (autoeval, umbral de TPs, nota más alta entre
instancias de parciales/TPI, Nota Final redondeada 0-10) y escribe el Excel
final — una hoja por materia, resumen de conteo arriba, agrupado por
comisión/tutor abajo. Verificá el resultado abriendo el Excel antes de darlo
por bueno: el resumen (Promocionados/Regulares/Recursantes) tiene que sumar
igual a la cantidad total de alumnos procesados en esa materia.

## Reglas duras

- Nunca pidas DNI ni contraseña — el login lo hace el usuario en su navegador.
- Nunca calcules el Estado Alumno "a mano" en la respuesta — siempre por
  `scripts/generar_informe_cierre.py`. Es la única forma de que el mismo
  alumno dé el mismo resultado en dos corridas distintas.
- Nunca compares una nota cruda contra 60/90/40 sin normalizarla primero a
  porcentaje del máximo de ese ítem — items en distinta escala (0-10 vs 0-100)
  conviven en el mismo curso.
- Nunca asumas los nombres de ítems o `course_id` de una auditoría anterior sin
  reverificarlos en la Fase 2 — cambian de cohorte a cohorte.
- Nunca inventes el umbral de TPs — preguntalo siempre en la Fase 1, no hay un
  número fijo acordado para esa perilla.
- No modifiques nada en Moodle: esta skill es de solo lectura. Pedí permiso
  explícito antes de cualquier descarga.

## Componentes de la skill

| Archivo | Para qué |
|---|---|
| `scripts/generar_informe_cierre.py` | Calcula Estado Alumno (árbol de decisión) y genera el Excel final (una hoja por materia) a partir del JSON clasificado |
| `assets/ejemplo_input.json` | Ejemplo completo de JSON de entrada, validado contra los 3 casos reales de la planilla modelo (Promociona/Regulariza/Recursa) |
| `references/reglas-cierre.md` | Normativa completa de cierre, clasificación de ítems (3 categorías por unidad), particularidades por materia |
| `references/moodle-export.md` | Método técnico: exportador del calificador, regex de clasificación, normalización de escala, mapeo comisión→tutor |
