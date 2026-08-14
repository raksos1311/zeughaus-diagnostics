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

## Hipótesis actuales

| ID | Hipótesis | Prioridad | Estado |
|---|---|---:|---|
| H1 | El remount `ro → rw` falla durante determinados boots | Alta | En investigación |
| H2 | Btrfs detecta una condición y remonta `ro` | Alta | Sin evidencia hasta ahora |
| H3 | Otro componente vuelve a montar `/` como `ro` | Media | En investigación |
| H4 | SSHFS persistente de heinrici bloquea Nautilus | Alta | Evidencia fuerte |
| H5 | Problema físico/controlador del NVMe no reflejado en Btrfs stats/SMART | Media-baja | No demostrado |

## Próxima prueba

Analizar el **boot anterior**, que corresponde al incidente real, en lugar del boot actual que funcionó correctamente.

Se buscarán específicamente:

- `systemd-remount-fs`
- `read-only` / `readonly`
- `remount`
- `BTRFS error`
- `BTRFS critical`
- `I/O error`
- `nvme timeout`
- `abort`
- `corrupt`
- estado de `libvirtd`

## Regla de trabajo

No se harán cambios permanentes basados únicamente en síntomas. Cada modificación debe tener:

1. hipótesis que pretende corregir;
2. cambio concreto;
3. prueba posterior;
4. resultado registrado;
5. posibilidad de revertirlo.
