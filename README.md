# jitsi-on-arm64
Instrucciones para instalar Jitsi-Meet en Armbian 20.08

**1. Deshabilitar zram**

```bash
swapoff -a
systemctl disable armbian-zram-config.service
systemctl disable armbian-ramlog.service
```

**2. Setear /etc/hosts agregando el fqdn y el hostame con el ip**

**3. Instalar JavaVM (adoptopenjdk) y setear entorno**

```bash
wget -qO - https://adoptopenjdk.jfrog.io/adoptopenjdk/api/gpg/key/public | sudo apt-key add -
sudo add-apt-repository --yes https://adoptopenjdk.jfrog.io/adoptopenjdk/deb/
sudo apt update -y
apt install adoptopenjdk-8-hotspot -y
echo "JAVA_HOME=$(readlink -f /usr/bin/java | sed "s:bin/java::")" | sudo tee -a /etc/profile
source /etc/profile
```

**4. Instalar nginx**

```bash
sudo apt install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

**5. Instalar Jitsi-Meet, setear el host, crear el certificado y detener los servicios**

```bash
wget -qO - https://download.jitsi.org/jitsi-key.gpg.key | sudo apt-key add -
echo "deb https://download.jitsi.org stable/"  | sudo tee -a /etc/apt/sources.list.d/jitsi-stable.list
sudo apt update
sudo apt install -y jitsi-meet
sudo /usr/share/jitsi-meet/scripts/install-letsencrypt-cert.sh
sudo systemctl stop prosody jitsi-videobridge2 jicofo
```

**6. Compilar libjnisctp para la plataforma arm**

```bash
sudo apt install automake autoconf build-essential libtool git maven m4
git clone https://github.com/sctplab/usrsctp.git
git clone https://github.com/jitsi/jitsi-sctp
mv ./usrsctp ./jitsi-sctp/usrsctp/
cd ./jitsi-sctp
mvn package -DbuildSctp -DbuildNativeWrapper -DdeployNewJnilib -DskipTests
cp ./jniwrapper/native/target/libjnisctp-linux-arm.so \
 ./jniwrapper/native/src/main/resources/lib/linux/libjnisctp.so
mvn package
sudo cp ./jniwrapper/native/target/jniwrapper-native-1.0-SNAPSHOT.jar \
 $(ls /usr/share/jitsi-videobridge/lib/jniwrapper-native-*)
```

**7. Solucionar errores de certificado de Jicofo y Videobridge contra Prosody**

Editar /etc/jitsi/jicofo/sip-communicator.properties y agregar `org.jitsi.jicofo.ALWAYS_TRUST_MODE_ENABLED=true`

```bash
sudo nano /etc/jitsi/jicofo/sip-communicator.properties
```
luego editar /etc/jitsi/videobridge/sip-communicator.properties y agregar `org.jitsi.videobridge.xmpp.user.SHARD.DISABLE_CERTIFICATE_VERIFICATION=true`

```bash
sudo nano /etc/jitsi/videobridge/sip-communicator.properties
```

**8. Reiniciar los servicios**

```bash
sudo systemctl stop prosody jitsi-videobridge2 jicofo
```

**9. Enjoy**

*Fuentes:*
*Deshabilitar ZRAM https://github.com/MichaIng/DietPi/issues/2738 *
*Instalar Jitsi en Ubuntu 20.04 https://www.vultr.com/docs/install-jitsi-meet-on-ubuntu-20-04-lts *
*Instalar repositorios de adoptopenjdk https://adoptopenjdk.net/installation.html?variant=openjdk8&jvmVariant=hotspot *
*Instalar Jitsi en plataformas arm https://github.com/jitsi/jitsi-meet/issues/6449 *
*Errores de certificados de Prosody https://github.com/jitsi/jitsi-meet/issues/2117 *
