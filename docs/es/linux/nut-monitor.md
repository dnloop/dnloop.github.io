---
publish: true
---

# Nut-Monitor

## Introducción

Estaba realizando mantenimiento de sistema y descubrí algunos inconvenientes con mi configuración actual, de manera que decidí documentar el proceso y compartirlo.

Utilizo la UPS CTB-1200. Está conectada por cable usb. luego de seguir las instrucciones de la [wiki](https://wiki.archlinux.org/title/Network_UPS_Tools#) y establecer una configuración básica; los siguientes archivos necesitaron modificaciones para que la instalación sea efectiva.

## Configuración

### Configuración del driver

No todos los modelos son compatibles. Tuve la suerte de encontrar mi dispositivo en la página de [Network UPS Tools](https://networkupstools.org/stable-hcl.html), la cual provee los parámetros de configuración necesarios.

Existen varios archivos que deben modificarse de acuerdo al modelo de UPS del usuario, los cuales residen en la carpeta `/etc/nut`.

```shell
/etc/nut
├── hosts.conf
├── nut.conf # Habilitar el monitor
├── ups.conf # Configuración del driver 
├── upsd.conf
├── upsd.users # Requerido para enviar comandos al demonio
├── upsmon.conf # Configuración del demonio
├── upssched.conf # Temporizador y planificación de comandos
├── upssched-cmd # Script personalizado del usuario
├── upsset.conf
├── upsstats.html
└── upsstats-single.html
```

#### **nut.conf**
 
 En este archivo se debe cambiar la siguiente línea de 'none' a 'standalone' para habilitar el monitor.

```sh
# Valores y comentarios por defecto omitidos
MODE=standalone 
```

#### **ups.conf**

En este archivo se encuentra la configuración del driver. La compatibilidad de los parámetros dependerán del driver. A continuación se muestran valores de propósito general que deberían funcionar para la mayoría de los drivers.

```shell
# Valores y comentarios por defecto omitidos, 
# agregar la configuración del driver deseado
# al final.
[ctb-1200]
    driver = blazer_usb
    port = auto
    offdelay = 20
    ondelay = 30
    ignorelb
    override.battery.charge.low = 20
    override.battery.charge.warning = 40
```

#### **ups.users**

En este archivo se encuentran las configuraciones de usuario. Es requerido para enviar comandos al demonio.

```sh
[dnloop]
     password = 1212
     upsmon master
     actions = SET
     instcmds = ALL
```

#### **upsmon.conf**

En este archivo se encuentra la configuración del demonio. Proporciona una interfaz para enviar comandos al dispositivo.

```sh
# Valores y comentarios por defecto omitidos
SHUTDOWNCMD "/sbin/shutdown -h +0"
NOTIFYCMD /usr/sbin/upssched
NOTIFYFLAG ONLINE      SYSLOG+WALL+EXEC
NOTIFYFLAG ONBATT      SYSLOG+WALL+EXEC
NOTIFYFLAG LOWBATT     SYSLOG+WALL+EXEC
NOTIFYFLAG REPLBATT    SYSLOG+WALL+EXEC
MONITOR ctb-1200@localhost 1 dnloop 1212 master # misma credenciales que ups.users
```

#### **upssched.conf**

En este archivo se encuentra la configuración del temporizador y planificación de comandos. Aquí se puede ajustar el tiempo de espera hasta la re conexión o apagón. Es recomendable no esperar demasiado en una interrupción de electricidad debido a la edad de la batería, es mejor apagar rápido el sistema que esperar para apagar y la batería no mantenga la carga suficiente para terminar el sistema.

```sh
# Valores y comentarios por defecto omitidos
CMDSCRIPT /etc/nut/upssched-cmd

PIPEFN /var/lib/nut/upssched/upssched.pipe
LOCKFN /var/lib/nut/upssched/upssched.lock

AT ONBATT * START-TIMER shutdown_onbatt 40 # valor en segundos
AT ONBATT * EXECUTE info_onbatt

AT ONLINE * CANCEL-TIMER shutdown_onbatt
AT ONLINE * EXECUTE ups-back-on-power

AT LOWBATT * EXECUTE shutdown_lowbatt

AT REPLBATT * EXECUTE replace_batt

```

### Script personalizado

Este script es donde se almacenan los comandos personalizados. Se llama a través de `upssched.conf`. Se puede modificar para ejecutar el manejo apropiado de los servicios y notificar los registros de eventos. Solo estoy registrando y forzando un apagado por ahora.

**/var/lib/nut/upssched/**

Los archivos de pipe y lock son necesarios para que el demonio funcione e informe al registrador de mensajes el estado de los comandos ejecutados.

```sh
touch upssched.pipe
touch upssched.lock
chown nut:nut upssched.*
chmod 770 upssched.*
```

**upssched-cmd**

Este script es donde se almacenan los comandos personalizados. Se llama a través de `upssched.conf`. Se puede modificar para ejecutar el manejo apropiado de los servicios y notificar los registros de eventos. Solo estoy registrando y forzando un apagado por ahora.

```sh
#! /bin/sh
case $1 in
    shutdown_onbatt)
        logger -t upsmon[upssched] "shutdown_onbatt): Triggering shutdown after being on battery"
        /sbin/shutdown -h +0
        ;;

    shutdown_lowbatt)
        logger -t upsmon[upssched] "shutdown_lowbatt): Triggering shutdown when battery.charge.low is under 50%"
        /sbin/shutdown -h +0
        ;;

    info_onbatt)
        logger -t upsmon[upssched] "info_onbatt): Now on battery"
        ;;

    ups-back-on-power)
        logger -t upsmon[upssched] "ups-back-on-power): UPS back on power"
        ;;

    replace_batt)
        message="Quick self-test indicates battery requires replacement"
        logger -t upsmon[upssched] "replace_batt): $message"
        ;;

    *)
        logger -t upsmon[upssched] "*) = Unrecognized command: $1"
        ;;
esac

```

### Systemd Targets

Estos son los objetivos de systemd extraídos de la wiki de Arch, deberían estar presentes después de seguir el guía.

```sh
╭─ /etc/systemd/system/

nut-driver@ctb-1200.service.d
├── nut-driver-enumerator-generated-checksum.conf
├── nut-driver-enumerator-generated.conf
├── nut-driver-enumerator-generated-devicename.conf
└── nut-driver-enumerator-generated-doclink.conf
nut-driver@.service.d
└── nut-driver-enumerator-generated-checksum.conf
nut-driver.target.wants
└── nut-driver@ctb-1200.service -> /usr/lib/systemd/system/nut-driver@.service
nut.target.wants
├── nut-driver-enumerator.service -> /usr/lib/systemd/system/nut-driver-enumerator.service
├── nut-driver.target -> /usr/lib/systemd/system/nut-driver.target
└── nut-server.service -> /usr/lib/systemd/system/nut-server.service

```

### Resumen de servicios

* nut-server.service
* nut-monitor.service
* nut-driver-enumerator.service

### Luego de la instalación

Se puede verificar una instalación correcta con: 

```sh
sudo upsd
```

Debería mostrar algo similar a:

```sh
Network UPS Tools upsd 2.8.3 release
not listening on ::1 port 3493
not listening on 127.0.0.1 port 3493
no listening interface available
```

Un servidor configurado de forma apropiada debería mostrar entradas similares, la más reciente es cuando la UPS se desconecta de la pared. Las siguientes entradas, `info_onbatt` y `ups-back-on-power` provienen del script personalizado.

**`sudo journalctl -f` output**

```sh 
Broadcast message from nut@lyra (somewhere) (Mon Jun 23 17:36:42 2025):        
                                                                               
UPS ctb-1200@localhost on battery                                              
                                                                               
                                                                               
Broadcast message from nut@lyra (somewhere) (Mon Jun 23 17:36:42 2025):        
                                                                               
UPS ctb-1200@localhost on battery                                              
                                                                               
Jun 23 17:36:42 lyra nut-monitor[8567]: UPS ctb-1200@localhost on battery
Jun 23 17:36:43 lyra upsmon[9256]: info_onbatt): Now on battery
...
[OMITIDO]
...                                                                              
Broadcast message from nut@lyra (somewhere) (Mon Jun 23 17:36:52 2025):        
                                                                               
UPS ctb-1200@localhost on line power                                           
                                                                               
                                                                               
Broadcast message from nut@lyra (somewhere) (Mon Jun 23 17:36:52 2025):        
                                                                               
UPS ctb-1200@localhost on line power                                           
                                                                               
Jun 23 17:36:52 lyra nut-monitor[8567]: UPS ctb-1200@localhost on line power
Jun 23 17:36:52 lyra upsmon[9395]: ups-back-on-power): UPS back on power

```

**`sudo journalctl -rxb | grep upsd` output**

```sh   
Jun 23 17:25:17 lyra upsd[8540]: User dnloop@::1 logged into UPS [ctb-1200]
Jun 23 17:25:13 lyra upsd[8540]: upsnotify: logged the systemd watchdog situation once, will not spam more about it
Jun 23 17:25:13 lyra upsd[8540]: upsnotify: failed to notify about state NOTIFY_STATE_READY_WITH_PID: no notification tech defined, will not spam more about it
Jun 23 17:25:13 lyra upsd[8540]: upsnotify: notify about state NOTIFY_STATE_READY_WITH_PID with libsystemd: was requested, but not running as a service unit now, will not spam more about it
Jun 23 17:25:13 lyra upsd[8540]: Running as foreground process, not saving a PID file
Jun 23 17:25:13 lyra upsd[8540]: Found 1 UPS defined in ups.conf
Jun 23 17:25:13 lyra upsd[8540]: Connected to UPS [ctb-1200]: blazer_usb-ctb-1200
Jun 23 17:25:13 lyra upsd[8540]: listening on 127.0.0.1 port 3493
Jun 23 17:25:13 lyra upsd[8540]: listening on ::1 port 3493
Jun 23 17:25:13 lyra nut-server[8540]: Network UPS Tools upsd 2.8.3 release
```

**`sudo journalctl -rxb | grep nut-server` output**

```sh
Jun 23 17:25:17 lyra nut-server[8540]: User dnloop@::1 logged into UPS [ctb-1200]
Jun 23 17:25:13 lyra nut-server[8540]: upsnotify: logged the systemd watchdog situation once, will not spam more about it
Jun 23 17:25:13 lyra nut-server[8540]: upsnotify: failed to notify about state NOTIFY_STATE_READY_WITH_PID: no notification tech defined, will not spam more about it
Jun 23 17:25:13 lyra nut-server[8540]: upsnotify: notify about state NOTIFY_STATE_READY_WITH_PID with libsystemd: was requested, but not running as a service unit now, will not spam more about it
Jun 23 17:25:13 lyra nut-server[8540]: Running as foreground process, not saving a PID file
Jun 23 17:25:13 lyra nut-server[8540]: Found 1 UPS defined in ups.conf
Jun 23 17:25:13 lyra nut-server[8540]: Connected to UPS [ctb-1200]: blazer_usb-ctb-1200
Jun 23 17:25:13 lyra nut-server[8540]: listening on 127.0.0.1 port 3493       
Jun 23 17:25:13 lyra nut-server[8540]: listening on ::1 port 3493  
```

Para obtener información sobre la UPS, ejecutar `upsc <upsname>`

```sh
upsc ctb-1200
battery.charge: 100
battery.charge.low: 20
battery.charge.warning: 40
battery.voltage: 27.10
battery.voltage.high: 26.00
battery.voltage.low: 20.80
battery.voltage.nominal: 24.0
device.type: ups
driver.debug: 0
driver.flag.allow_killpower: 0
driver.flag.ignorelb: enabled
driver.name: blazer_usb
driver.parameter.offdelay: 20
driver.parameter.ondelay: 30
driver.parameter.override.battery.charge.low: 20
driver.parameter.override.battery.charge.warning: 40
driver.parameter.pollinterval: 2
driver.parameter.port: auto
driver.parameter.synchronous: auto
driver.state: quiet
driver.version: 2.8.3
driver.version.internal: 0.21
driver.version.usb: libusb-1.0.29 (API: 0x0100010A)
input.current.nominal: 5.0
input.frequency: 50.0
input.frequency.nominal: 50
input.voltage: 224.4
input.voltage.fault: 224.4
input.voltage.nominal: 220
output.voltage: 226.5
ups.beeper.status: enabled
ups.delay.shutdown: 18
ups.delay.start: 1800
ups.load: 5
ups.productid: 5161
ups.status: OL
ups.type: offline / line interactive
ups.vendorid: 0665
```

## Referencias

* [Network UPS Tools](https://wiki.archlinux.org/title/Network_UPS_Tools#NUT-Monitor)
* [NUT & UPS, howto shut down client after 2 min on battery?](https://community.ipfire.org/t/nut-ups-howto-shut-down-client-after-2-min-on-battery/9096/6)
* [nut – testing the shutdown mechanism](https://dan.langille.org/2020/09/10/nut-testing-the-shutdown-mechanism/)
* [6. Configuration notes](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html)
