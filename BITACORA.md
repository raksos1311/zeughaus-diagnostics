# Bitácora de diagnóstico — zeughaus

**Sistema:** Fedora 43 / kernel 7.1.6-101.fc43.x86_64  
**Filesystem raíz:** Btrfs en `/dev/nvme0n1p5`, subvolumen `root`  
**Inicio de investigación:** 2026-08-13/14

## Objetivos

1. Determinar por qué `/` queda en `ro` después de algunos arranques.
2. Determinar si Btrfs, NVMe o el proceso `ro → rw` de systemd están implicados.
3. Determinar por qué algunos servicios, especialmente `libvirtd`, no arrancan cuando ocurre.
4. Corregir el problema de forma permanente, no mediante `remount,rw` manual.
5. Corregir el montaje SSHFS de `heinrici` para que un servidor apagado no bloquee Nautilus.
6. Conseguir que el journal persistente conserve correctamente los boots para poder diagnosticar futuros incidentes.
7. Revisar y dejar correctamente configurado `systemd --user` + linger para servicios persistentes de usuario.

## Incidente de referencia

- GDM comenzó a bloquearse.
- Una TTY también terminó bloqueándose.
- Se utilizó REISUB.
- Tras reiniciar, el sistema arrancó, pero `libvirtd` no inició.
- El journal indicó que el sistema estaba en `ro`.
- Se ejecutó un remount manual a `rw` y `libvirtd` arrancó correctamente.
- Al día siguiente volvió a producirse un problema relacionado con acceso a `heinrici` apagado y Nautilus quedó bloqueado.

## Pruebas realizadas

### Prueba 001 — Estado actual de `/`

**Resultado: OK.**

`/` está actualmente montado `rw`.

### Prueba 002 — `systemd-remount-fs` en el arranque actual

**Resultado: OK.**

`systemd-remount-fs.service` terminó con `status=0/SUCCESS`.

El kernel arranca inicialmente con `root=... ro`; esto es normal. El remount posterior funciona en este arranque.

### Prueba 003 — Estadísticas Btrfs

**Resultado: SIN ERRORES REGISTRADOS.**

Todos los contadores de `btrfs device stats` están en cero: write/read/flush/corruption/generation errors.

### Prueba 004 — Kernel/Btrfs/NVMe en arranque actual

**Resultado: sin errores Btrfs/NVMe relevantes.**

Se observaron mensajes ACPI/HP BIOS, pero no hay por ahora evidencia de que sean la causa del `ro`.

### Prueba 005 — SSHFS de heinrici

**Resultado: PROBLEMA IDENTIFICADO, pendiente de corrección.**

Existen dos montajes SSHFS:

- `/home/oscar/scans`: ya utiliza `x-systemd.automount` y `x-systemd.idle-timeout=600`.
- `/home/oscar/heinrici`: se inicia mediante `~/.config/autostart/heinrici.desktop` con `sshfs -o reconnect,...`.

El segundo es candidato claro a bloquear Nautilus cuando `heinrici` está apagado. Se migrará a un montaje administrado por systemd/automount después de preservar el estado actual.

### Prueba 006 — Persistencia del journal

**Resultado: DIAGNÓSTICO PARCIALMENTE RESUELTO.**

Se comprobó que `/var/log/journal` existe y es escribible. `journalctl --verify` no mostró corrupción. El journal persistente disponible correspondía únicamente a abril, mientras que el boot actual estaba inicialmente en `/run/log/journal`.

La prueba manual de escritura en `/var/log/journal` tuvo éxito y `sudo journalctl --flush` terminó correctamente. Después del flush aparecieron journals persistentes nuevos correspondientes al boot de agosto y `journalctl --list-boots` volvió a mostrar el boot actual.

Conclusión: **journald sí puede escribir en almacenamiento persistente; no hay evidencia de corrupción o permisos incorrectos.** Queda por determinar por qué el journal automático no estaba siendo persistido/volcado normalmente durante el arranque.

### Prueba 007 — Cronología de `systemd-remount-fs`

**Resultado: CORRECCIÓN DE HIPÓTESIS.**

Inicialmente parecía que `systemd-remount-fs` se ejecutaba cuatro horas después del montaje de Btrfs porque los timestamps del journal mostraban, por ejemplo, Btrfs a las 14:59 y remount a las 18:59.

`systemd-analyze time` demuestra que el boot completo tardó solamente **6 min 40,516 s**, y que `graphical.target` se alcanzó después de **1 min 6,824 s** de userspace. Por tanto, el remount no estuvo realmente esperando cuatro horas.

La diferencia de aproximadamente cuatro horas entre timestamps del kernel y systemd apunta fuertemente a un **salto/ajuste del reloj del sistema durante el arranque**. La hipótesis de que `systemd-remount-fs` estuviera bloqueado cuatro horas queda **descartada**.

### Prueba 008 — Localización del evento `rw → ro`

**Resultado: PRUEBA INCONCLUYENTE / CORREGIDA.**

El primer archivo generado quedó vacío por un problema con el filtro utilizado. No se extrae ninguna conclusión de ese archivo.

Se repitió manualmente la búsqueda sobre el boot actual y posteriormente sobre `boot -1`.

### Prueba 009 — Búsqueda dirigida en boot actual

**Resultado: BOOT SANO.**

El boot actual muestra:

- Btrfs detectado y montado alrededor de 3,9 s.
- `systemd-remount-fs` funciona alrededor de 10 s.
- No aparecen errores Btrfs.
- No aparecen `I/O error`, `timeout`, `reset`, `forced readonly` ni errores NVMe relevantes.

Por tanto, el boot actual **no contiene evidencia de una transición `rw → ro`**.

### Prueba 010 — Búsqueda dirigida en `boot -1`

**Resultado: NO ES EL BOOT DEL INCIDENTE RECIENTE.**

`journalctl --list-boots` mostró que el `boot -1` disponible corresponde al **17 de abril**, no al reinicio con REISUB ocurrido recientemente.

La búsqueda de Btrfs/NVMe/RO en ese boot de abril tampoco mostró una causa del problema actual.

Esto es importante: **el boot problemático reciente no está disponible en el journal persistente**, por lo que no podemos hacer una autopsia retrospectiva de ese incidente.

El kernel del boot actual y del boot de abril muestra `Previous system reset reason [0x00080800]: software wrote 0x6 to reset control register 0xCF9`, compatible con un reset provocado por software como REISUB; esto no constituye evidencia de fallo de hardware.

### Prueba 011 — Arquitectura de journald y flush persistente

**Resultado: JOURNAL PERSISTENTE FUNCIONANDO, PERO FLUSH TARDÍO OBSERVADO.**

La configuración efectiva mantiene `Storage=auto` (valor por defecto). `/var/log/journal` existe desde abril, tiene permisos/SELinux context correctos y actualmente contiene journals persistentes.

`systemd-journal-flush.service` termina con `status=0/SUCCESS` a las 18:59:22, pero `systemd-journald` registra el flush efectivo hacia `/var/log/journal` a las 22:17:19, cuando también cambia la fecha de modificación de `/var/log/journal`.

No se ha demostrado todavía por qué existe esta diferencia. Queda pendiente correlacionar qué ocurrió alrededor de las 22:17:19.

### Prueba 012 — Escritura real RW y disponibilidad de `/run`

**Resultado: OK.**

`/` está montado `btrfs rw` y `/run` está montado `tmpfs rw`.

Las primeras pruebas de `touch` como usuario `oscar` fallaron con `Permiso denegado`, lo que fue correctamente identificado como un problema de permisos de la prueba, no del filesystem.

Las pruebas corregidas con `sudo` demostraron:

- escritura real en `/root`: **OK**;
- escritura real en `/run`: **OK**;
- prueba controlada de escritura en `/root`: `RESULT=RW`.

Esto valida el mecanismo que utilizará el watchdog: **detectar RO mediante una escritura real y no solamente leyendo las opciones de `findmnt`.**

### Prueba 013 — Diseño e incorporación del watchdog forense v0.1

**Resultado: IMPLEMENTADO EN REPOSITORIO; NO INSTALADO.**

Se incorporó `tools/zeughaus-ro-watch`, versión 0.1, y su unidad `systemd/zeughaus-ro-watch.service`.

El watchdog fue diseñado para ejecutarse como root, probar una escritura real en `/root` cada 10 segundos, guardar evidencia bajo `/run/zeughaus-ro-watch/` y no ejecutar `remount,rw` automáticamente.

### Prueba 014 — Revisión y endurecimiento del watchdog v0.2

**Resultado: IMPLEMENTADO EN REPOSITORIO; PENDIENTE DE INSTALACIÓN/PRUEBA.**

La versión 0.1 fue revisada antes de instalarse. Se detectaron dos riesgos: una captura forense podía quedar esperando indefinidamente en un comando bloqueado y `sync` podía introducir una espera precisamente durante un incidente de I/O.

La versión 0.2 incorpora:

- timeout de 15 s por comando forense, con 2 s adicionales para matar procesos que no terminen;
- eliminación de `sync` de la ruta crítica;
- `INCIDENT_TRIGGER.txt` escrito **antes** de las operaciones pesadas;
- captura de `/proc/mounts` y `/proc/self/mountinfo` además de `findmnt`;
- captura de `btrfs filesystem df` y del subvolumen por defecto;
- retest independiente de escritura mediante Python para conservar errno/ERRNAME;
- conservación de `rc` y mensaje de error de la primera escritura fallida;
- almacenamiento de la evidencia exclusivamente bajo `/run`, sin intentar reparar el filesystem.

La versión 0.2 queda preparada para una primera prueba controlada antes de habilitarla permanentemente.

## Hipótesis actuales

| ID | Hipótesis | Prioridad | Estado |
|---|---|---:|---|
| H1 | El remount `ro → rw` falla durante determinados boots | Alta | No demostrado; boot actual OK |
| H2 | Btrfs detecta una condición y remonta `ro` | Alta | Sin evidencia en boots disponibles |
| H3 | Otro componente vuelve a montar `/` como `ro` | Alta | En investigación |
| H4 | SSHFS persistente de heinrici bloquea Nautilus | Alta | Evidencia fuerte |
| H5 | Problema físico/controlador del NVMe no reflejado en Btrfs stats/SMART | Media-baja | No demostrado |
| H6 | Journald no está haciendo correctamente el flush/persistencia durante determinados arranques | Alta | Evidencia parcial; flush efectivo tardío observado |
| H7 | Salto de reloj durante el arranque explica la diferencia 14:59→18:59 observada en timestamps | Alta | Evidencia fuerte |
| H8 | `home-oscar-scans.mount` contribuye a demoras/bloqueos durante el arranque | Media | Evidencia: ~42 s en `systemd-analyze blame` |

## Próximo paso

Hacer `git pull` y revisar/instalar el watchdog forense v0.2 en zeughaus. Primero se probará manualmente/como servicio sin habilitarlo permanentemente. Una vez validado, se dejará activo para capturar el próximo incidente real `rw → ro`.

Cuando ocurra un incidente, **no reiniciar inmediatamente** si la interfaz sigue respondiendo: primero se preservará la evidencia bajo `/run/zeughaus-ro-watch/`. Después de restaurar `/` a `rw`, se utilizará `zeughaus-ro-harvest` para copiarla al repositorio y analizarla.

No ejecutar todavía `btrfs check`, scrub ni reparaciones a ciegas.

## Pendientes posteriores

- Migrar `/home/oscar/heinrici` desde `~/.config/autostart/heinrici.desktop` a systemd user + automount, evitando bloqueos de Nautilus cuando `heinrici` está apagado.
- Revisar `loginctl`/`systemd --user` y configurar linger correctamente para servicios persistentes.
- Investigar el error histórico de sincronización de caché de `/dev/sda` (KIOXIA `KBG40ZNV512G`) visto en el boot de abril, separándolo del NVMe raíz.
- Corregir posteriormente la configuración RTC/UTC, pero sin mezclar esa modificación con el diagnóstico del filesystem.

## Regla de trabajo

No se harán cambios permanentes basados únicamente en síntomas. Cada modificación debe tener:

1. hipótesis que pretende corregir;
2. cambio concreto;
3. prueba posterior;
4. resultado registrado;
5. posibilidad de revertirlo.
