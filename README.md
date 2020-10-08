# jitsi-on-arm64
Después de haber probado instalar Jitsi-Meet en Ubuntu en una PC, decidí desempolvar mi vieja Renegade (https://libre.computer/products/boards/roc-rk3328-cc/) esperando que fuese igual de simple la instalación. Al falta de un Ubuntu mas actual (la última que viene oficial es 18.04) decidí usar la última versión de Armbian disponible, después de bastantes intentos fallidos y buscar en muchos sitios, logre hacerlo funcionar y armé esta guía. La Renegade es una placa SBC de formato compatible con Raspberry PI, procesador Rockchip RK3328 de arquitectura Arm Cortex-A53 de 64 bits y en este caso 4GB de RAM, por lo que supongo que esta guía debe funcionar en cualquier distribución de paquetes deb con hardware arm64 tipo RPi4, ODroid o Thinkerboard.

*Update* Agregué los cambios para instalar Jitsi-Meet en una Raspberry Pi4 usando Ubuntu Server 20.04

**1. Deshabilitar zram** *Pasar de largo para Raspberry Pi*

Este primer paso se debe a que por default armbian arma una partición de swap utilizando zram, que obviamente consume memoria y le resta para utilizarla por la JavaVM

```bash
swapoff -a
systemctl disable armbian-zram-config.service
systemctl disable armbian-ramlog.service
```

**2. Setear /etc/hosts agregando el fqdn y el hostame con el ip**

Al estar en este caso el host detrás de un NAT, debemos agregar el fqdn del al archivo hosts utilizando el ip interno, por ejemplo:

    192.168.0.10 	jitsionarm.prueba.com jitsionarm

**3. Instalar JavaVM (adoptopenjdk) y setear entorno**

Después de luchar varias veces con el jre y jdk 11 que trae por default de armbian 20.08 busqué alguna implementación estable de Java 8 que tenga soporte y repositorios de fácil instalación y así terminé dando con el proyecto AdoptOpenJDK https://adoptopenjdk.net/

```bash
wget -qO - https://adoptopenjdk.jfrog.io/adoptopenjdk/api/gpg/key/public | sudo apt-key add -
sudo add-apt-repository --yes https://adoptopenjdk.jfrog.io/adoptopenjdk/deb/
sudo apt update -y
apt install adoptopenjdk-8-hotspot -y
echo "JAVA_HOME=$(readlink -f /usr/bin/java | sed "s:bin/java::")" | sudo tee -a /etc/profile
source /etc/profile
```
En el caso del Ubuntu 20.04 en RPi4 es mucho mas sencillo ya que trae en su repositorios varias versiones de JDK

```bash
apt install openjdk-8-jdk-headless -y
echo "JAVA_HOME=$(readlink -f /usr/bin/java | sed "s:bin/java::")" | sudo tee -a /etc/profile
source /etc/profile
```

**4. Instalar nginx**

```bash
sudo apt install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

**5. Instalar Jitsi-Meet, setear el host y crear el certificado**

En este paso el instalador de Jitsi-Meet te solicita el fqdn del host y te consulta si querés utilizar un certificado autofirmado o utilizar alguno existente, para este caso selecciono la primer opción.

```bash
wget -qO - https://download.jitsi.org/jitsi-key.gpg.key | sudo apt-key add -
echo "deb https://download.jitsi.org stable/"  | sudo tee -a /etc/apt/sources.list.d/jitsi-stable.list
sudo apt update
sudo apt install -y jitsi-meet
```

Luego de instalado utilizamos un script que trae el propio Jitsi-Meet para generar un certificado de Let's Encrypt https://letsencrypt.org/es/ que es reconocido por la mayoría de los navegadores y por ende no tenés que instalarlo manualmente.

```bash
sudo /usr/share/jitsi-meet/scripts/install-letsencrypt-cert.sh
```

**6. Compilar libjnisctp para la plataforma arm**

Este quizá es el punto mas complicado de la instalación, ya que el binario que incluye el repositorio de Jitsi-Meet de la libjnisctp.so no viene para arm, por lo que hay que compilarla nativamente junto con la librería jniwrapper que le permite interactuar con la JavaVM, previamente deteniendo los servicios Jicofo, Prosody y Videobridge

```bash
sudo systemctl stop prosody jitsi-videobridge2 jicofo
sudo apt install -y automake autoconf build-essential libtool git maven m4
git clone https://github.com/sctplab/usrsctp.git
git clone https://github.com/jitsi/jitsi-sctp
mv ./usrsctp ./jitsi-sctp/usrsctp/
cd ./jitsi-sctp
mvn package -DbuildSctp -DbuildNativeWrapper -DdeployNewJnilib -DskipTests
cp $(ls ./jniwrapper/native/target/libjnisctp-linux-*) \
./jniwrapper/native/src/main/resources/lib/linux/libjnisctp.so
mvn package
sudo cp ./jniwrapper/native/target/jniwrapper-native-1.0-SNAPSHOT.jar \
 $(ls /usr/share/jitsi-videobridge/lib/jniwrapper-native-*)
```

**7. Solucionar errores de certificado de Jicofo y Videobridge contra Prosody** *Pasar de largo en Raspberry Pi con Ubuntu 20.04* 

Este paso es un workaroud hasta que encuentre la forma correcta de instalar el certificado en Prosody, la falla se detecta cuando aparecen errores `PKIX path building failed` en los logs de Jicofo y Videobridge o más fácil cuando intentan sumar un 3er participante al meet les apaga el audio y video a los restantes. Para eso tenemos que editar /etc/jitsi/jicofo/sip-communicator.properties y agregar `org.jitsi.jicofo.ALWAYS_TRUST_MODE_ENABLED=true`

```bash
sudo nano /etc/jitsi/jicofo/sip-communicator.properties
```
luego editar /etc/jitsi/videobridge/sip-communicator.properties y agregar `org.jitsi.videobridge.xmpp.user.SHARD.DISABLE_CERTIFICATE_VERIFICATION=true`

```bash
sudo nano /etc/jitsi/videobridge/sip-communicator.properties
```

**8. Reiniciar los servicios**

```bash
sudo systemctl start prosody jitsi-videobridge2 jicofo
```

**9. Enjoy**

Acá les dejo los sitios de los cuales fui sacando información que luego se transformó en esta guía.

*Deshabilitar ZRAM https://github.com/MichaIng/DietPi/issues/2738*

*Instalar Jitsi en Ubuntu 20.04 https://www.vultr.com/docs/install-jitsi-meet-on-ubuntu-20-04-lts*

*Instalar repositorios de adoptopenjdk https://adoptopenjdk.net/installation.html?variant=openjdk8&jvmVariant=hotspot*

*Instalar Jitsi en plataformas arm https://github.com/jitsi/jitsi-meet/issues/6449*

*Errores de certificados de Prosody https://github.com/jitsi/jitsi-meet/issues/2117*
