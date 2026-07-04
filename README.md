# informe-cierre-cursada

Calcula el **cierre de cursada** (Promociona / Regulariza / Recursa) por
alumno en Programación I, II y III, y genera un **Excel con una hoja por
materia**, agrupado por comisión y tutor.

> El cálculo de Estado Alumno nunca lo hace el agente "a ojo" — siempre pasa
> por un script determinístico, para que un cierre de cursada dé el mismo
> resultado hoy que en la próxima corrida.

---

## ¿Qué hace?

Productiza el cierre de cursada de Programación I/II/III: en vez de leer notas
a mano en el calificador de Moodle y decidir caso por caso quién promociona,
regulariza o recursa, la skill pregunta lo que hace falta decidir (umbral de
TPs, materias, carpeta destino) y automatiza el resto.

1. **Bootstrap** — pide que el usuario se loguee solo en el campus (nunca pide
   DNI ni contraseña) y confirma que la cuenta ve todas las comisiones.
2. **Alcance** — pregunta materia(s) y el umbral de % de TPs aprobados (la
   única perilla de la normativa sin número fijo), con las reglas de
   autoevaluación/parciales/TPI ya precargadas.
3. **Exportar** — usa el exportador nativo del calificador de Moodle (no
   scraping de HTML) para traer todas las notas de la materia en un solo
   archivo, y clasifica cada columna por su nombre real.
4. **Mapear y calcular** — cruza cada alumno con su comisión y tutor, y corre
   `scripts/generar_informe_cierre.py` — aplica el árbol de decisión de la
   normativa y arma el Excel final, mismo layout siempre: resumen de conteo,
   detalle por comisión/tutor.

---

## Instalación

```bash
npx skills add https://github.com/juancruzrobledo/informe-cierre-cursada
```

La skill queda disponible para tu agente y se carga sola cuando pedís cerrar
el cursado, un informe de cierre de curso, o saber quién aprobó/regularizó/
recursa una materia.

### Dependencias

```bash
python -m pip install -r requirements.txt
```

---

## Uso

Le decís al agente algo como:

```
"Quiero cerrar el cursado de Programación I, II y III"
"Decime quién regularizó, recursa y aprobó en Prog III"
"Armame la planilla de cierre de cursada de este cuatrimestre"
```

El agente te pregunta la materia (si no la dijiste), el umbral de TPs
aprobados y confirma las reglas de cierre con el default ya cargado, te pide
loguearte en el campus, y te devuelve el Excel final con una hoja por materia
— sin que tengas que explicar de nuevo la normativa ni el formato cada vez.

---

## Estructura

```
informe-cierre-cursada/
├── SKILL.md                          # Flujo que sigue el agente (6 fases)
├── README.md
├── requirements.txt                  # openpyxl
├── scripts/
│   └── generar_informe_cierre.py     # Árbol de decisión + Excel final (una hoja por materia)
├── assets/
│   └── ejemplo_input.json            # Ejemplo de entrada, validado contra los 3 casos de la planilla modelo
└── references/
    ├── reglas-cierre.md              # Normativa completa + clasificación de ítems del calificador
    └── moodle-export.md              # Método técnico: exportador, regex de clasificación, mapeo comisión→tutor
```

---

## Por qué esta estructura

- **El árbol de decisión va en un script, no en prosa repetida**: un cierre de
  cursada tiene consecuencia académica real — dejarlo como instrucciones que
  el agente "interpreta" cada vez arriesga que el mismo alumno dé un resultado
  distinto en dos corridas. El script se corrió contra los 3 casos reales de
  la planilla modelo (Promociona/Regulariza/Recursa) antes de darlo por
  bueno.
- **Exportador nativo del calificador, no scraping de HTML**: a diferencia de
  `informe-pendientes-curso` (que necesita el estado exacto de entrega,
  página por assign), acá alcanza con notas consolidadas — el exportador de
  Moodle ya las da en un solo archivo, sin paginación ni tablas con
  rowspan/colspan.
- **Las reglas van en `references/`, separadas del árbol de decisión**: la
  normativa (porcentajes, qué cuenta como TP vs autoevaluación vs miniquiz) es
  el tipo de cosa que puede cambiar de cátedra a cátedra — tenerla documentada
  aparte del código hace explícito qué se puede tocar sin tocar el script.
- **Comparte padrón y mapeo comisión→tutor con `informe-pendientes-curso`**:
  son la misma cuenta, el mismo curso, el mismo tutor por comisión — no hacía
  falta reinventar ese pedazo, solo referenciarlo.

---

## Licencia

Apache-2.0
