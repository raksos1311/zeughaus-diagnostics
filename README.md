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

### Corrección v0.4

En v0.3, el forense intentaba ejecutar la función Bash `write_test` directamente mediante `timeout`. `timeout` sólo puede ejecutar un programa externo, por lo que esa segunda prueba habría fallado aunque el watchdog principal hubiese detectado correctamente `EROFS`.

v0.4 exporta `write_test` y lo ejecuta mediante `timeout -> bash`, conservando el límite temporal.

Commit: `2bec0d904207f2f6c5a6b747658ac83b63cc1883`

## Próximo procedimiento

1. Hacer `git pull` y revisar v0.4 localmente.
2. Validar sintaxis y ejecutar una prueba controlada del watchdog.
3. Instalarlo como servicio systemd.
4. No automatizar todavía ningún `remount,rw`: durante la investigación queremos preservar el estado original del incidente.
5. Cuando vuelva a ocurrir el RO, conservar primero el incidente generado en `/run`.
6. Analizar `INCIDENT_TRIGGER.txt`, `write-error-retest.txt`, estado Btrfs, estado de montaje y mensajes kernel capturados.
7. Sólo después decidir si corresponde corregir Btrfs, systemd, kernel/firmware, configuración de montaje o hardware.

## SSHFS hacia heinrici

Hay además un problema independiente: Nautilus puede quedar bloqueado cuando intenta acceder a `/home/oscar/heinrici` y `heinrici` está apagado. El montaje actual usa SSHFS con `reconnect`, pero la existencia de un montaje FUSE persistente puede provocar esperas al acceder a la ruta.

Esto se tratará como una segunda línea de trabajo: primero estabilizar la captura del incidente RO y luego rediseñar el montaje SSHFS para que la indisponibilidad de `heinrici` no bloquee la sesión gráfica.

## Regla de investigación

Cada prueba debe quedar registrada con:

- hipótesis;
- comando/prueba ejecutada;
- resultado;
- interpretación;
- decisión siguiente.

No se considera una solución simplemente "mirar el journal". El objetivo es llegar a una causa reproducible o, al menos, descartar sistemáticamente las causas principales.
