# Venus-Os-rasberry-pi-4
Guía de Integración del Módulo de Comunicación RemoteGPIO

Este documento describe la arquitectura de RemoteGPIO y explica cómo desarrollar un módulo de comunicación
personalizado para integrar cualquier dispositivo de E/S con Venus OS de Victron. Información extraida de: https://community.victronenergy.com/t/remotegpio-v4-0-0-release-beta-for-now/41652

1. Arquitectura del Sistema
RemoteGPIO se divide en dos partes para garantizar estabilidad y modularidad:
1.1 Motor Central (dbus-rgpio.py)
 Servicio persistente que no conoce protocolos de hardware específicos.
 Funciones principales:
o Administrar el módulo del kernel para GPIO virtuales.
o Crear y administrar servicios D-Bus (com.victronenergy.switch.*).
o Crear /run/io-ext/ para el descubrimiento de entradas digitales.
o Proporcionar una API MQTT interna para comunicación con módulos.

1.2 Módulos de Comunicación (ej. dingtian_mqtt_bridge.py)
 Scripts independientes que se ejecutan como servicios.
 Funciones:
o Comunicar con un dispositivo específico (MQTT, Modbus, etc.).
o Traducir estados del hardware a llamadas de la API del Motor Central.
o Traducir comandos del Motor Central a comandos de hardware.

2. Estructura de Archivos y Gestión de Servicios
A. Ubicación de archivos
Archivo Ruta
Script de módulo /data/RemoteGPIO/modules/
Archivo de configuración
.ini

/data/RemoteGPIO/conf/

Servicio DaemonTools /service/ (ej. /service/my_new_bridge)
B. Control de Servicio con setup_rgpio.py
 Permite iniciar/detener módulos desde la GUI de Venus OS.
 Para controlar un módulo, agregarlo a /data/RemoteGPIO/conf/setup_rgpio.ini.
Ejemplo para Relay4:
[device_config]
# ... otras configuraciones ...
Relay4 = My New Module
Service4 = my_new_bridge
 Relay4: nombre amigable en GUI.
 Service4: nombre exacto del servicio en /service/.
Cuando el usuario alterna “Relay4” en la GUI:
svc -u /service/my_new_bridge # iniciar
svc -d /service/my_new_bridge # detener

3. API Interna (MQTT)
Toda comunicación con el Motor Central se realiza mediante MQTT en localhost:1883.
A. Registro y Desregistro de Dispositivos
Registro:
Tema: rgpio/api/device/register/MYDEVICE_001
Payload: {&quot;serial&quot;: &quot;MYDEVICE_001&quot;, &quot;num_inputs&quot;: &quot;4&quot;, &quot;num_relays&quot;: &quot;8&quot;, &quot;device_instance&quot;: &quot;260&quot;}
Retain: true
 Crea servicio D-Bus com.victronenergy.switch.MYDEVICE_001.
 Crea /run/io-ext/MYDEVICE_001 con enlaces simbólicos.
Desregistro:
Tema: rgpio/api/device/register/MYDEVICE_001
Payload: &quot;UNREGISTER&quot;
Retain: true

B. Estado del Dispositivo (Conexión)
Tema: rgpio/api/device/status/MYDEVICE_001
Payload: &quot;CONNECTED&quot; # o &quot;DISCONNECTED&quot;
Retain: true
 Actualiza /State y /Connected en D-Bus.

C. Entradas Digitales (Módulo → Motor)
 Comunicación unidireccional: módulo escribe en sysfs.
input_path = &quot;/run/io-ext/MYDEVICE_001/input_2&quot;
new_value = &quot;1&quot;
try:
with open(f&quot;{input_path}/direction&quot;, &#39;w&#39;) as f:
f.write(&#39;out&#39;)
with open(f&quot;{input_path}/value&quot;, &#39;w&#39;) as f:
f.write(new_value)
with open(f&quot;{input_path}/direction&quot;, &#39;w&#39;) as f:
f.write(&#39;in&#39;)
except IOError as e:
logger.error(f&quot;No se pudo escribir en la ruta sysfs: {e}&quot;)

D. Comandos de Relé (Motor → Módulo)
Tema: rgpio/api/relay/set/MYDEVICE_001/4
Payload: &quot;ON&quot; # o &quot;OFF&quot;
 Módulo debe traducir a comando físico para el relé.

E. Retroalimentación del Estado del Relé (Módulo → Motor)
Tema: rgpio/api/device/relay/read/MYDEVICE_001/1
Payload: &quot;OFF&quot;
 Actualiza /SwitchableOutput/relay_1/State en D-Bus.

F. Nombres de Relé (Opcional)
Tema: rgpio/api/device/MYDEVICE_001/Relay_1
Payload: &quot;Bomba de Agua&quot;
 Actualiza /SwitchableOutput/relay_1/Name en D-Bus.
 Aparece en la GUI de Venus OS.

4. Ejemplo de Flujo Completo
1. Módulo detecta hardware conectado → registra dispositivo en MQTT.
2. Motor Central crea entradas y salidas virtuales en D-Bus.
3. Usuario activa un relé desde GUI → MQTT envía comando al módulo → módulo enciende relé físico.
4. Módulo reporta estado del relé → Motor Central actualiza GUI y D-Bus.
5. Cuando módulo se apaga → envía UNREGISTER → limpieza de entradas/salidas.
