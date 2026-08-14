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

### Prueba 013 — Diseño e incorporación del watchdog forense

**Resultado: IMPLEMENTADO EN REPOSITORIO; PENDIENTE DE INSTALACIÓN/PRUEBA.**

Se incorporó `tools/zeughaus-ro-watch` y su unidad `systemd/zeughaus-ro-watch.service`.

El watchdog ejecuta como root, prueba una escritura real en `/root` cada 10 segundos, captura un incidente cuando la escritura falla y guarda la evidencia bajo `/run/zeughaus-ro-watch/`.

No intenta hacer `remount,rw` automáticamente.

### Prueba 014 — Watchdog v0.2 endurecido

**Resultado: SUPERADO EN DISEÑO; reemplazado antes de instalación por v0.3.**

La v0.2 añadió timeouts de 15 s por operación forense, eliminó `sync` de la ruta crítica, escribió `INCIDENT_TRIGGER.txt` antes de las operaciones pesadas y amplió la captura con `/proc/mounts`, `/proc/self/mountinfo` y más información Btrfs.

Antes de instalarla se detectó que la prueba normal todavía dependía de analizar el texto producido por `bash`, lo que podía ser sensible a idioma/localización. Por ello se corrigió nuevamente antes de la primera instalación.

### Prueba 015 — Watchdog v0.3: errno real y clasificación del incidente

**Resultado: IMPLEMENTADO EN REPOSITORIO; PENDIENTE DE INSTALACIÓN/PRUEBA.**

La v0.3 convierte la prueba de escritura principal en una operación Python que obtiene directamente el `errno` del kernel. Ya no se intenta inferir `EROFS` a partir del texto localizado del error.

Se distinguen explícitamente:

- `ro-incident`: `ERRNO=30` / `ERRNAME=EROFS`;
- `write-failure`: cualquier otro fallo de escritura, que se conserva como pista separada y **no se interpreta automáticamente como filesystem RO**.

Además se mantiene la captura temprana del incidente, los timeouts, la captura de `btrfs property get -ts / ro`, el registro de duración de la indisponibilidad y la ausencia de reparación automática.

### Prueba 016 — Corrección del watchdog v0.4

**Resultado: CORREGIDO EN REPOSITORIO; PENDIENTE DE INSTALACIÓN/PRUEBA.**

Se detectó un bug en v0.3: el bloque de captura ejecutaba la función Bash `write_test` directamente como argumento de `timeout`. `timeout` necesita ejecutar un programa externo; una función Bash no puede utilizarse de esa forma.

La v0.4 corrige la ruta de la prueba forense para que pueda ejecutarse bajo un límite temporal y conserva el mecanismo de obtención directa de `errno` mediante Python.

La sintaxis local de `tools/zeughaus-ro-watch` fue validada con:

```bash
bash -n tools/zeughaus-ro-watch
```

**Resultado: OK.**

Commit de corrección: `2bec0d904207f2f6c5a6b747658ac83b63cc1883`.

### Prueba 017 — Definición del servicio systemd

**Resultado: DEFINIDO EN REPOSITORIO; PENDIENTE DE INSTALACIÓN.**

Se actualizó `systemd/zeughaus-ro-watch.service` para ejecutar exactamente el binario que ya fue instalado localmente en `/usr/local/sbin/zeughaus-ro-watch`.

Configuración registrada:

- `Type=simple`;
- `Restart=always`;
- `RestartSec=5`;
- ejecución como `root`;
- `UMask=0077`;
- evidencia bajo `/run`;
- sin `ProtectSystem`, porque el watchdog debe poder observar/escribir su prueba sobre `/` antes de que el filesystem pueda pasar a RO;
- `WantedBy=multi-user.target` para su posterior habilitación.

El servicio **todavía no está instalado ni habilitado** en el sistema en este punto de la investigación.

### Prueba 018 — Intento de arranque antes de crear la unidad local

**Resultado: ESPERADO / SIN EFECTO.**

Se ejecutó:

```bash
sudo systemctl start zeughaus-ro-watch.service
```

systemd respondió:

```text
Unit zeughaus-ro-watch.service not found.
```

Esto confirma que no existía una unidad local activa antes de la instalación planificada.

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

Hacer `git pull` y revisar/instalar el unit file del watchdog. Después se ejecutará `daemon-reload`, se arrancará **sin habilitarlo todavía para el boot** y se verificará su funcionamiento durante la sesión actual. Sólo después de esa prueba se habilitará para futuros arranques.

Cuando ocurra un incidente, **no reiniciar inmediatamente** si la interfaz sigue respondiendo: primero se preservará la evidencia bajo `/run/zeughaus-ro-watch/`. Después de restaurar `/` a `rw`, se utilizará el procedimiento de cosecha para copiarla al repositorio y analizarla.

No ejecutar todavía `btrfs check`, scrub ni reparaciones a ciegas.

## Desinstalación posterior obligatoria

Este watchdog es una herramienta **temporal de diagnóstico**, no una configuración permanente del sistema.

Al finalizar la investigación:

1. detener el servicio;
2. deshabilitarlo si fue habilitado;
3. eliminar `/etc/systemd/system/zeughaus-ro-watch.service`;
4. eliminar `/usr/local/sbin/zeughaus-ro-watch`;
5. ejecutar `systemctl daemon-reload`;
6. conservar en el repositorio el registro de instalación, funcionamiento y eliminación.

La bitácora debe reflejar la configuración real utilizada, no una versión idealizada del sistema.

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
