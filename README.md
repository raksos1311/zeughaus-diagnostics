# zeughaus-diagnostics

Diagnóstico reproducible de problemas de Fedora/Btrfs en **zeughaus**, con especial atención a montajes de `/` en read-only, persistencia de journald, servicios que no arrancan después de un reinicio y montajes SSHFS hacia `heinrici`.

## Estado actual — 2026-08-14

### Hallazgos confirmados

- La raíz y `/home` están actualmente montadas en Btrfs `rw` sobre `/dev/nvme0n1p5`.
- El kernel command line contiene `root=... ro`; esto es normal durante el arranque y **no demuestra por sí mismo** que Btrfs permanezca en read-only.
- `systemd-remount-fs.service` termina con `status=0/SUCCESS`.
- Los contadores Btrfs del NVMe están en cero: `write_io_errs=0`, `read_io_errs=0`, `flush_io_errs=0`, `corruption_errs=0`, `generation_errs=0`.
- En los dos arranques recientes revisados no aparecen errores Btrfs/NVMe, I/O errors, corrupción, aborts o resets que expliquen por sí solos el cambio a RO.
- Los logs persistentes de journald sí están funcionando y `/var/log/journal` está en la raíz Btrfs.
- El problema de pérdida de contexto entre reinicios sigue abierto: `journalctl --list-boots` sólo conserva los arranques de abril y el arranque actual; por tanto, varios reinicios posteriores no quedaron disponibles como boots persistentes.

### Hipótesis actual

Todavía **no sabemos qué causa el estado RO**. No debemos asumir que es un SSD defectuoso ni que es simplemente un fallo de `systemd-remount-fs`.

La prueba decisiva será capturar el estado **en el instante exacto** en que una escritura contra `/` devuelva `EROFS`.

## Watchdog `zeughaus-ro-watch`

El watchdog escribe una pequeña prueba cada 10 s contra `/root`. Si la operación falla, obtiene el `errno` directamente mediante Python y almacena evidencia en `/run`, que continúa siendo escribible aunque `/` esté montado RO.

La versión actual es **v0.4**.

### Estado de instalación

El watchdog está actualmente **instalado, habilitado y ejecutándose** en `zeughaus`.

- Unidad activa: `/etc/systemd/system/zeughaus-ro-watch.service`
- Ejecutable: `/usr/local/sbin/zeughaus-ro-watch`
- Evidencia temporal: `/run/zeughaus-ro-watch/`
- Servicio: `systemctl status zeughaus-ro-watch.service`
- Logs del servicio: `sudo journalctl -u zeughaus-ro-watch.service -b --no-pager`

La instalación y activación fueron verificadas el 2026-08-14.

### Características deliberadas

- `Restart=always` para recuperar el watchdog si el proceso termina.
- ejecución como `root`, necesaria para probar escritura en `/root` independientemente de los permisos del usuario.
- estado forense bajo `/run`, que es `tmpfs` y no depende de que `/` siga siendo escribible.
- sin `ProtectSystem`, porque el objetivo es detectar precisamente cambios de estado de `/`.
- **sin remount automático**: durante la investigación queremos preservar el estado original del incidente.

## Procedimiento cuando `/` pase a RO

Ésta es la parte crítica. El watchdog está diseñado para que el usuario **no tenga que reparar el sistema antes de capturar la evidencia**.

### Si ocurre el incidente y la interfaz/TTY sigue respondiendo

**NO ejecutar `mount -o remount,rw /`.**

**NO ejecutar `btrfs check`, scrub ni reparaciones.**

**NO reiniciar inmediatamente.**

Primero dejar que `zeughaus-ro-watch` detecte la pérdida de escritura y capture el incidente en `/run`.

Comprobar únicamente:

```bash
sudo find /run/zeughaus-ro-watch -maxdepth 2 -type f -printf '%p\n'
```

y:

```bash
sudo cat /run/zeughaus-ro-watch/LAST_INCIDENT
```

Debe aparecer una carpeta del tipo:

```text
/run/zeughaus-ro-watch/ro-incident-YYYYMMDD-HHMMSS-NNNNNNNNN
```

**No modificar ni borrar esa carpeta.**

Después de confirmar que existe el incidente, avisar para realizar la cosecha de evidencia al repositorio. No copiar `/run` completo al repositorio: se seleccionarán los artefactos forenses relevantes y se registrará el incidente de forma reproducible.

### Si la máquina queda completamente congelada

No hay ninguna acción útil que hacer antes del reinicio forzado/REISUB. En ese caso puede perderse parte de la evidencia, pero el objetivo del watchdog es capturarla antes del bloqueo si el kernel y el proceso siguen ejecutándose.

### Si el filesystem vuelve a RW por sí solo

No intervenir. El watchdog registra la recuperación en `/run/zeughaus-ro-watch/WRITE_FAILURE_RECOVERY`. La duración del intervalo no escribible también queda registrada en `WRITE_FAILURE_ACTIVE` mientras el incidente está activo.

### Principio de investigación

La prioridad durante un incidente es:

```text
preservar estado → capturar evidencia → analizar → recuperar/reiniciar
```

No:

```text
reparar → reiniciar → intentar reconstruir qué ocurrió
```

La segunda estrategia fue precisamente la que nos dejó sin el boot problemático en el journal histórico.

## Corrección v0.4

En v0.3, el forense intentaba ejecutar la función Bash `write_test` directamente mediante `timeout`. `timeout` sólo puede ejecutar un programa externo, por lo que esa segunda prueba habría fallado aunque el watchdog principal hubiese detectado correctamente `EROFS`.

v0.4 corrige la ejecución de la prueba forense y conserva el `errno` real de la operación de escritura.

Commit de corrección: `2bec0d904207f2f6c5a6b747658ac83b63cc1883`.

La sintaxis local fue validada con:

```bash
bash -n tools/zeughaus-ro-watch
```

## Desinstalación posterior obligatoria

Este watchdog es una herramienta **temporal de diagnóstico**, no una configuración permanente del sistema.

Al finalizar la investigación:

1. detener el servicio;
2. deshabilitarlo;
3. eliminar `/etc/systemd/system/zeughaus-ro-watch.service`;
4. eliminar `/usr/local/sbin/zeughaus-ro-watch`;
5. ejecutar `systemctl daemon-reload`;
6. conservar en el repositorio el registro de instalación, funcionamiento y eliminación.

La bitácora debe reflejar la configuración real utilizada, no una versión idealizada del sistema.

## Regla de investigación

Cada prueba debe quedar registrada con:

- hipótesis;
- comando/prueba ejecutada;
- resultado;
- interpretación;
- decisión siguiente.

No se considera una solución simplemente "mirar el journal". El objetivo es llegar a una causa reproducible o, al menos, descartar sistemáticamente las causas principales.

## Próximas líneas de trabajo

1. Esperar el siguiente incidente RO y capturar evidencia con el watchdog.
2. Analizar el incidente antes de cualquier reparación.
3. Correlacionar el instante del `EROFS` con kernel, Btrfs, NVMe, systemd y estado de montaje.
4. Continuar investigando la persistencia de journald.
5. Posteriormente migrar `/home/oscar/heinrici` desde `~/.config/autostart/heinrici.desktop` a systemd user + automount.
6. Revisar `systemd --user` + linger para servicios persistentes.

No ejecutar todavía `btrfs check`, scrub ni reparaciones a ciegas.
