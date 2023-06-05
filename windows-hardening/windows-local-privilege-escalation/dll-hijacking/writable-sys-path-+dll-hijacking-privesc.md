# Writable Sys Path +Dll Hijacking Privesc

<details>

<summary><a href="https://cloud.hacktricks.xyz/pentesting-cloud/pentesting-cloud-methodology"><strong>☁️ HackTricks Cloud ☁️</strong></a> -<a href="https://twitter.com/hacktricks_live"><strong>🐦 Twitter 🐦</strong></a> - <a href="https://www.twitch.tv/hacktricks_live/schedule"><strong>🎙️ Twitch 🎙️</strong></a> - <a href="https://www.youtube.com/@hacktricks_LIVE"><strong>🎥 Youtube 🎥</strong></a></summary>

* ¿Trabajas en una **empresa de ciberseguridad**? ¿Quieres ver tu **empresa anunciada en HackTricks**? ¿O quieres tener acceso a la **última versión de PEASS o descargar HackTricks en PDF**? ¡Consulta los [**PLANES DE SUSCRIPCIÓN**](https://github.com/sponsors/carlospolop)!
* Descubre [**The PEASS Family**](https://opensea.io/collection/the-peass-family), nuestra colección exclusiva de [**NFTs**](https://opensea.io/collection/the-peass-family)
* Obtén el [**swag oficial de PEASS y HackTricks**](https://peass.creator-spring.com)
* **Únete al** [**💬**](https://emojipedia.org/speech-balloon/) [**grupo de Discord**](https://discord.gg/hRep4RUj7f) o al [**grupo de telegram**](https://t.me/peass) o **sígueme** en **Twitter** [**🐦**](https://github.com/carlospolop/hacktricks/tree/7af18b62b3bdc423e11444677a6a73d4043511e9/\[https:/emojipedia.org/bird/README.md)[**@carlospolopm**](https://twitter.com/hacktricks_live)**.**
* **Comparte tus trucos de hacking enviando PRs al** [**repositorio de hacktricks**](https://github.com/carlospolop/hacktricks) **y al** [**repositorio de hacktricks-cloud**](https://github.com/carlospolop/hacktricks-cloud).

</details>

## Introducción

Si descubres que puedes **escribir en una carpeta de la Ruta del Sistema** (ten en cuenta que esto no funcionará si puedes escribir en una carpeta de la Ruta del Usuario), es posible que puedas **escalar privilegios** en el sistema.

Para hacerlo, puedes abusar de un **Dll Hijacking** donde vas a **secuestrar una biblioteca que está siendo cargada** por un servicio o proceso con **más privilegios** que los tuyos, y debido a que ese servicio está cargando una Dll que probablemente ni siquiera existe en todo el sistema, intentará cargarla desde la Ruta del Sistema donde puedes escribir.

Para obtener más información sobre **qué es el Dll Hijacking**, consulta:

{% content-ref url="../dll-hijacking.md" %}
[dll-hijacking.md](../dll-hijacking.md)
{% endcontent-ref %}

## Privesc con Dll Hijacking

### Encontrar una Dll faltante

Lo primero que necesitas es **identificar un proceso** que se esté ejecutando con **más privilegios** que tú y que esté intentando **cargar una Dll desde la Ruta del Sistema** en la que puedes escribir.

El problema en estos casos es que probablemente esos procesos ya estén en ejecución. Para encontrar qué Dlls faltan en los servicios que necesitas, debes lanzar procmon lo antes posible (antes de que se carguen los procesos). Entonces, para encontrar las Dlls faltantes, haz lo siguiente:

* **Crea** la carpeta `C:\privesc_hijacking` y agrega la ruta `C:\privesc_hijacking` a la **variable de entorno de Ruta del Sistema**. Puedes hacer esto **manualmente** o con **PS**:
```powershell
# Set the folder path to create and check events for
$folderPath = "C:\privesc_hijacking"

# Create the folder if it does not exist
if (!(Test-Path $folderPath -PathType Container)) {
    New-Item -ItemType Directory -Path $folderPath | Out-Null
}

# Set the folder path in the System environment variable PATH
$envPath = [Environment]::GetEnvironmentVariable("PATH", "Machine")
if ($envPath -notlike "*$folderPath*") {
    $newPath = "$envPath;$folderPath"
    [Environment]::SetEnvironmentVariable("PATH", $newPath, "Machine")
}
```
* Ejecuta **`procmon`** y ve a **`Options`** --> **`Enable boot logging`** y presiona **`OK`** en el mensaje.
* Luego, **reinicia**. Cuando la computadora se reinicie, **`procmon`** comenzará a **grabar** eventos lo antes posible.
* Una vez que **Windows** se haya **iniciado, ejecuta `procmon`** nuevamente, te dirá que ha estado ejecutándose y te **preguntará si deseas almacenar** los eventos en un archivo. Di **sí** y **almacena los eventos en un archivo**.
* **Después** de que se **genere el archivo**, **cierra** la ventana abierta de **`procmon`** y **abre el archivo de eventos**.
* Agrega estos **filtros** y encontrarás todas las Dlls que algún **proceso intentó cargar** desde la carpeta de Ruta del sistema escribible:

<figure><img src="../../../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

### Dlls perdidas

Al ejecutar esto en una **máquina virtual (vmware) de Windows 11** gratuita, obtuve estos resultados:

<figure><img src="../../../.gitbook/assets/image (253).png" alt=""><figcaption></figcaption></figure>

En este caso, los .exe son inútiles, así que ignóralos, las Dlls perdidas eran de:

| Servicio                         | Dll                | Línea de comando                                                    |
| ------------------------------- | ------------------ | -------------------------------------------------------------------- |
| Programador de tareas (Schedule)       | WptsExtensions.dll | `C:\Windows\system32\svchost.exe -k netsvcs -p -s Schedule`          |
| Servicio de directiva de diagnóstico (DPS) | Unknown.DLL        | `C:\Windows\System32\svchost.exe -k LocalServiceNoNetwork -p -s DPS` |
| ???                             | SharedRes.dll      | `C:\Windows\system32\svchost.exe -k UnistackSvcGroup`                |

Después de encontrar esto, encontré esta interesante publicación de blog que también explica cómo [**abusar de WptsExtensions.dll para privesc**](https://juggernaut-sec.com/dll-hijacking/#Windows\_10\_Phantom\_DLL\_Hijacking\_-\_WptsExtensionsdll). Que es lo que **haremos ahora**.

### Explotación

Entonces, para **escalar privilegios**, vamos a secuestrar la biblioteca **WptsExtensions.dll**. Teniendo la **ruta** y el **nombre**, solo necesitamos **generar la Dll maliciosa**.

Puedes [**intentar usar cualquiera de estos ejemplos**](../dll-hijacking.md#creating-and-compiling-dlls). Podrías ejecutar cargas útiles como: obtener una shell inversa, agregar un usuario, ejecutar un beacon...

{% hint style="warning" %}
Ten en cuenta que **no todos los servicios se ejecutan** con **`NT AUTHORITY\SYSTEM`** algunos también se ejecutan con **`NT AUTHORITY\LOCAL SERVICE`** que tiene **menos privilegios** y no podrás crear un nuevo usuario abusando de sus permisos.\
Sin embargo, ese usuario tiene el privilegio **`seImpersonate`**, por lo que puedes usar la [**suite potato para escalar privilegios**](../roguepotato-and-printspoofer.md). Entonces, en este caso, una shell inversa es una mejor opción que intentar crear un usuario.
{% endhint %}

En el momento de escribir esto, el servicio **Programador de tareas** se ejecuta con **Nt AUTHORITY\SYSTEM**.

Habiendo **generado la Dll maliciosa** (_en mi caso usé una shell inversa x64 y obtuve una shell de vuelta, pero Defender la mató porque era de msfvenom_), guárdala en la Ruta del sistema escribible con el nombre **WptsExtensions.dll** y **reinicia** la computadora (o reinicia el servicio o haz lo que sea necesario para volver a ejecutar el servicio/programa afectado).

Cuando se reinicie el servicio, la **dll debería cargarse y ejecutarse** (puedes **reutilizar** el truco de **procmon** para verificar si la **biblioteca se cargó como se esperaba**).

<details>

<summary><a href="https://cloud.hacktricks.xyz/pentesting-cloud/pentesting-cloud-methodology"><strong>☁️ HackTricks Cloud ☁️</strong></a> -<a href="https://twitter.com/hacktricks_live"><strong>🐦 Twitter 🐦</strong></a> - <a href="https://www.twitch.tv/hacktricks_live/schedule"><strong>🎙️ Twitch 🎙️</strong></a> - <a href="https://www.youtube.com/@hacktricks_LIVE"><strong>🎥 Youtube 🎥</strong></a></summary>

* ¿Trabajas en una **empresa de ciberseguridad**? ¿Quieres ver tu **empresa anunciada en HackTricks**? ¿O quieres tener acceso a la **última versión de PEASS o descargar HackTricks en PDF**? ¡Revisa los [**PLANES DE SUSCRIPCIÓN**](https://github.com/sponsors/carlospolop)!
* Descubre [**The PEASS Family**](https://opensea.io/collection/the-peass-family), nuestra colección de [**NFTs**](https://opensea.io/collection/the-peass-family) exclusivos.
* Obtén el [**swag oficial de PEASS y HackTricks**](https://peass.creator-spring.com)
* **Únete al** [**💬**](https://emojipedia.org/speech-balloon/) [**grupo de Discord**](https://discord.gg/hRep4RUj7f) o al [**grupo de telegram**](https://t.me/peass) o **sígueme** en **Twitter** [**🐦**](https://github.com/carlospolop/hacktricks/tree/7af18b62b3bdc423e11444677a6a73d4043511e9/\[https:/emojipedia.org/bird/README.md)[**@carlospolopm**](https://twitter.com/hacktricks_live).
* **Comparte tus trucos de hacking enviando PRs al** [**repositorio de hacktricks**](https://github.com/carlospolop/hacktricks) **y al** [**repositorio de hacktricks-cloud**](https://github.com/carlospolop/hacktricks-cloud).

</details>