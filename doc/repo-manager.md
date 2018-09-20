# repository manager

## For AGL project

> At first you need to initialize your agl project source.
[Documentation](http://docs.automotivelinux.org/docs/getting_started/en/dev/reference/source-code.html)

```bash
mkdir -p "${WORKDIR}/meta" "${WORKDIR}/build"
cd ${WORKDIR}/meta
mkdir -p ~/bin
export PATH=~/bin:$PATH
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

```bash
rm -rf ${WORKDIR}/meta/*
cd ${WORKDIR}/meta
repo init -u https://gerrit.automotivelinux.org/gerrit/AGL/AGL-repo
repo sync
git clone https://github.com/iotbzh/YoMo.git
```

Note : You can choose another source release [Here](http://docs.automotivelinux.org/docs/getting_started/en/dev/reference/source-code.html#download-source)

## For Yocto Project

> At first you need to initialize your yocto project source.

```bash
mkdir -p "${WORKDIR}/meta" "${WORKDIR}/build"
cd ${WORKDIR}/meta
git clone -b sumo git://git.yoctoproject.org/poky.git
git clone https://github.com/iotbzh/YoMo.git
```

Note : You have to change your working directory. Or you have to clean "${WORKDIR}/meta"

> init your build directory

```bash
cd "${WORKDIR}/build"
source ../meta/poky/oe-init-build-env yomo
```
========> . agl-init-build-env
## Build Project

> Add meta-yomo  to conf/bblayers.conf

* Edit your file

```bash
vim conf/bblayers.conf
```

* And add

```bash
BBLAYERS += "${METADIR}/YoMo/meta-yomo"
```

Note: You must replace ${WORKDIR} by your path

* If you use meta-qt5 add:

```bash
BBLAYERS += "${METADIR}/YoMo/meta-qt5-yomo"
```

* Build an image

```bash
bitbake core-image-sato
```

And build your SDK RPM

```bash
bitbake core-image-sato -c populate_sdk
```

## Build yocto-repo-manager

### HTTP server

> You need to have a http server to serve your rpm repositories

For this documentation, we are going to use an Apache server.

But feel free to use your own HTTP server.

#### Install Apache server

For debian:

```bash
sudo apt-get install apache2
```

#### Create certificat

For HTTPS, you need to have a certificat.

```bash
sudo openssl req -new -x509 -days 365 -nodes -out /etc/ssl/certs/yomo.crt -keyout /etc/ssl/private/yomo.key
```

output:

```bash
Generating a 2048 bit RSA private key
.................+++
.............+++
writing new private key to 'yomo.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:FR
State or Province Name (full name) [Some-State]:France
Locality Name (eg, city) []:Vannes
Organization Name (eg, company) [Internet Widgits Pty Ltd]:IoT.bzh
Organizational Unit Name (eg, section) []:IoTVannes
Common Name (e.g. server FQDN or YOUR name) []:XXXX
Email Address []:ronan.lemartret@iot.bzh
```

```bash
sudo chmod 440 /etc/ssl/certs/yomo.crt
```

#### Configure apache server

```bash
export USER_EMAIL="ronan.lemartret@iot.bzh"
export SRV_DIR="/home/devel/share/http_svr/data"
export SRV_LOG="/home/devel/share/http_svr/log"

sudo mkdir -p ${SRV_DIR} ${SRV_LOG}

sudo bash -c "cat << EOF > /etc/apache2/sites-available/http_yomo.conf
<VirtualHost *:80>
    ServerAdmin ${USER_EMAIL}
    ServerName http_yomo
    DocumentRoot ${SRV_DIR}
    <Directory />
        Options FollowSymLinks
        AllowOverride None
    </Directory>
    <Directory ${SRV_DIR}>
        Options Indexes MultiViews
        AllowOverride None
        Require all granted
    </Directory>
    ErrorLog ${SRV_LOG}/error.log
    LogLevel warn
</VirtualHost>
EOF"

sudo bash -c "cat << EOF > /etc/apache2/sites-available/https_yomo.conf
<VirtualHost *:443>
    ServerAdmin ${USER_EMAIL}
    ServerName https_yomo
    DocumentRoot ${SRV_DIR}
    <Directory />
        SSLRequireSSL
        Options FollowSymLinks
        AllowOverride None
    </Directory>

  ServerSignature Off
  LogLevel info

  SSLEngine on
  SSLCertificateFile /etc/ssl/certs/yomo.crt
  SSLCertificateKeyFile /etc/ssl/private/yomo.key

    <Directory ${SRV_DIR}>
        Options Indexes MultiViews
        AllowOverride None
        Require all granted
    </Directory>
    ErrorLog ${SRV_LOG}/error.log
    LogLevel warn
</VirtualHost>
EOF"
```

```bash
sudo rm /etc/apache2/sites-enabled/000-default.conf
```

```bash
sudo a2ensite http_yomo.conf
sudo a2ensite https_yomo.conf
sudo a2enmod ssl
sudo systemctl restart apache2
```

### build the repo tool

```bash
bitbake yocto-repo-manager
bitbake yocto-repo-manager-native -c addto_recipe_sysroot
```

### Publish the rpm repositories

Publish the rpm:

```bash
export ARCH="intel_corei7_64"
export PUBDIR="yomo_repositories/${ARCH}" #add distro, etc.
export SDK_BS_DIR="sdk-bootstrap"
export YOUR_HTTP_SRV="your_http_server"
mkdir -p ${SRV_DIR}/${PUBDIR} ${SRV_DIR}/${SDK_BS_DIR}
oe-run-native yocto-repo-manager-native repo-manager -i ./tmp/deploy/rpm/ -o ${SRV_DIR}/${PUBDIR} -v
```

Publish repositories config file:

```bash
cat >${SRV_DIR}/${SDK_BS_DIR}/SDK-configuration.json<<EOF
{
    "repo":{
        "runtime":{
            "Name":"${ARCH}",
            "runtime":"http://${YOUR_HTTP_SRV}/${PUBDIR}/runtime/"
        },
        "sdk":{
            "sdk":"http://${YOUR_HTTP_SRV}/${PUBDIR}/nativesdk/"
        }
    }
 }
EOF
```
======> url générées : testRepository en trop dans le path

Build SDK config files:

```bash
bitbake sysroots-conf nativesdk-sysroots-conf -c install
```

Publish SDK config files:

```bash
cp tmp/work/x86_64-nativesdk-*-linux/nativesdk-sysroots-conf/0.1-r0/image/opt/*/*/sysroots/x86_64-*-linux/usr/share/sdk_default.json ${SRV_DIR}/${SDK_BS_DIR}
cp tmp/work/*/sysroots-conf/0.1-r0/image/usr/share/*_default.json ${SRV_DIR}/${SDK_BS_DIR}
```

### Build the sdk bootstrap

```bash
bitbake sdk-bootstrap
```

Publish your SDK boot strap

```bash
export SDK_INSTALL_SCRIPT=${basename $(ls tmp/deploy/sdk/x86_64-sdk-bootstrap-*.sh))
cp tmp/deploy/sdk/${SDK_INSTALL_SCRIPT} ${SRV_DIR}/${SDK_BS_DIR}
```

### Init your SDK

At first download and install the sdk-bootstrap

```bash
export BOOTSTRAP_INSTALL=/xdt/sdk-bootstrap
wget http://${YOUR_HTTP_SRV}/${SDK_BS_DIR}/${SDK_INSTALL_SCRIPT}
chmod a=x ./${SDK_INSTALL_SCRIPT}
sudo ./${SDK_INSTALL_SCRIPT} -d ${BOOTSTRAP_INSTALL} -y
```
=======> http://${YOUR_HTTP_SRV}/testRepository/sdk-bootstrap/x86_64-sdk-bootstrap-2.5.sh
=======> ./x86_64-sdk-bootstrap-6.90.0+snapshot-20180918.sh

Init your sdk-bootstrap:

```bash
export PATH=${BOOTSTRAP_INSTALL}/sysroots/x86_64-aglsdk-linux/usr/bin:$PATH
```

Or: ====> AND

```bash
. ${BOOTSTRAP_INSTALL}/environment-setup-x86_64-pokysdk-linux
```
======>environment-setup-x86_64-aglsdk-linux

Download repositories config files:

```bash
wget http://${YOUR_HTTP_SRV}/${SDK_BS_DIR}/SDK-configuration.json
wget http://${YOUR_HTTP_SRV}/${SDK_BS_DIR}/qemux86_default.json
wget http://${YOUR_HTTP_SRV}/${SDK_BS_DIR}/sdk_default.json
```
(====> ARCH)


```bash
init-sdk-rootfs -i qemux86_default.json -i SDK-configuration.json -i sdk_default.json -o /xdt/sdk-yomo
```

### Update your SDK

Init native sysroot:

```bash
cd /xdt/sdk-yomo/${ARCH}-sdk/
./dnf4Native install packagegroup-cross-canadian-*
```
=======> cd /xdt/sdk-yomo/myProject-sdk/

=======> pb avec génération ./dnf4Native cf init-sdk-rootfs.py

Native tool:

```bash
cd /xdt/sdk-yomo/${ARCH}-sdk/
./dnf4Native search cmake
./dnf4Native install nativesdk-cmake
```

Target tool:

```bash
cd /xdt/sdk-yomo/${ARCH}-sdk/
./dnf4Target search libglib-2.0-dev
```

### Use your SDK on AGL

```bash
cd /xdt/sdk-yomo/${ARCH}-sdk/
PKG="nativesdk-packagegroup-qt5-toolchain-host nativesdk-packagegroup-sdk-host packagegroup-cross-canadian-*"
./dnf4Native install ${PKG}
PKG="packagegroup-qt5-toolchain-target linux-libc-headers-dev libjson-c-dev af-binder-dev"
./dnf4Target install ${PKG}

git clone https://gerrit.automotivelinux.org/gerrit/apps/hvac
cd hvac
. /xdt/sdk-yomo/${ARCH}-sdk/env-init-SDK.sh
qmake
make
```
