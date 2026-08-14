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

`/` está actualmente montado `rw`:

```text
/dev/nvme0n1p5[/root] btrfs rw,... subvol=/root /
```

### Prueba 002 — `systemd-remount-fs` en el arranque actual

**Resultado: OK.**

`systemd-remount-fs.service` terminó con `status=0/SUCCESS`.

El kernel arranca inicialmente con `root=... ro`; esto es normal. El remount posterior funciona en este arranque.

### Prueba 003 — Estadísticas Btrfs

**Resultado: SIN ERRORES REGISTRADOS.**

Todos los contadores de `btrfs device stats` están en cero:

- write_io_errs: 0
- read_io_errs: 0
- flush_io_errs: 0
- corruption_errs: 0
- generation_errs: 0

Esto reduce considerablemente la probabilidad de un error de I/O Btrfs/NVMe persistente, aunque no lo descarta completamente.

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

La diferencia de aproximadamente cuatro horas entre timestamps del kernel y systemd apunta fuertemente a un **salto/ajuste del reloj del sistema durante el arranque** (por ejemplo, sincronización de tiempo). La hipótesis de que `systemd-remount-fs` estuviera bloqueado cuatro horas queda **descartada**.

`systemd-analyze blame` sí identifica como demora relevante `home-oscar-scans.mount` (~42 s), además de `plymouth-quit-wait.service` (~21 s) y `NetworkManager-wait-online.service` (~9 s), pero ninguna explica un retraso de horas.

## Hipótesis actuales

| ID | Hipótesis | Prioridad | Estado |
|---|---|---:|---|
| H1 | El remount `ro → rw` falla durante determinados boots | Alta | En investigación |
| H2 | Btrfs detecta una condición y remonta `ro` | Alta | Sin evidencia hasta ahora |
| H3 | Otro componente vuelve a montar `/` como `ro` | Media | En investigación |
| H4 | SSHFS persistente de heinrici bloquea Nautilus | Alta | Evidencia fuerte |
| H5 | Problema físico/controlador del NVMe no reflejado en Btrfs stats/SMART | Media-baja | No demostrado |
| H6 | Journald no está haciendo correctamente el flush persistente durante determinados arranques | Alta | Evidencia parcial |
| H7 | Salto de reloj durante el arranque explica la diferencia 14:59→18:59 observada en timestamps | Alta | Evidencia fuerte; pendiente de confirmar con servicio de sincronización de tiempo |
| H8 | `home-oscar-scans.mount` contribuye a demoras/bloqueos durante el arranque | Media | Evidencia: ~42 s en `systemd-analyze blame` |

## Próxima prueba

Investigar la sincronización/ajuste del reloj durante el arranque para confirmar H7 y, simultáneamente, revisar por qué `systemd-journal-flush` no está dejando el journal persistente automáticamente.

Después se retomará el diagnóstico de la causa real del estado `ro` en los boots problemáticos.

## Pendientes posteriores

- Migrar `/home/oscar/heinrici` desde `~/.config/autostart/heinrici.desktop` a systemd user + automount, evitando bloqueos de Nautilus cuando `heinrici` está apagado.
- Revisar `loginctl`/`systemd --user` y configurar linger correctamente para servicios persistentes.
- Investigar el error histórico de sincronización de caché de `/dev/sda` (KIOXIA `KBG40ZNV512G`) visto en el boot de abril, separándolo del NVMe raíz.

## Regla de trabajo

No se harán cambios permanentes basados únicamente en síntomas. Cada modificación debe tener:

1. hipótesis que pretende corregir;
2. cambio concreto;
3. prueba posterior;
4. resultado registrado;
5. posibilidad de revertirlo.
