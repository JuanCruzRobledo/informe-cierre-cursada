# Reglas de cierre — mismas para Programación I, II y III

Confirmadas por el usuario el 04/07/2026 (misma normativa de cátedra para las
tres materias). No las reinventes ni las asumas distintas sin que el usuario lo
diga explícitamente.

## Árbol de decisión (por alumno, por materia)

```
autoeval_ok = TODAS las autoevaluaciones de todas las unidades dieron >= 90%
              (una sola unidad por debajo de 90% ya rompe la condición —
              NO es un promedio general)
tp_ok       = % de TPs con estado "Aprobado" >= UMBRAL_TP
              (UMBRAL_TP se pregunta en cada corrida, Fase 1 de SKILL.md —
              no hay un número fijo acordado, es una perilla por corrida)
P1_max      = nota más alta entre TODAS las instancias rendidas del Parcial 1
P2_max      = nota más alta entre TODAS las instancias rendidas del Parcial 2
TPI_max     = nota más alta entre TODAS las instancias rendidas del TPI

SI autoeval_ok Y tp_ok Y P1_max>=60 Y P2_max>=60 Y TPI_max>=60:
    → PROMOCIONA
    → Nota Final = round( [(P1_max+P2_max)/2]*0.4 + TPI_max*0.6 , /10, entero )

SINO SI autoeval_ok Y tp_ok Y P1_max>=40 Y P2_max>=40:
    → REGULARIZA (Cursado Aprobado)
    → Habilitado para Final = "Sí" si TPI_max>=60, si no "No (falta aprobar TPI)"

SINO:
    → RECURSA
    → Habilitado para Final = "No"
```

Fuente literal de la normativa (pegada por el usuario, no reformular):

> ✅ Aprobación directa (Promoción): 90% en todas las autoevaluaciones + ambos
> parciales ≥60% (1ra instancia o recuperación) + TPI ≥60% (1ra instancia o
> recuperación).
> 🟡 Cursado aprobado (habilita a rendir final): 90% en todas las
> autoevaluaciones + ambos parciales ≥40% (1ra instancia o recuperación).
> 📚 Examen Final: requiere "Cursado Aprobado" + TPI aprobado. Si no aprobó el
> TPI durante la cursada, se habilita entrega 10 días antes del final.
> NF = [(P1+P2)/2]*0.4 + TPI*0.6 (solo aplica para aprobación directa).
> ⚠️ Si no cumple ninguna condición → recursa.

La condición de TPs (`tp_ok`) **no está en la normativa pegada arriba** — la
agregó el usuario aparte, confirmando que la planilla modelo la usa (un TP
Desaprobado ahí correlaciona con RECURSA). A diferencia del resto de los
umbrales, este NO tiene un número fijo: **la skill tiene que preguntarlo en
cada corrida** (Fase 1 de SKILL.md), con un default sugerido de 100% (todos los
TPs aprobados) si el usuario no tiene un número mejor a mano.

## Supuestos de cálculo (confirmados, no reinventar)

- **N/E cuenta como no-aprobado** para el umbral de TPs — igual que
  "Desaprobado", no es una categoría neutra.
- **Las notas nunca se comparan crudas contra 60/90/40** — siempre como
  porcentaje del máximo configurado de ese ítem en Moodle. Un ítem puede estar
  configurado sobre 10 o sobre 100; normalizar antes de comparar (ver
  `moodle-export.md`, sección "Normalización de escala").
- **Nota Final en escala 0 a 10, sin decimales**, redondeo estándar
  (`round_half_up`, no bankers' rounding — ver `generar_informe_cierre.py`).
- Si un alumno no rindió ninguna instancia de un parcial/TPI, ese máximo es
  `None` → se muestra `"N/E"` en el reporte y automáticamente no cumple ningún
  umbral (no hace falta chequearlo aparte).

## Clasificación de ítems del calificador — 3 categorías por unidad, no 2

El calificador real (verificado en vivo el 04/07/2026 sobre Programación
I-Marzo 2026, course id=38) expone **tres tipos de ítem por unidad**, con
iconos distintos en el nombre:

1. **`Cuestionario de Actividad 1/2/3` 🧠** — miniquizzes de práctica dentro de
   la unidad. **Se ignoran por completo** (confirmado con el usuario) — no
   están en la normativa de cierre pegada arriba, ni cuentan para el 90% de
   autoevaluaciones ni para nada más. Si en una corrida real algo no cierra,
   esta es la primera hipótesis a revisar (ver "Duda abierta" al final).
2. **`Actividad de cierre unidad N` 🎯🏁** — esto ES el TP de esa unidad.
   Calificación por escala (`Aprobado`/`Desaprobado`), va a la lista `tps` del
   JSON de entrada del script.
3. **`Cuestionario de Autoevaluación - Unidad N` 🤓📖** — esto ES la
   autoevaluación de la regla del 90%. Calificación numérica, va a
   `autoeval_pcts`.

También existe un ítem `Antes de comenzar con el estudio de la materia...`
(cuestionario diagnóstico de entrada, sin icono de unidad) — **no es la
autoevaluación de ninguna unidad**, excluirlo siempre.

**Duda abierta (documentada, no resuelta):** el usuario no está 100% seguro de
que los miniquizzes 🧠 deban ignorarse — lo dejó así por default pero pidió que
quede anotado. Si al cerrar una materia real el número de "regularizados"
o "promocionados" no cuadra con lo que el tutor esperaba a ojo, preguntale
explícitamente si los miniquizzes deberían sumar al cálculo de autoevaluación
antes de asumir que el error está en otro lado.

## Particularidades por materia (heredadas de `informe-pendientes-curso`, PERO reinterpretadas)

Ojo: las exclusiones de la skill vieja (`materias-conocidas.md`) son para
**auditoría de correcciones pendientes**, no para cierre de cursada. No las
copies literal — la mayoría se invierten:

- **Programación I (course id=38, "P1_Marzo 2026")**: TP1 y TP2 se corrigen
  automático con Active-IA en el flujo de corrección normal, PERO para el
  cierre de cursada **SÍ cuentan** en el umbral de TPs — un alumno no puede
  regularizar si no los tiene aprobados, sin importar quién los corrigió.
- **Programación II (course id=42)**: sin exclusiones conocidas — todos los
  TPs y unidades cuentan. Reverificar el id en vivo (Fase 2), no asumir.
- **Programación III (course id=44)**: las unidades 9 y 10 son **"opcionales,
  no se evalúan"** (confirmado en sesión anterior) — excluirlas del
  denominador tanto de `tps` como de `autoeval_pcts`, no metas placeholders
  `null` por ellas. El TPI de Prog 3 no tiene instancia de recuperatorio
  habilitada en la cohorte de referencia — `tpi_pcts` va a tener como máximo
  una entrada, no lo trates como error si es así.

Todas estas particularidades cambian de cohorte a cohorte — reverificalas en
vivo en la Fase 2 de `SKILL.md`, igual que hace `informe-pendientes-curso` con
los `assign id`.
