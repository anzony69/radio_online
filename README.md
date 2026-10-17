# Radio con Liquidsoap y Listen2MyRadio

<img width="1123" height="1400" alt="Panel de Control de Radio En Línea" src="https://github.com/user-attachments/assets/4448435d-40c3-44bf-ad53-1022e00ae81a" />


Servidor de radio online autoalojado en un contenedor LXC de Proxmox. Liquidsoap reproduce archivos MP3 locales y envía el audio a un servidor de streaming Listen2MyRadio mediante Shoutcast v2.

## Características

- Emisión continua 24/7 desde un contenedor LXC, sin tarjeta de sonido.
- Reproducción de archivos MP3 almacenados localmente.
- Playlists generadas mediante `find` a partir de las carpetas elegidas.
- Envío del audio codificado en MP3 a Listen2MyRadio mediante Shoutcast v2.
- Servicio gestionado por `systemd`, con reinicio automático.

## Arquitectura

- **Hipervisor:** Proxmox VE.
- **Contenedor:** LXC con Ubuntu Server 22.04.
- **Motor de emisión:** Liquidsoap.
- **Biblioteca musical:** `/mnt/MUSICA`.
- **Archivos y configuración de radio:** `/opt/radio`.
- **Destino de streaming:** Listen2MyRadio, protocolo Shoutcast v2.

> Esta guía refleja la instalación descrita en las notas del proyecto. Adapta rutas, usuario, host y puerto a tu entorno.

## 1. Preparar el contenedor LXC y montar la música

Crea un contenedor LXC con Ubuntu Server 22.04 en Proxmox. Monta o enlaza el almacenamiento musical dentro del contenedor en:

```text
/mnt/MUSICA
```

Comprueba que el contenido está disponible:

```bash
ls /mnt/MUSICA
find /mnt/MUSICA -type f -iname '*.mp3' | head
```

Si no aparecen archivos, revisa el montaje del almacenamiento y los permisos de lectura del usuario que ejecutará Liquidsoap.

## 2. Instalar Liquidsoap

Actualiza los índices e instala Liquidsoap desde los repositorios de Ubuntu:

```bash
sudo apt update
sudo apt install liquidsoap
```

Liquidsoap puede leer los archivos directamente, codificar el audio y enviarlo al servidor de streaming sin necesitar una tarjeta de sonido física.

Comprueba la versión instalada:

```bash
liquidsoap --version
```

## 3. Crear la estructura de directorios

Crea el directorio de trabajo:

```bash
sudo mkdir -p /opt/radio
sudo chown anzony:anzony /opt/radio
```

En este ejemplo, el usuario de servicio es `anzony`. Si tu contenedor utiliza otro usuario, sustitúyelo en los comandos y en el servicio systemd.

## 4. Generar las playlists con `find`

Las playlists M3U contienen rutas a los archivos que Liquidsoap reproducirá. Puedes generar una playlist general o seleccionar solo carpetas concretas.

### 4.1. Playlist con toda la música (opcional)

```bash
find /mnt/MUSICA -type f -iname '*.mp3' > /opt/radio/playlist.m3u
```

### 4.2. Playlist `Temas` con carpetas seleccionadas

El siguiente ejemplo reproduce música de las carpetas indicadas en la instalación original. Si tu estructura de directorios es diferente, modifica las rutas.

```bash
# Crear o vaciar la playlist
: > /opt/radio/Temas.m3u

find "/mnt/MUSICA/Temas/House/Lo Mejor" -type f -iname '*.mp3' >> /opt/radio/Temas.m3u
find "/mnt/MUSICA/Temas/Remember" -type f -iname '*.mp3' >> /opt/radio/Temas.m3u
find "/mnt/MUSICA/Temas/Techno/Lo Mejor" -type f -iname '*.mp3' >> /opt/radio/Temas.m3u
find "/mnt/MUSICA/Temas/Techno/Hardtechno/Lo Mejor" -type f -iname '*.mp3' >> /opt/radio/Temas.m3u

find "/mnt/MUSICA/Temas/Trance/AreA" -type f -iname '*.mp3' >> /opt/radio/Temas.m3u
find "/mnt/MUSICA/Temas/Trance/DSi" -type f -iname '*.mp3' >> /opt/radio/Temas.m3u
find "/mnt/MUSICA/Temas/Trance/Lo Mejor/Hasta 2005" -type f -iname '*.mp3' >> /opt/radio/Temas.m3u
find "/mnt/MUSICA/Temas/Trance/Antiguo/Lo Mejor" -type f -iname '*.mp3' >> /opt/radio/Temas.m3u
find "/mnt/MUSICA/Temas/Trance/Antiguo/Epoca AreA/Varios" -type f -iname '*.mp3' >> /opt/radio/Temas.m3u
find "/mnt/MUSICA/Temas/Trance/2009-2010" -type f -iname '*.mp3' >> /opt/radio/Temas.m3u
find "/mnt/MUSICA/Temas/Trance/2008" -type f -iname '*.mp3' >> /opt/radio/Temas.m3u
find "/mnt/MUSICA/Temas/Trance/2006-2007" -type f -iname '*.mp3' >> /opt/radio/Temas.m3u
```

Comprueba el resultado:

```bash
wc -l /opt/radio/Temas.m3u
head /opt/radio/Temas.m3u
```

Puedes crear otras playlists por género, época o cualquier otro criterio repitiendo el patrón y guardando los resultados en archivos distintos, por ejemplo `Trance.m3u`.

> Si una ruta no existe, `find` mostrará un aviso y no añadirá canciones de esa carpeta. Comprueba que las rutas coinciden con los nombres reales de los directorios.

## 5. Configurar Listen2MyRadio

En el panel de administración de Listen2MyRadio, configura el tipo de fuente como **Shoutcast v2**. En la instalación original, el cambio de Shoutcast v1 a v2 permitió que la fuente conectara correctamente.

Anota estos datos que proporciona el servicio:

- **Hostname:** servidor de streaming asignado a tu cuenta.
- **Puerto:** puerto de emisión, no el puerto del panel de administración.
- **Transmitir contraseña / source password:** contraseña que utiliza Liquidsoap para autenticar la fuente.

No publiques la contraseña de emisión ni la guardes en un repositorio público. En los ejemplos siguientes se utiliza un marcador de posición.

## 6. Configurar Liquidsoap

Crea el archivo:

```text
/opt/radio/radio.liq
```

Contenido de ejemplo:

```liquidsoap
set("init.allow_root", true)
set("log.file.path", "/opt/radio/radio.log")

# Playlist que se va a emitir
radio = playlist("/opt/radio/Temas.m3u")
radio = mksafe(radio)

output.shoutcast(
  %mp3(bitrate=128),
  host = "TU_HOSTNAME_LISTEN2MYRADIO",
  port = TU_PUERTO,
  password = "TU_SOURCE_PASSWORD",
  radio
)
```

Sustituye `TU_HOSTNAME_LISTEN2MYRADIO`, `TU_PUERTO` y `TU_SOURCE_PASSWORD` por los valores reales del panel de Listen2MyRadio. No uses literalmente los marcadores de posición.

### Qué hace la configuración

- `playlist("/opt/radio/Temas.m3u")`: carga las rutas de los MP3 desde la playlist.
- `mksafe(radio)`: convierte la fuente en una fuente segura para la salida, evitando que un fallo de disponibilidad de la fuente detenga inmediatamente la cadena de emisión.
- `%mp3(bitrate=128)`: codifica la emisión a MP3 con un bitrate de 128 kb/s.
- `output.shoutcast(...)`: conecta Liquidsoap al servidor Shoutcast configurado.

### Probar la emisión manualmente

Ejecuta:

```bash
liquidsoap -v /opt/radio/radio.liq
```

Comprueba que Liquidsoap carga la playlist y que logra establecer la conexión con Listen2MyRadio. En las notas de la instalación, la conexión correcta se identificaba con un mensaje similar a `Connection setup was successful`.

Detén la prueba manual antes de iniciar el servicio systemd para evitar dos procesos intentando emitir a la vez.

## 7. Crear el servicio systemd

Crea el archivo:

```text
/etc/systemd/system/radio.service
```

Contenido:

```ini
[Unit]
Description=Radio Anzony Liquidsoap
After=network-online.target
Wants=network-online.target

[Service]
User=anzony
Group=anzony
ExecStart=/usr/bin/liquidsoap /opt/radio/radio.liq
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Activa el servicio y arráncalo:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now radio.service
```

Comprueba el estado:

```bash
sudo systemctl status radio.service
```

Consulta los registros en tiempo real:

```bash
sudo journalctl -u radio.service -f
```

El estado esperado es `active (running)`. En los registros deben aparecer la preparación de los temas y los mensajes de conexión al servidor de streaming.

## 8. Uso diario

### Actualizar la playlist `Temas`

Para regenerar la playlist, vacía el archivo y vuelve a ejecutar los comandos `find` del apartado 4.2:

```bash
: > /opt/radio/Temas.m3u
```

A continuación, repite las órdenes `find` de las carpetas que quieras incluir.

### Cambiar a otra playlist

Por ejemplo, para emitir desde `/opt/radio/Trance.m3u`, cambia la línea del archivo `radio.liq`:

```liquidsoap
radio = playlist("/opt/radio/Trance.m3u")
```

Asegúrate de que la playlist existe y contiene rutas válidas. Reinicia el servicio para aplicar el cambio:

```bash
sudo systemctl restart radio.service
```

### Comandos de mantenimiento

```bash
# Estado del servicio
sudo systemctl status radio.service

# Registros del servicio
sudo journalctl -u radio.service -f

# Últimos registros
sudo journalctl -u radio.service -n 100 --no-pager

# Reiniciar la emisión
sudo systemctl restart radio.service

# Validar el contenido de la playlist
wc -l /opt/radio/Temas.m3u
head /opt/radio/Temas.m3u
```

## 9. Problemas habituales

- **No aparecen canciones:** verifica el montaje en `/mnt/MUSICA`, las rutas de los `find`, el contenido de la playlist y los permisos de lectura.
- **Falla la conexión al servidor:** revisa hostname, puerto de emisión, contraseña de fuente y que Listen2MyRadio esté configurado en Shoutcast v2.
- **El servicio arranca y se detiene:** consulta `journalctl -u radio.service -n 100 --no-pager` y prueba el script manualmente.
- **La playlist se queda vacía:** comprueba que las rutas existan y que los archivos tengan extensión `.mp3` (la búsqueda con `-iname` no distingue mayúsculas y minúsculas).
- **Se ejecutan dos emisores a la vez:** detén cualquier ejecución manual de Liquidsoap antes de iniciar el servicio.

## 10. Resultado

La configuración permite emitir música local de forma continua desde un LXC sin tarjeta de sonido, controlar las canciones mediante playlists M3U generadas con `find` y enviar el flujo MP3 a Listen2MyRadio a través de Shoutcast v2.

## Referencias

- [Liquidsoap: documentación oficial](https://www.liquidsoap.info/)
- [Liquidsoap: formatos de codificación](https://www.liquidsoap.info/doc-dev/encoding_formats.html)
- [Liquidsoap: parsers de playlists](https://www.liquidsoap.info/doc-dev/playlist_parsers.html)
- [Liquidsoap: salida Shoutcast](https://www.liquidsoap.info/doc-dev/shoutcast.html)
- [Listen2MyRadio](https://www.listen2myradio.com/)
