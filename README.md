# Plataforma Académica de Virtualización con Proxmox VE

## Descripción

Este proyecto propone una plataforma organizada de virtualización para alojar y administrar entornos de proyectos universitarios mediante **Proxmox VE**. La propuesta busca separar los servicios de producción, los laboratorios docentes, el desarrollo y las pruebas, de modo que cada entorno tenga recursos, permisos y políticas de respaldo definidos.

> **Estado del proyecto:** documento de diseño y planificación. Las capacidades indicadas como objetivo y las configuraciones propuestas no deben interpretarse como recursos ya instalados ni como funciones que estén actualmente operativas.

## Objetivo general

Diseñar una plataforma de virtualización segura, organizada y ampliable para administrar máquinas virtuales y contenedores de proyectos universitarios, aprovechando de forma controlada el servidor físico disponible.

### Objetivos específicos

- Separar los entornos académicos por finalidad y nivel de criticidad.
- Definir una distribución inicial de recursos de cómputo, memoria y almacenamiento.
- Establecer controles de acceso para administradores, docentes y usuarios autorizados.
- Proponer procedimientos de respaldo y recuperación verificables.
- Identificar las limitaciones del servidor actual y las mejoras necesarias para crecer.
- Preparar la plataforma para futuras tareas de automatización y supervisión.

## Infraestructura de referencia

La auditoría técnica utilizada para la propuesta registró los siguientes datos generales:

| Componente | Estado observado en la auditoría |
|---|---|
| Plataforma | Proxmox VE sobre Debian GNU/Linux 13 |
| Servidor físico | HP Z820 Workstation |
| Procesadores | Intel Xeon E5-2630 v2; el conteo de CPU lógicas del informe requiere validación adicional |
| Memoria RAM | Aproximadamente 15 GiB detectados; alrededor de 12 GiB estaban en uso durante la captura |
| Swap | 8 GiB configurados; aproximadamente 973 MiB en uso durante la captura |
| Disco principal | Aproximadamente 298 GiB, con volúmenes LVM para el sistema y el almacenamiento de máquinas virtuales |
| Discos secundarios | Dos discos de aproximadamente 931.5 GiB cada uno; su contenido, salud y espacio libre deben verificarse antes de asignarlos |

Las capacidades de los discos son tamaños aproximados detectados, **no una confirmación de espacio libre**. Antes de modificar almacenamiento, se debe revisar su contenido, estado de salud y configuración actual.

Por seguridad, este README no publica direcciones IP, credenciales, tokens ni otros datos de acceso del servidor.

## Arquitectura lógica propuesta

La plataforma se organiza en cuatro entornos separados:

| Entorno | Propósito | vCPU propuestos | RAM objetivo | Almacenamiento objetivo |
|---|---|---:|---:|---:|
| Producción | Servicios estables que deban mantenerse disponibles | 6 | 128 GiB | 200 GiB |
| Laboratorios docentes | Prácticas y proyectos de las asignaturas, con separación entre docentes | 28 en total | 256 GiB en total | 400 GiB |
| Desarrollo | Construcción y configuración de proyectos | 4 | 64 GiB | 150 GiB |
| Pruebas | Validaciones temporales y experimentos controlados | 2 | 32 GiB | 80 GiB |
| **Total de recursos asignados a entornos** |  | **40 vCPU** | **480 GiB** | **830 GiB** |

La planificación contempla además reservar aproximadamente **32 GiB de RAM para el host**, por lo que el objetivo de memoria física sería de alrededor de **512 GiB**. Estos valores son una propuesta de dimensionamiento futuro, no la capacidad actual del servidor.

La asignación de CPU es una planificación inicial y debe ajustarse después de verificar la información real de los procesadores, medir la carga y probar el rendimiento. No se deben crear todas las máquinas con estos valores en el servidor actual: la RAM auditada no alcanza para desplegar esta distribución.

### Separación y administración

- Utilizar *pools* y permisos ACL de Proxmox para organizar recursos por entorno, asignatura o docente.
- Aplicar reglas de firewall y, cuando la infraestructura de red lo permita, VLAN para separar los segmentos.
- Crear plantillas de sistemas operativos aprobadas y clones para acelerar la preparación de máquinas de prácticas.
- Asignar a cada usuario únicamente los permisos necesarios para sus actividades.
- Documentar qué máquinas son de producción, desarrollo, pruebas o laboratorio.

## Tecnologías previstas

- **Proxmox VE:** administración central de virtualización.
- **KVM/QEMU:** máquinas virtuales con sistemas operativos propios.
- **LXC:** contenedores Linux ligeros que comparten el kernel del host; no sustituyen una máquina virtual cuando se necesita un kernel independiente.
- **LVM-thin:** almacenamiento usado habitualmente para discos virtuales; la disponibilidad real debe comprobarse desde Proxmox.
- **NAS o Proxmox Backup Server (PBS):** destino externo recomendado para copias de seguridad.
- **API de Proxmox:** posible base para automatizar tareas administrativas con tokens de privilegios limitados.

## Copias de seguridad y recuperación

Las copias de seguridad deben guardarse en un destino separado del almacenamiento principal del servidor, por ejemplo un NAS o una instancia independiente de Proxmox Backup Server.

Política inicial propuesta:

- **Producción:** respaldo diario, con retención definida por la institución.
- **Desarrollo y laboratorios:** respaldo semanal para las máquinas que contengan trabajo que deba conservarse.
- **Pruebas temporales:** respaldo según el valor de los datos; no conservar copias innecesarias.
- **Restauración:** realizar pruebas mensuales de recuperación y registrar el resultado.
- **Supervisión:** revisar fallos de tareas, espacio libre y antigüedad de la última copia válida.

Una **instantánea (*snapshot*)** sirve para volver rápidamente a un estado anterior, por ejemplo antes de una actualización. No equivale a una copia de seguridad independiente: si se pierde el almacenamiento que contiene la VM y su snapshot, ambos podrían perderse. Las instantáneas deben ser temporales y no reemplazar los respaldos externos.

Las tareas de respaldo pueden planificarse en la interfaz de Proxmox, en **Datacenter → Backup**, una vez definido y probado el almacenamiento de destino.

## Seguridad y operación

1. Crear cuentas individuales; evitar compartir la cuenta de administración.
2. Usar roles y permisos mínimos por entorno o *pool*.
3. No guardar contraseñas, tokens API, claves privadas ni datos sensibles dentro de este repositorio.
4. Restringir el acceso de administración a redes y usuarios autorizados.
5. Revisar periódicamente permisos, consumo de recursos, almacenamiento y registros.
6. Probar los cambios importantes primero en desarrollo o pruebas cuando sea viable.
7. Registrar las intervenciones y el resultado de las verificaciones.

### Automatizaciones futuras

- **Aprovisionamiento de laboratorios:** crear o limpiar máquinas a partir de plantillas mediante solicitudes autorizadas, límites de recursos y tokens API de privilegios mínimos.
- **Supervisión de respaldos:** detectar copias fallidas o antiguas y generar alertas. Las acciones destructivas deben requerir validación humana.

Estas automatizaciones son ideas de evolución; no se consideran implementadas hasta que existan código, configuración y evidencias de pruebas.

## Limitaciones conocidas

- La auditoría registra un único servidor físico. Si ese host falla, las máquinas virtuales que dependen de él también pueden quedar fuera de servicio.
- La alta disponibilidad y la migración automatizada entre nodos no forman parte de la capacidad demostrada de la infraestructura actual. Para una solución de alta disponibilidad se necesita diseñar y validar una infraestructura multinodo y sus requisitos de almacenamiento y red.
- Que un servicio relacionado con clúster esté activo no demuestra por sí solo que exista un clúster correctamente configurado.
- La memoria actual auditada es insuficiente para el dimensionamiento objetivo descrito en este documento.
- El estado, la ocupación y la salud de los discos secundarios aún deben verificarse antes de tomar decisiones de particionado, borrado o reasignación.
- El conteo de CPU lógicas reportado requiere una comprobación independiente antes de cerrar el dimensionamiento.

## Plan de implementación

1. **Validar el inventario:** confirmar CPU, RAM, almacenamiento, salud de discos, espacio libre y carga real.
2. **Ajustar la capacidad:** evaluar la ampliación de memoria compatible y decidir el uso de los discos tras comprobar su contenido.
3. **Organizar accesos:** definir *pools*, roles, permisos ACL y segmentación de red según las capacidades disponibles.
4. **Preparar plantillas:** crear y probar imágenes base para los sistemas operativos necesarios en los proyectos académicos.
5. **Configurar respaldos:** disponer de un destino externo, calendarizar tareas y ejecutar una restauración de prueba.
6. **Supervisar:** documentar umbrales para RAM, swap, CPU, almacenamiento y fallos de respaldo.
7. **Automatizar de forma controlada:** comenzar con tareas acotadas, tokens con permisos mínimos y registro de operaciones.
8. **Documentar evidencias:** incorporar capturas sin credenciales, IP internas visibles ni otra información sensible.

## Estructura sugerida del repositorio

La siguiente estructura es una propuesta y puede adaptarse a los archivos que ya existan en el repositorio:

```text
.
├── README.md
├── docs/
│   ├── proyecto/
│   │   └── Informe_Tecnico_Plataforma_Proxmox_Universidad.docx
│   ├── planificacion/
│   └── presentaciones/
└── evidencias/
    └── capturas/
```

Guardar en `docs/proyecto/` el informe del proyecto, en `docs/planificacion/` cronogramas y tareas, en `docs/presentaciones/` las diapositivas y en `evidencias/capturas/` las capturas de validación que no expongan información sensible.

## Criterios de validación

Antes de considerar operativa la propuesta, se debe comprobar y documentar que:

- Los recursos del host y el espacio disponible se han verificado.
- Los usuarios sólo pueden acceder a los recursos autorizados.
- Los entornos de laboratorio están separados conforme al diseño aprobado.
- Las tareas de respaldo terminan correctamente en el destino externo.
- Al menos una máquina puede restaurarse y arrancar desde una copia de seguridad.
- La supervisión detecta una tarea fallida o un umbral de capacidad alcanzado.
- La documentación y las capturas no exponen credenciales ni información sensible.

## Documentación relacionada

Consultar el informe técnico y los documentos incluidos en `docs/` cuando estén disponibles. Las decisiones de dimensionamiento deben revisarse frente a la auditoría más reciente y a las pruebas reales de carga antes de pasar de propuesta a implementación.

---

**Proyecto académico:** diseño de una plataforma de virtualización para proyectos universitarios.  
**Plataforma de referencia:** Proxmox VE.  
**Estado:** propuesta técnica pendiente de validación e implementación gradual.
