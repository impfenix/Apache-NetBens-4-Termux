Leia este guia em Inglês | 阅读中文版指南

# Guia de Compilação e Empacotamento
Este documento detalha o processo para compilar o Apache NetBeans a partir do código-fonte diretamente no Termux e como empacotar o resultado em um instalador ```.apk```.

Parte 1: Compilando o NetBeans no Termux
Este processo é demorado e consome muitos recursos. Execute-o com seu dispositivo conectado à energia e a uma rede Wi-Fi estável.

Script de Compilação
Este script automatiza a instalação de dependências, o download do código-fonte e a compilação.

Copie o conteúdo abaixo e salve em um arquivo chamado build-netbeans.sh no seu Termux.

Dê permissão de execução com o comando: chmod +x build-netbeans.sh.

Execute o script com: ./build-netbeans.sh.

#!/data/data/com.termux/files/usr/bin/bash

# --- Script para Compilar e Instalar o Apache NetBeans no Termux ---

echo "=== PASSO 1: Instalando dependências de compilação ==="
pkg install ant openjdk-17 git -y

echo ""
echo "=== PASSO 2: Baixando o código-fonte do Apache NetBeans (isso pode demorar) ==="
git clone [https://github.com/apache/netbeans.git](https://github.com/apache/netbeans.git)
cd netbeans

echo ""
echo "=== PASSO 3: Compilando o NetBeans (este é o passo mais demorado!) ==="
ant

echo ""
echo "=== PASSO 4: Instalando o NetBeans no diretório do Termux ==="
mv nbbuild/netbeans /data/data/com.termux/files/usr/share/netbeans

echo ""
echo "=== PASSO 5: Criando um atalho de execução ==="
cat <<'EOF' > /data/data/com.termux/files/usr/bin/netbeans
#!/data/data/com.termux/files/usr/bin/sh
export JAVA_HOME=/data/data/com.termux/files/usr/lib/jvm/java-17-openjdk
/data/data/com.termux/files/usr/share/netbeans/bin/netbeans
EOF
chmod +x /data/data/com.termux/files/usr/bin/netbeans

echo ""
echo "=========================================="
echo "    INSTALAÇÃO CONCLUÍDA COM SUCESSO!"
echo "=========================================="

Parte 2: Empacotando o Instalador .apk
Após a compilação, use os passos a seguir para criar o pacote de instalação .apk. Este processo utiliza o Google Cloud Shell para contornar limitações de ferramentas e espaço no Termux.

2.1 - Preparação no Termux
Primeiro, crie um pacote .zip com os arquivos compilados.

# Crie a pasta de destino no Android
mkdir -p /sdcard/Download/Ports-4-Termux
# Copie os arquivos do NetBeans
cp -r /data/data/com.termux/files/usr/share/netbeans /sdcard/Download/Ports-4-Termux/
cp /data/data/com.termux/files/usr/bin/netbeans /sdcard/Download/Ports-4-Termux/netbeans_launcher
# Crie o script de instalação
cat <<'EOF' > /sdcard/Download/Ports-4-Termux/install.sh
#!/data/data/com.termux/files/usr/bin/bash
echo "Instalando Apache NetBeans para Termux..."
mv ./netbeans /data/data/com.termux/files/usr/share/
mv ./netbeans_launcher /data/data/com.termux/files/usr/bin/netbeans
chmod +x /data/data/com.termux/files/usr/bin/netbeans
echo "NetBeans instalado com sucesso!"
EOF
# Crie o arquivo .zip final
cd /sdcard/Download/
zip -r netbeans-termux-package.zip ./Ports-4-Termux/

2.2 - Construção no Google Cloud Shell
Acesse o Google Cloud Shell.

Instale as ferramentas do Android SDK com os comandos abaixo:

wget [https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip](https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip) && unzip commandlinetools-linux-11076708_latest.zip -d android_sdk/cmdline-tools && mv android_sdk/cmdline-tools/cmdline-tools/* android_sdk/cmdline-tools/ && rmdir android_sdk/cmdline-tools/cmdline-tools && rm commandlinetools-linux-11076708_latest.zip

yes | ./android_sdk/cmdline-tools/bin/sdkmanager --sdk_root=./android_sdk "platforms;android-34" "build-tools;34.0.0"

Faça o upload do pacote netbeans-termux-package.zip e do seu ícone para o instalador (ex: installer_icon.png).

Execute os comandos no terminal do Cloud Shell para preparar o ambiente, empacotar e assinar o .apk.

# Preparar estrutura e arquivos
mkdir -p ~/apk-builder/res/drawable
mkdir -p ~/apk-builder/assets
cat <<'EOF' > ~/apk-builder/AndroidManifest.xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="[http://schemas.android.com/apk/res/android](http://schemas.android.com/apk/res/android)"
  package="com.luanadsleite.netbeansinstaller"
  android:versionCode="1"
  android:versionName="1.0">
  <application android:label="Apache NetBeans Termux Installer" android:icon="@drawable/icon">
  </application>
</manifest>
EOF
mv ~/installer_icon.png ~/apk-builder/res/drawable/icon.png
unzip ~/netbeans-termux-package.zip -d ~/apk-builder/assets/
mv ~/apk-builder/assets/Ports-4-Termux/* ~/apk-builder/assets/
rmdir ~/apk-builder/assets/Ports-4-Termux

# Definir variáveis de ambiente
BUILD_TOOLS=~/android_sdk/build-tools/34.0.0
PLATFORM=~/android_sdk/platforms/android-34/android.jar

# Empacotar e Assinar
cd ~/apk-builder
$BUILD_TOOLS/aapt package -f -M AndroidManifest.xml -S res -I $PLATFORM -F netbeans-unsigned.apk
zip -r netbeans-unsigned.apk assets/
keytool -genkey -v -keystore termux-dev.keystore -alias devkey -keyalg RSA -keysize 2048 -validity 10000
$BUILD_TOOLS/apksigner sign --ks termux-dev.keystore netbeans-unsigned.apk
mv netbeans-unsigned.apk ~/netbeans-termux-installer.apk

Baixe o .apk final a partir do menu "Download" do Cloud Shell.
