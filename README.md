# Manual de operación — adquisición DM24 DIAM

## 1. Objetivo y arquitectura

Este servicio reemplaza la adquisición realizada por el Güralp NAM para la estación DM24 DIAM.

Flujo de datos:

```text
DM24 172.16.18.87:1568
        │ GCF/MSS sobre TCP
        ▼
gcf2ew (contenedor Earthworm de 32 bits)
        │ TYPE_TRACEBUF2 — DIAM_RING
        ▼
ew2ringserver
        │ miniSEED por DataLink 127.0.0.1:16000
        ▼
ringserver del servidor 172.16.10.2
        │ SeedLink 172.16.10.2:18000
        ▼
clientes SeedLink
```

Canales publicados:

```text
CM.DIAM.00.HHE
CM.DIAM.00.HHN
CM.DIAM.00.HHZ
```

## 2. Datos importantes

| Elemento | Valor |
|---|---|
| Servidor | `172.16.10.2` (`tororoi`) |
| Estación DM24 | `172.16.18.87` |
| Puerto GCF/MSS | `1568/TCP` |
| Imagen Docker | `earthworm-diam:gcf-test` |
| Contenedor | `earthworm-diam` |
| Directorio del proyecto | `/home/julian/diam-earthworm-docker` |
| Logs persistentes | `/home/julian/earthworm-diam-container-logs` |
| Ring Earthworm | `DIAM_RING`, clave `3100`, 16 MB |
| DataLink ringserver | `127.0.0.1:16000` |
| SeedLink ringserver | `172.16.10.2:18000` |
| Red/estación/localización | `CM.DIAM.00` |

La imagen es de arquitectura `linux/386` porque la biblioteca `libgcf.a` suministrada con `gcf2ew` es de 32 bits.

## 3. Revisión rápida diaria

### 3.1 Estado del contenedor

```bash
sudo docker ps --filter name=earthworm-diam
```

Debe mostrar el contenedor `earthworm-diam` con estado `Up`.

Consulta detallada:

```bash
sudo docker inspect earthworm-diam \
  --format 'estado={{.State.Status}} reinicios={{.RestartCount}} inicio={{.State.StartedAt}}'
```

Resultado esperado:

```text
estado=running reinicios=0
```

### 3.2 Procesos internos

```bash
sudo docker top earthworm-diam
```

Deben aparecer estos tres procesos:

```text
startstop
gcf2ew gcf2ew.d
ew2ringserver ew2ringserver.d
```

También se puede consultar el estado de Earthworm:

```bash
sudo docker exec earthworm-diam bash -lc \
  'source /opt/earthworm/earthworm_8.0/environment/ew_linux.bash; status'
```

`gcf2ew` y `ew2ringserver` deben aparecer como `Alive`. Un proceso `Zombie` requiere intervención.

### 3.3 Conexiones de red

```bash
sudo ss -tnp | grep -E '172.16.18.87:1568|127.0.0.1:16000'
```

Deben existir dos conexiones `ESTAB`:

- `gcf2ew` hacia `172.16.18.87:1568`.
- `ew2ringserver` hacia `127.0.0.1:16000`.

Prueba básica del puerto del DM24:

```bash
nc -vz -w 5 172.16.18.87 1568
```

### 3.4 Logs recientes

```bash
sudo docker logs --since 30m earthworm-diam
```

Buscar errores importantes:

```bash
sudo docker logs --since 30m earthworm-diam 2>&1 | \
  grep -Ei 'fatal|zombie|exiting|failure|disconnect|missed|lost'
```

Los mensajes siguientes no son necesariamente fallas:

```text
Queue is empty, waiting for producer
Interactive() thread exiting
```

El primero indica que durante 30 segundos no entraron nuevos paquetes. El segundo es normal al ejecutar `startstop` sin terminal interactiva.

Sí requieren atención:

```text
Zombie
Socket Connection Failure
Exiting because ...
missed N packets
Total blocks missed N
```

## 4. Arrancar, parar y reiniciar

### Parar el servicio

```bash
sudo docker stop -t 30 earthworm-diam
```

Esto detiene el contenedor sin borrarlo.

### Arrancar un contenedor detenido

```bash
sudo docker start earthworm-diam
```

### Reiniciar el servicio

```bash
sudo docker restart -t 30 earthworm-diam
```

Después del reinicio:

```bash
sudo docker ps --filter name=earthworm-diam
sudo docker logs --since 5m earthworm-diam
sudo docker top earthworm-diam
```

### Eliminar el contenedor

Eliminar el contenedor no elimina la imagen ni los logs persistentes montados desde el host.

```bash
sudo docker stop -t 30 earthworm-diam
sudo docker rm earthworm-diam
```

### Crear nuevamente el contenedor

```bash
sudo docker run -d \
  --init \
  --platform linux/386 \
  --name earthworm-diam \
  --restart unless-stopped \
  --network host \
  -v /home/julian/earthworm-diam-container-logs:/opt/earthworm/run_diam/log \
  earthworm-diam:gcf-test
```

No deben ejecutarse simultáneamente dos contenedores que adquieran la misma estación.

## 5. Verificación de datos en Earthworm

### Observar paquetes TRACEBUF2 durante 90 segundos

```bash
sudo docker exec earthworm-diam bash -lc \
  'source /opt/earthworm/earthworm_8.0/environment/ew_linux.bash; timeout 90 sniffring DIAM_RING INST_WILDCARD MOD_GCF2EW_DIAM TYPE_TRACEBUF2'
```

### Ver encabezados de los canales

```bash
sudo docker exec earthworm-diam bash -lc \
  'source /opt/earthworm/earthworm_8.0/environment/ew_linux.bash; timeout 90 sniffwave DIAM_RING wild wild CM wild n noflush verbose'
```

Se esperan:

```text
DIAM.HHE.CM.00
DIAM.HHN.CM.00
DIAM.HHZ.CM.00
```

Parámetros esperados: 100 muestras por segundo, paquetes de 1000 muestras y datos enteros de 32 bits.

## 6. Verificación en SeedLink

La opción `-L` lista estaciones y `-Q` lista canales y periodos disponibles.

Como Debian Bookworm no incluye un paquete `slinktool`, se compila temporalmente dentro de un contenedor; esto no instala software en el host.

```bash
sudo docker run --rm --network host debian:bookworm-slim bash -lc 'apt-get update >/dev/null && apt-get install -y --no-install-recommends build-essential ca-certificates curl >/dev/null && curl -fsSL https://github.com/EarthScope/slinktool/archive/refs/tags/v4.5.0.tar.gz | tar -xz && cd slinktool-4.5.0 && make >/dev/null && ./slinktool -Q 127.0.0.1:18000' 2>&1 | grep -i DIAM
```

Resultado esperado:

```text
CM DIAM  00 HHE D fecha_inicial - fecha_final
CM DIAM  00 HHN D fecha_inicial - fecha_final
CM DIAM  00 HHZ D fecha_inicial - fecha_final
```

Ejecute la consulta dos veces, separadas por varios minutos. Las fechas finales deben avanzar. Los antiguos streams `OV DIAM` pueden permanecer temporalmente en el búfer, pero no deben seguir actualizándose.

## 7. Archivos de configuración

Archivos del proyecto:

```text
/home/julian/diam-earthworm-docker/Dockerfile
/home/julian/diam-earthworm-docker/entrypoint.sh
/home/julian/diam-earthworm-docker/params/gcf2ew.d
/home/julian/diam-earthworm-docker/params/ew2ringserver.d
/home/julian/diam-earthworm-docker/params/startstop_unix.d
```

Revisar el mapeo y los filtros:

```bash
cd /home/julian/diam-earthworm-docker
grep -nE 'DIAM|HHE|HHN|HHZ|HostAddress|PortNumber' \
  params/gcf2ew.d params/ew2ringserver.d
```

Después de modificar archivos de `params/`, es necesario reconstruir la imagen y recrear el contenedor:

```bash
cd /home/julian/diam-earthworm-docker
sudo docker build --platform linux/386 -t earthworm-diam:gcf-test .
sudo docker stop -t 30 earthworm-diam
sudo docker rm earthworm-diam
sudo docker run -d --init --platform linux/386 \
  --name earthworm-diam --restart unless-stopped --network host \
  -v /home/julian/earthworm-diam-container-logs:/opt/earthworm/run_diam/log \
  earthworm-diam:gcf-test
```

Antes de reconstruir, se recomienda conservar una copia del directorio del proyecto.

## 8. Logs persistentes

Los logs se conservan en el host aunque se elimine el contenedor:

```bash
ls -lh /home/julian/earthworm-diam-container-logs
```

Logs de adquisición:

```bash
tail -100 /home/julian/earthworm-diam-container-logs/gcf2ew*.log
```

Logs de exportación hacia ringserver:

```bash
tail -100 /home/julian/earthworm-diam-container-logs/ew2ringserver*.log
```

Buscar pérdida de bloques:

```bash
grep -E 'missed|lost|Total Blocks' \
  /home/julian/earthworm-diam-container-logs/gcf2ew*.log | tail -50
```

La pérdida persistente de bloques debe investigarse. Puede originarse en la red, en el servicio MSS del DM24, en entrega por ráfagas o en competencia con otro cliente de adquisición.

## 9. Diagnóstico de fallas

### El contenedor no está ejecutándose

```bash
sudo docker ps -a --filter name=earthworm-diam
sudo docker inspect earthworm-diam \
  --format 'estado={{.State.Status}} salida={{.State.ExitCode}} error={{.State.Error}}'
sudo docker logs --tail 200 earthworm-diam
```

Intentar arrancarlo:

```bash
sudo docker start earthworm-diam
```

### gcf2ew no conecta con el DM24

```bash
ip route get 172.16.18.87
ping -c 4 172.16.18.87
nc -vz -w 5 172.16.18.87 1568
sudo ss -tnp | grep '172.16.18.87:1568'
```

Revisar también si el NAM u otro cliente mantiene una sesión que interfiera con el servicio MSS.

### ew2ringserver no conecta con ringserver

```bash
nc -vz -w 5 127.0.0.1 16000
sudo ss -ltnp | grep ':16000'
sudo ss -tnp | grep '127.0.0.1:16000'
sudo tail -100 /tmp/ringserver.out
```

### El contenedor está activo pero gcf2ew está Zombie

```bash
sudo docker restart -t 30 earthworm-diam
```

Después, confirme procesos, conexiones y datos. La política de reinicio de Docker vigila el proceso principal del contenedor; por ello se recomienda implementar posteriormente un `HEALTHCHECK` que detecte fallas de procesos secundarios.

### Los streams aparecen, pero las fechas no avanzan

1. Confirmar conexión con el DM24.
2. Ejecutar `sniffring` durante cinco minutos.
3. Revisar `gcf2ew*.log` en busca de pérdidas o desconexiones.
4. Confirmar que `ew2ringserver` continúa conectado a DataLink.
5. Consultar de nuevo SeedLink con `slinktool -Q`.

## 10. Reinicio del servidor

El contenedor tiene:

```text
--restart unless-stopped
```

Por tanto, Docker debe arrancarlo automáticamente después de reiniciar el servidor, siempre que el servicio Docker esté habilitado:

```bash
sudo systemctl is-enabled docker
sudo systemctl is-active docker
```

Después de reiniciar el servidor:

```bash
sudo docker ps --filter name=earthworm-diam
sudo docker top earthworm-diam
sudo docker logs --since 10m earthworm-diam
```

## 11. Comando de diagnóstico completo

```bash
echo '=== CONTENEDOR ==='
sudo docker inspect earthworm-diam --format 'estado={{.State.Status}} reinicios={{.RestartCount}} inicio={{.State.StartedAt}}'
echo '=== PROCESOS ==='
sudo docker top earthworm-diam
echo '=== CONEXIONES ==='
sudo ss -tnp | grep -E '172.16.18.87:1568|127.0.0.1:16000' || true
echo '=== LOGS RECIENTES ==='
sudo docker logs --since 10m earthworm-diam 2>&1 | tail -100
echo '=== PERDIDAS GCF ==='
grep -E 'missed|lost|Total Blocks' /home/julian/earthworm-diam-container-logs/gcf2ew*.log | tail -20 || true
```

## 12. Recomendaciones para producción

- Observar DIAM por lo menos 12 a 24 horas antes de agregar estaciones.
- Registrar bloques recibidos y bloques perdidos.
- Agregar un `HEALTHCHECK` que compruebe procesos y frescura de datos.
- Configurar rotación de logs para evitar crecimiento ilimitado.
- Crear una etiqueta estable para la imagen, por ejemplo `earthworm-diam:1.0`, en lugar de `gcf-test`.
- Respaldar Dockerfile, configuraciones y documentación.
- No ejecutar dos adquisidores contra el mismo servicio DM24 sin comprobar que MSS admite múltiples clientes.
- Verificar la licencia de la biblioteca Güralp/ISTI incluida con `gcf2ew` antes del uso comercial definitivo.

## 13. Criterio de funcionamiento correcto

El servicio se considera operativo cuando se cumplen simultáneamente estas condiciones:

1. El contenedor está `running` y sin reinicios inesperados.
2. `startstop`, `gcf2ew` y `ew2ringserver` están activos.
3. Las conexiones a `172.16.18.87:1568` y `127.0.0.1:16000` están `ESTAB`.
4. Earthworm recibe `TYPE_TRACEBUF2` de los tres canales.
5. SeedLink muestra `CM.DIAM.00.HHE`, `HHN` y `HHZ`.
6. Las fechas finales de los tres streams avanzan.
7. No existe pérdida sostenida o creciente de bloques GCF.
