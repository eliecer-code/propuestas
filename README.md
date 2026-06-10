# Vincular Presupuesto a un Único Modelo de Revit

Para evitar la corrupción de datos y garantizar que los elementos de un modelo de Revit no se mezclen con el presupuesto de otro, es necesario establecer un vínculo unidireccional y exclusivo (1:1) entre el archivo del modelo y el presupuesto almacenado en la base de datos.

A continuación se detalla la propuesta técnica para lograrlo.

---

## 1. Identificación Única del Modelo de Revit

En la API de Revit existen tres formas principales de identificar un archivo de modelo de forma única y persistente.

### Opción A: `ProjectInformation.UniqueId` (Recomendada para flujo general)

Cada archivo de Revit contiene un único elemento que almacena la información del proyecto. Su identificador único (`UniqueId`) es un GUID persistente.

**Código C#:**

```csharp
doc.ProjectInformation.UniqueId
```

**Comportamiento:**

* Se mantiene idéntico si el archivo se renombra o se mueve de carpeta.
* Cambia únicamente si se crea un archivo nuevo desde cero o si se fuerza su recreación.

**Limitación:**

Si un usuario duplica físicamente el archivo `.rvt` en Windows (Copy-Paste) para iniciar otro proyecto derivado, ambos archivos compartirán inicialmente el mismo identificador.

---

### Opción B: `GetWorksharingCentralGUID()` (Recomendada para trabajo colaborativo)

Si el proyecto utiliza trabajo compartido (*Worksharing*) o modelos centrales alojados en servidor local o Autodesk Construction Cloud.

**Código C#:**

```csharp
doc.GetWorksharingCentralGUID()
```

**Comportamiento:**

* Devuelve un identificador global único asignado al modelo central.
* Es la opción más segura para entornos corporativos y colaborativos.

**Limitación:**

Retorna `Guid.Empty` cuando el archivo corresponde a un modelo local de un único usuario sin trabajo colaborativo.

---

### Opción C: Parámetro del Proyecto Personalizado o Extensible Storage

Consiste en almacenar un GUID personalizado generado por el Add-in al momento de vincular el modelo por primera vez.

**Comportamiento:**

Si el Add-in no detecta el parámetro `CQEING_Model_ID`, genera uno nuevo mediante:

```csharp
Guid.NewGuid().ToString()
```

Posteriormente lo almacena en el archivo y lo envía a la base de datos.

**Ventajas:**

* Control total sobre la identificación del modelo.
* Permite implementar funcionalidades de desvinculación o clonación segura cuando un usuario crea una copia del archivo.

> **Recomendación**
>
> Utilizar una estrategia híbrida:
>
> * Si el modelo es colaborativo, usar `GetWorksharingCentralGUID()`.
> * Si el modelo no es colaborativo, usar `ProjectInformation.UniqueId`.

---

## 2. Modificación en la Base de Datos (API Web)

En la base de datos (Railway / Prisma), la tabla `Presupuesto` (o la entidad equivalente que almacene la cabecera de la cuantificación) debe incluir un nuevo campo para guardar la identificación única del modelo de Revit asociado.

### Ejemplo en Prisma

```prisma
model Presupuesto {
  id             Int      @id @default(autoincrement())
  codigo         String   @unique
  descripcion    String

  // Nuevo campo para asociar el modelo de Revit
  revit_model_id String?  @unique

  // ... resto de campos
}
```

### Equivalente en PostgreSQL

Si la tabla ya existe, se puede agregar la columna y la restricción de unicidad mediante una migración:

```sql
ALTER TABLE presupuesto
ADD COLUMN revit_model_id VARCHAR(255);

ALTER TABLE presupuesto
ADD CONSTRAINT uq_presupuesto_revit_model_id
UNIQUE (revit_model_id);
```

Si la tabla se crea desde cero, la definición podría ser:

```sql
CREATE TABLE presupuesto (
    id SERIAL PRIMARY KEY,
    codigo VARCHAR(100) NOT NULL UNIQUE,
    descripcion TEXT,
    revit_model_id VARCHAR(255) UNIQUE
);
```

### Consideraciones

* `revit_model_id` debe ser opcional (`nullable`) durante la etapa inicial de adopción.
* Cuando un presupuesto se crea desde la aplicación web o se importa desde una fuente externa, aún no tendrá un modelo de Revit asociado.
* PostgreSQL permite múltiples registros con valor `NULL` en una columna marcada como `UNIQUE`, por lo que varios presupuestos sin vincular pueden coexistir sin conflicto.
* Una vez asignado un valor a `revit_model_id`, la restricción de unicidad garantiza que un mismo modelo de Revit no pueda asociarse a más de un presupuesto.


### Consideraciones

* `revit_model_id` debe ser opcional (`nullable`) durante la etapa inicial de adopción.
* Cuando un presupuesto se crea desde la aplicación web o se importa desde una fuente externa, aún no tendrá un modelo de Revit asociado.
* Una vez que el Add-in vincule el modelo, este campo almacenará el identificador único correspondiente.
* La restricción `@unique` garantiza que un mismo modelo de Revit no pueda asociarse a múltiples presupuestos.


## 3. Flujo de Validación en el Add-in

Cuando el usuario abre el Add-in y selecciona un presupuesto, el sistema debe validar que dicho presupuesto esté asociado al modelo de Revit actualmente abierto.

### Flujo de Validación

```text
┌────────────────────────────────────────┐
│ Usuario carga Presupuesto en el Add-in │
└────────────────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────┐
│ Obtener ID del Modelo de Revit Activo │
└───────────────────────────────────────┘
                    │
                    ▼
◇────────────────────────────────────────────◇
│ ¿El presupuesto ya tiene 'revit_model_id'? │
◇────────────────────────────────────────────◇
            │                      │
           NO                     SÍ
            │                      │
            ▼                      ▼
┌───────────────────────┐    ◇────────────────────────────────◇
│ Asociar modelo actual │    │ ¿ID en DB == ID Modelo actual? │
│ y guardar ID en la DB │    ◇────────────────────────────────◇
└───────────────────────┘                │
                                         │
                           ┌─────────────┴─────────────┐
                           │                           │
                          SÍ                          NO
                           │                           │
                           ▼                           ▼
             ┌───────────────────────────┐  ┌─────────────────────────────────┐
             │ Permitir edición y carga  │  │ Bloquear edición y mostrar      │
             │ de elementos              │  │ alerta de conflicto             │
             └───────────────────────────┘  └─────────────────────────────────┘
```

### Reglas de Negocio

1. Cuando un presupuesto no tiene un `revit_model_id` asociado, el Add-in debe vincular automáticamente el presupuesto al modelo de Revit actualmente abierto.
2. El identificador obtenido del modelo debe almacenarse en la base de datos.
3. En aperturas posteriores, el Add-in debe comparar el identificador almacenado con el identificador del modelo activo.
4. Si ambos identificadores coinciden, el usuario puede continuar trabajando normalmente.
5. Si los identificadores son diferentes, se debe bloquear cualquier operación de asociación, edición o sincronización de elementos para evitar corrupción de datos.

### Mensaje de Alerta Sugerido

> [!CAUTION]
> **Conflicto de Asociación de Modelo**
>
> Este presupuesto ya está vinculado al modelo de Revit con ID **[ID_VINCULADO]**.
>
> Estás intentando abrirlo desde el modelo con ID **[ID_ACTUAL]**.
>
> Para evitar la corrupción de datos y la mezcla de elementos entre proyectos, se ha bloqueado la asociación y sincronización de elementos en este archivo.

### Resultado Esperado

Con esta validación se garantiza una relación exclusiva **1:1** entre un presupuesto y un modelo de Revit, evitando que elementos de distintos proyectos se mezclen accidentalmente dentro de la misma cuantificación.

