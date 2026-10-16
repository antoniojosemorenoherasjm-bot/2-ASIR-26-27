# GitLab vs OpenProject

## ¿Qué es GitLab?
Plataforma para **desarrollar software en equipo**. Su núcleo es **Git**: guarda el código, registra cada cambio y permite trabajar a varias personas sin pisarse.
Alrededor de eso añade:
- **Repositorios** con historial de cambios y ramas.
- **Issues** (tareas), **tableros Kanban** e **hitos** para organizar el trabajo.
- **Merge requests**: revisar los cambios de un compañero antes de aceptarlos.
- **CI/CD**: automatizar pruebas y despliegues cada vez que se sube código.
- **Wiki** para documentación.

Es como GitHub, pero más orientado a DevOps.

## ¿Qué es OpenProject?
Herramienta de **gestión de proyectos**. No guarda código: organiza el trabajo.
- **Paquetes de trabajo** (tareas) con responsable, fechas y estado.
- **Diagramas de Gantt** y cronogramas.
- **Tableros ágiles** (Scrum/Kanban) y backlogs.
- **Control de tiempo** dedicado a cada tarea.
- Wiki, foros, reuniones y presupuestos.

Es como un Microsoft Project o un Jira de código abierto.

## Comparativa

| | GitLab | OpenProject |
|---|---|---|
| Propósito | Código + DevOps | Planificación y seguimiento |
| Guarda código (Git) | Sí | No |
| Tareas y Kanban | Sí, básicos | Sí, muy completos |
| Diagrama de Gantt | Limitado | Sí, su punto fuerte |
| Control de horas | Básico | Completo |
| Nube gratuita | **Sí** (gitlab.com) | **No** (solo prueba de 14 días; luego es de pago) |
| Autoalojado gratis | Sí | Sí (edición Community) |
| Recursos si se autoaloja | Pesado: 8-16 GB de RAM y unos 40 GB de disco | Más ligero |

## El matiz importante
Habéis decidido usar **la nube** para no cargar el servidor. Eso cambia la comparación:
- **OpenProject en la nube no es gratis.** Si lo queréis gratis, hay que autoalojarlo, que es justo lo que queríais evitar.
- **GitLab sí es gratis en la nube**, y es justo el que más consumiría en local. En la nube, ese problema desaparece.

## Recomendación: GitLab (gitlab.com)
1. **Es gratis en la opción que habéis elegido.** OpenProject solo lo sería instalándolo.
2. **Cubre lo que necesitáis.** Con issues, tableros e hitos podéis organizaros sin una herramienta aparte.
3. **Encaja con lo que ya hacéis.** Usáis GitHub para las ideas y en IAW estáis empezando con Git. GitLab es casi lo mismo, así que la curva de aprendizaje es mínima.
4. **Suma al perfil de ASIR.** Git y CI/CD se piden en sistemas y DevOps. Un Gantt bonito pesa menos en la defensa.

**Cuándo elegiría OpenProject:** si el profesor exige planificación formal (Gantt, horas por persona) o si el proyecto apenas tiene código. En ese caso, merece la pena autoalojarlo: es mucho más ligero que GitLab y no os quitaría recursos para el resto de servicios.

**Pregunta que deberíais resolver antes:** ¿ya usáis GitHub? Si es así, GitHub tiene *Projects* (tableros y tareas) y podríais no necesitar ninguna de las dos. Cambiar a GitLab solo compensa si queréis aprender una herramienta nueva o usar su CI/CD.
