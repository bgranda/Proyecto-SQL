# Proyecto BBDD · Del CSV al modelo relacional

Base de datos relacional en **PostgreSQL** (alojada en **Render**) creada a partir de los datos sin normalizar de alumnos y profesores de un bootcamp. Proyecto de la vertical de Data Science de **The Bridge**.

**Equipo:** [nombres del equipo]

---

## Qué hay en este repositorio

```
bootcamp-db-project/
├── data/                         Datos de entrada (CSV originales)
│   ├── clase_1.csv … clase_4.csv   Alumnos: 2 clases de Data Science y 2 de Full Stack
│   └── claustro.csv                Profesores
├── Diagramas/
│   ├── diagrama_ER.png           Modelo entidad-relación (entregable 1)
│   ├── diagrama_logico.png       Modelo lógico (entregable 2)
│   └── modelo_logico.dbml        Código para regenerar el modelo lógico en dbdiagram.io
├── ModeloE/                      Diagrama E/R editable (draw.io)
├── sql/
│   └── crear_tablas.sql          Creación de las 10 tablas con PK, FK, UNIQUE y CHECK
└── src/                          Notebooks de carga, verificación y consultas
```

---

## El modelo

10 tablas de tres tipos:

| Tipo | Tablas | Para qué sirven |
| --- | --- | --- |
| Principales | `alumnos`, `profesores`, `promociones`, `proyectos` | Las entidades del bootcamp |
| Catálogos | `campus`, `verticales`, `modalidad` | Listas que pueden crecer añadiendo filas, sin cambiar la estructura |
| Cruce (N:M) | `matriculas`, `profesor_promocion`, `alumno_proyecto` | Relaciones de muchos a muchos |

![Modelo entidad-relación](Diagramas/diagrama_ER_1.png)

![Modelo lógico](Diagramas/diagrama_logico.png)

### Claves

- **PK:** un id propio (`serial`) en todas las tablas.
- **UNIQUE:** el mínimo de datos reales que no se puede repetir.

| Tabla | UNIQUE |
| --- | --- |
| `campus`, `verticales`, `modalidad` | el nombre |
| `promociones` | `fecha_comienzo` + `id_campus` + `id_vertical` |
| `proyectos` | `nombre_proyecto` + `id_vertical` |
| `alumnos` | `email` |
| `profesores` | `nombre_profesor` (suponemos que no hay dos profesores con el mismo nombre) |
| `matriculas` | `id_alumno` + `id_promocion` |
| `profesor_promocion` | `id_profesor` + `id_promocion` |
| `alumno_proyecto` | `id_matricula` + `id_proyecto` |

Además, dos `CHECK`: `rol` solo admite `TA` o `LI`, y `resultado` solo `Apto` o `No Apto`.

---

## Decisiones de diseño

- **Una promoción es fecha + campus + vertical.** "Septiembre" no basta: no distingue años ni verticales.
- **Proyectos como filas, no como columnas.** Un proyecto nuevo es una fila nueva, no un cambio en la estructura.
- **Matrícula entre alumno y promoción.** Un alumno puede repetir o cursar otra vertical. Los resultados de los proyectos cuelgan de la matrícula, para saber de qué promoción es cada uno.
- **Rol y modalidad en cada asignación del profesor.** Un profesor puede ser TA en una edición y LI en otra, o dar clase presencial un año y online otro.
- **"No hay clase sin LI" es una regla de negocio, no una restricción.** Los datos no la cumplen (3 de las 6 promociones no tienen LI), así que la cardinalidad mínima es 0 y la regla queda documentada.
- **Nombres completos de las verticales.** Los CSV dicen DS y FS; guardamos Data Science y Full Stack.
- **La fecha de los profesores sale de los datos.** El claustro solo indica el mes; la fecha se toma de los alumnos de ese mismo mes. Si hubiera dos promociones del mismo mes, la carga se detiene y pide la fecha en lugar de inventarla.
- **El mes no se guarda:** se calcula a partir de la fecha de comienzo.

Principio que seguimos en todo el diseño: **un dato que no está en los CSV no se inventa.** Un hueco vacío dice la verdad; un dato inventado, no.

---

## Cómo se cargan los datos

```
CSV ──pandas──► tablas de paso (stg_) ──INSERT … SELECT con JOIN──► tablas del modelo
```

1. **Crear las tablas:** `sql/crear_tablas.sql`.
2. **Preparar los datos con pandas:** añadir la vertical (solo se sabe por el archivo), convertir las fechas, traducir DS/FS y pasar los proyectos de columnas a filas (`melt`).
3. **Subirlos a Render como tablas de paso:** `stg_alumnos`, `stg_claustro` y `stg_resultados`.
4. **Llenar cada tabla con `INSERT … SELECT`**, usando `JOIN` para traducir los nombres ("Madrid", "Data Science") a sus ids.

Cada `INSERT` termina con un `EXCEPT` contra la propia tabla: solo inserta lo que falta, así que la carga se puede repetir sin duplicar datos.

---

## Cómo reproducirlo

**1. Instalar las librerías**

```
pip install pandas sqlalchemy "psycopg[binary]" psycopg2-binary python-dotenv
```

**2. Crear un archivo `.env`** en la raíz del proyecto con la URL de la base de datos:

```
DATABASE_URL=postgresql://usuario:contraseña@host/base_de_datos
```

El `.env` está en `.gitignore`: la contraseña no se sube nunca al repositorio.

**3. Ejecutar los notebooks de `src/` en este orden:**

| Orden | Notebook | Qué hace |
| --- | --- | --- |
| 1 | `crear_tablas.ipynb` | Borra y crea las 10 tablas vacías (¡borra los datos!) |
| 2 | `catalogos.ipynb` | Carga `campus`, `verticales` y `modalidad` |
| 3 | `alumnnos_profesores.ipynb` | Carga `alumnos` y `profesores` |
| 4 | `chuleta.ipynb` | Prepara los datos y sube las tablas de paso |
| 5 | `promociones.ipynb` | Carga `promociones`, `proyectos`, `matriculas`, `profesor_promocion` y `alumno_proyecto` con SQL |
| 6 | `verificacion.ipynb` | Comprueba la carga (solo lectura) |
| 7 | `consultas.ipynb` | Consultas de demostración (solo lectura) |

---

## Verificación

Resultado esperado de la carga:

| Tabla | Filas |
| --- | --- |
| campus · verticales · modalidad | 2 · 2 · 2 |
| promociones | 6 |
| proyectos | 10 |
| alumnos | 52 |
| profesores | 10 |
| matriculas | 52 |
| profesor_promocion | 10 |
| alumno_proyecto | 260 |

`verificacion.ipynb` compara lo que genera cada consulta de carga con lo que hay en la base de datos, con `EXCEPT` en los dos sentidos: en las cinco tablas cargadas con SQL no falta ni sobra ninguna fila. También comprueba que cada promoción tiene los mismos alumnos que filas tiene su CSV (15, 10, 14 y 13) y compara los resultados de un alumno con su fila original.

---

## Consultas de ejemplo

| Pregunta | Respuesta |
| --- | --- |
| ¿Qué proyecto cuesta más? | EDA en Data Science (56 % de aptos) y WebDev en Full Stack (59,3 %) |
| ¿Qué promoción fue mejor? | Sep-2023 · Madrid · Data Science, con un 81,3 % de aptos |
| ¿Quién aprobó todo? | 11 alumnos: 6 de Data Science y 5 de Full Stack |

Las consultas completas están en `src/consultas.ipynb`.

---

## Mejoras futuras

- **Una persona, varios papeles:** hoy, un alumno que pasa a ser TA quedaría duplicado. Se resolvería con una tabla de personas y su participación en cada promoción.
- **Modalidad de la promoción:** guardarla cuando haya datos que la indiquen.
- **Fechas de inicio y fin en `profesor_promocion`:** para saber qué TA cubrió cada periodo.

---

## Tecnologías

PostgreSQL · Render · Python · pandas · SQLAlchemy · Jupyter · dbdiagram.io · draw.io
