OSCAR: pero eso es lo que no me entiendes.... Cuando monto mi disco nvme0n1p6, que es donde está mi home (se llama fedora la particion), queda /home/home/oscar Eso es lo que no me has entendido. Me dijiste que "monto home en otro home", pero eso es lo que queda al montar el disco. Eso es todo..


Ahora sí. Por fin está claro. Y tienes razón: no era “otro home”, era la misma partición montada en el punto equivocado.

👉 Lo que describes:

Montas nvme0n1p6 en /home
Y aparece:
/home/home/oscar

💥 Eso significa:

Esa partición tiene como raíz una carpeta /home adentro

🧠 Traducción simple

Tu disco está así internamente:

/ (raíz de la partición)
└── home
    └── oscar
        └── programacion

Y tú lo estás montando en:

/home

Entonces queda:

/home/home/oscar
🎯 SOLUCIÓN CORRECTA

👉 NO debes montar la partición en /home
👉 Debes montar la carpeta interna /home de esa partición como /home

🔧 OPCIÓN RÁPIDA (AHORA MISMO)

Haz esto:

sudo umount /home
sudo mount /dev/nvme0n1p6 /mnt
sudo mount --bind /mnt/home /home
🔍 Verifica:
ls /home

👉 Ahora deberías ver directamente:

oscar

💥 Sin doble /home

🔁 SI ESO FUNCIONA → hacerlo permanente

Edita fstab:

sudo nano /etc/fstab
🔥 Agrega ESTA línea:
/dev/nvme0n1p6 /mnt btrfs defaults 0 0
/mnt/home /home none bind 0 0

👉 Y asegúrate de:

NO tener otra línea /home
🔁 Reinicia:
sudo reboot
💬 Te lo digo directo

No era:

error de usuario ❌
error de sistema ❌

👉 Era un clásico:

💥 “montaste la raíz de la partición, pero tu home está un nivel más abajo”

🧠 Alternativa más limpia (después)

Podrías montar directo con subvolumen si usas Btrfs, pero por ahora:

👉 el bind mount es simple y seguro

🚀 Haz esto ahora

Los 3 comandos:

umount /home
mount /dev/nvme0n1p6 /mnt
mount --bind /mnt/home /home

Si ves tu carpeta programacion después de eso:

👉 listo, recuperaste TODO

Dime si aparece bien. Esto ya es el último paso real. 💪

OSCAR:
Ya funciona....

Ahora dime qué hacer para que no vuelva a pasar...
