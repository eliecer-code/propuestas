# Reporte técnico — Identificación de modelos Revit y vinculación con presupuestos

**Proyecto:** Add-in Cuadro de Cantidades
**Módulo:** Vinculación de modelos Revit con presupuestos
**Fecha:** 14 de agosto de 2026
**Estado:** Investigación completada — pendiente de decisión funcional

---

## 1. Resumen ejecutivo

Durante las pruebas del add-in **Cuadro de Cantidades** se identificó un conflicto al intentar vincular un modelo local de Revit con un presupuesto diferente cuando dicho modelo local proviene de un modelo central que ya se encuentra vinculado a otro presupuesto.

El comportamiento observado es:

```text
Modelo Central
      │
      ├── Presupuesto A
      │
      ├── Local de Juan
      │
      └── Local de María
```

Los modelos locales derivados del mismo Central son identificados por el add-in como pertenecientes al mismo modelo/proyecto.

Esto provoca que, si el Central ya está vinculado al **Presupuesto A**, intentar vincular uno de sus modelos locales al **Presupuesto B** genere un conflicto de vinculación.

El análisis realizado sobre el código concluye que **el comportamiento actual del identificador no es un error técnico del add-in**, sino que está diseñado para considerar los modelos locales pertenecientes al mismo Central como una misma identidad de modelo.

Por esta razón, modificar simplemente el identificador para que cada Local sea considerado un modelo independiente podría generar problemas de integridad de datos en escenarios colaborativos.

---

# 2. Problema identificado

El escenario que genera el conflicto es el siguiente:

1. Se dispone de un archivo `.rvt` que funciona como plantilla.
2. El archivo se convierte en un modelo Central.
3. El modelo Central se vincula al **Presupuesto A**.
4. Los usuarios crean modelos Locales para trabajar colaborativamente.
5. El add-in identifica esos Locales como pertenecientes al mismo modelo Central.
6. Se intenta vincular un Local al **Presupuesto B**.
7. El sistema detecta que el modelo ya está vinculado.
8. Se muestra el error:

> "No se pudo vincular el modelo al presupuesto. Verifique la API o si el modelo ya está vinculado a otro presupuesto."

---

# 3. Comportamiento actual del identificador

Durante la auditoría se revisó específicamente:

```text
RevitModelIdentifier.cs
```

El análisis determinó que la implementación actual está diseñada para identificar correctamente un proyecto colaborativo.

En otras palabras:

```text
Central
   │
   ├── Local Juan
   │
   └── Local María
```

son considerados parte de una misma identidad de modelo.

Esto tiene sentido desde el punto de vista de un sistema colaborativo, ya que todos los usuarios están trabajando sobre el mismo proyecto.

---

# 4. ¿Por qué no es recomendable simplemente cambiar el Unique ID?

Una solución aparentemente sencilla sería generar un identificador diferente para cada Local.

Sin embargo, esto podría producir inconsistencias.

Por ejemplo:

```text
Central
 │
 ├── Local Juan → Presupuesto B
 │
 └── Local María → Presupuesto C
```

Juan podría modificar muros y sincronizarlos con el Central.

Posteriormente María podría modificar vigas y sincronizarlas.

Si cada Local tuviera una identidad independiente, el sistema podría terminar interpretando que:

```text
Muros → Presupuesto B

Vigas → Presupuesto C
```

aunque ambos elementos pertenezcan al mismo proyecto colaborativo.

Esto podría provocar problemas de integridad y sincronización de información.

---

# 5. Alternativas analizadas

Durante la investigación se contemplaron diferentes alternativas para resolver el conflicto.

## Alternativa A — Mantener el comportamiento actual

Mantener el identificador actual y considerar:

```text
Central + Locales
```

como una misma identidad de proyecto.

### Ventajas

* Mantiene la integridad del modelo colaborativo.
* Evita que diferentes usuarios trabajen accidentalmente sobre presupuestos diferentes.
* No requiere modificar la arquitectura actual.
* No genera migraciones de datos.

### Desventajas

* Un Local no puede utilizarse directamente para representar un proyecto independiente.
* No permite reutilizar directamente un Central ya vinculado para crear otro presupuesto.

---

## Alternativa B — Identificador diferente para cada Local

Generar un identificador diferente para:

```text
Central
Local Juan
Local María
```

### Ventajas

* Cada Local podría vincularse a un presupuesto diferente.

### Desventajas

Esta alternativa presenta un riesgo importante para la integridad de los datos.

Ejemplo:

```text
Local Juan
    ↓
Presupuesto B

Local María
    ↓
Presupuesto C
```

Si ambos sincronizan cambios con el mismo Central, diferentes elementos del mismo proyecto podrían terminar asociados a diferentes presupuestos.

### Impacto

**Alto riesgo funcional en escenarios colaborativos.**

No se recomienda.

---

## Alternativa C — Revisar el flujo de negocio

Mantener la identificación actual de los modelos colaborativos y definir correctamente cuándo un modelo debe considerarse:

* el mismo proyecto colaborativo;
* una copia independiente;
* un nuevo proyecto.

Esta alternativa implica establecer una regla clara sobre el flujo de trabajo antes de modificar el código.

---

## Alternativa D — ID personalizado inyectado

Otra posibilidad sería almacenar dentro del `.rvt` un identificador personalizado que permita controlar explícitamente la identidad que utiliza el add-in.

### Ventajas

* Mayor control sobre la identidad.
* Permite definir explícitamente cuándo un modelo representa un proyecto nuevo.
* Puede desacoplar la identificación del archivo físico.

### Desventajas

* Requiere escribir información adicional dentro del `.rvt`.
* Aumenta la complejidad del add-in.
* Requiere definir cuidadosamente cuándo se genera o cambia el identificador.
* Tiene impacto medio sobre la arquitectura actual.

---

# 6. Recomendación técnica

La recomendación obtenida durante la auditoría es **no modificar actualmente `RevitModelIdentifier.cs` para convertir cada Local en un modelo independiente**.

La implementación actual está correctamente orientada hacia escenarios colaborativos.

El problema parece estar principalmente relacionado con el **flujo de negocio esperado**.

Si el objetivo es utilizar un proyecto que ya fue vinculado a un presupuesto como base para crear un proyecto/presupuesto completamente diferente, el flujo recomendado sería:

```text
Modelo existente
       │
       ▼
Separar del archivo central (Detach)
       │
       ▼
Guardar como nuevo modelo
       │
       ▼
Convertir/crear nuevo Central
       │
       ▼
Nuevo identificador de modelo
       │
       ▼
Vincular al Presupuesto B
```

De esta forma se mantiene:

```text
Modelo Central original
        ↓
Presupuesto A
```

y se crea un modelo completamente independiente:

```text
Nuevo Central
        ↓
Presupuesto B
```

Sin generar conflictos entre ambos.

---

# 7. Escenario recomendado

### Proyecto original

```text
Modelo Central A
       │
       ├── Local Juan
       ├── Local María
       └── Presupuesto A
```

Todos los Locales deben continuar perteneciendo al mismo proyecto.

### Nuevo proyecto

Si se desea utilizar el modelo como base para otro presupuesto:

```text
Modelo Central A
       │
       ▼
Detach
       │
       ▼
Nuevo Modelo
       │
       ▼
Nuevo Central B
       │
       ▼
Presupuesto B
```

Esto permite separar completamente las identidades de ambos proyectos.

---

# 8. Riesgos de modificar actualmente el identificador

Modificar el comportamiento actual para generar un ID diferente para cada Local podría generar:

* Duplicidad de proyectos.
* Diferentes presupuestos asociados al mismo modelo colaborativo.
* Información fragmentada.
* Elementos del mismo proyecto asociados a diferentes presupuestos.
* Problemas al sincronizar cambios.
* Dificultades para mantener la trazabilidad.
* Posibles inconsistencias entre Revit y la base de datos.

Por lo tanto, **no se recomienda realizar esta modificación sin definir primero la regla de negocio**.

---

# 9. Decisión pendiente

Antes de modificar el código es necesario definir con el responsable del proyecto cuál de los siguientes comportamientos se desea.

### Opción 1 — Un proyecto colaborativo = un presupuesto

```text
Central
 ├── Local 1
 ├── Local 2
 └── Local 3
        ↓
   Un presupuesto
```

En este caso, la implementación actual del identificador es adecuada.

---

### Opción 2 — Cada Local puede representar un presupuesto independiente

```text
Central
 ├── Local 1 → Presupuesto A
 ├── Local 2 → Presupuesto B
 └── Local 3 → Presupuesto C
```

Esta opción requiere una modificación importante de la arquitectura de identificación y sincronización y debe analizarse antes de implementarse.

---

### Opción 3 — El Central se utiliza como plantilla para nuevos proyectos

En este caso se recomienda:

```text
Central original
       │
       ▼
Detach
       │
       ▼
Nuevo modelo
       │
       ▼
Nuevo Central
       │
       ▼
Nuevo presupuesto
```

Esta opción mantiene separadas las identidades y evita conflictos.

---

# 10. Conclusión

La auditoría determinó que el comportamiento actual de `RevitModelIdentifier.cs` **no debe considerarse un error por sí mismo**.

El identificador está diseñado para reconocer que los modelos Locales pertenecen al mismo Central, lo cual es coherente con el funcionamiento de Revit en un entorno colaborativo.

El conflicto aparece cuando se intenta utilizar un Local de un Central ya vinculado como si fuera un proyecto completamente independiente y vincularlo a otro presupuesto.

Por tanto:

> **No se recomienda modificar el identificador actual hasta definir formalmente el comportamiento esperado para proyectos colaborativos.**

La recomendación técnica actual es mantener la lógica de identificación y, cuando se necesite crear un proyecto/presupuesto completamente independiente a partir de un modelo Central existente, utilizar el flujo de **Detach / Separar del archivo central**, guardar el resultado como un nuevo modelo y posteriormente crear el nuevo Central.

---

# 11. Estado actual

| Elemento                              | Estado          |
| ------------------------------------- | --------------- |
| Auditoría del identificador           | ✅ Completada    |
| Revisión de `RevitModelIdentifier.cs` | ✅ Completada    |
| Identificación de causa               | ✅ Completada    |
| Modificación de código                | ⏸️ No realizada |
| Modificación de BD                    | ⏸️ No realizada |
| Decisión de arquitectura              | ⏳ Pendiente     |
| Validación con Product Owner          | ⏳ Pendiente     |

---

# 12. Próximo paso recomendado

Antes de realizar cualquier modificación al add-in, validar con el responsable del proyecto esta regla:

> **¿Un modelo Central y todos sus Locales deben obligatoriamente representar un único proyecto/presupuesto, o se requiere que un Local pueda convertirse en una identidad independiente y vincularse a otro presupuesto?**

La respuesta determinará si es necesario modificar la arquitectura de identificación del add-in o si el comportamiento actual debe mantenerse.
