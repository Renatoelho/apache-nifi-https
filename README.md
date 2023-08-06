# Instalando e Configurando o Apache NiFi com Segurança HTTPS

### Requisitos

- Ubuntu 20.04
- Apache Nifi 1.22.0 ou +
- Java OpenJDK 1.8.0_382
- Nano 4.8 ou +
- Wget 1.20.3 ou +
- Zip/Unzip 3.0 ou +

### Criando um usuário no ubuntu

```bash
sudo useradd -ms /bin/bash nifi -G sudo && sudo  passwd -d nifi
```

### Acessando com o novo usuário

```bash
su - nifi
```

### Instalando o Java JDK

```bash
sudo apt install openjdk-8-jdk -y
```

- Verifique se o JAVA foi instalado com sucesso

```bash
java -version
```

```text
...
openjdk version "1.8.0_382"
OpenJDK Runtime Environment (build 1.8.0_382-8u382-ga-1~20.04.1-b05)
OpenJDK 64-Bit Server VM (build 25.382-b05, mixed mode)
...
```

### Baixando o Apache Nifi versão .zip

```bash
sudo wget https://archive.apache.org/dist/nifi/1.22.0/nifi-1.22.0-bin.zip
```

### Descompactando o arquivo zipado

```bash
unzip nifi-1.22.0-bin.zip
```

### Movendo e já renomeando o diretório descompactado para o /opt/

```bash
sudo mv nifi-1.22.0 /opt/nifi
```

### Configurando o JAVA

- Verificando onde está o java instalado

```bash
update-alternatives --config java
```

```text
...
Existe apenas uma alternativa no grupo de ligação java que disponibiliza /usr/bin/java: /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java
Nada para configurar.
...
```

- Adicione a variável ***JAVA_HOME*** no arquivo ***.bashrc***

```text
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```

Salve e saía do arquivo.

- Atualize o ambiente

```bash
source ~/.bashrc
```

- Verifique se a variável foi configurada com sucesso

```bash
env | grep -Ei JAVA_HOME
```

### Criando um certificado autoassinado para acesso HTTPs

- Crie um diretório para armazenar o certificado em ***/opt/nifi/certs***

```bash
sudo mkdir -p /opt/nifi/certs
```

- Crie o arquivo ***openssl.cnf*** com os parâmetros do certificado e salve em ***/opt/nifi/certs***

```text
[req]
default_bits = 2048
distinguished_name = dn
req_extensions = req_ext
prompt = no

[dn]
CN = localhost
OU = Departamento de TI
O = Empresa Nifi
L = Brasilia
ST = DF
C = BR

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
DNS.2 = 127.0.0.1
```

- Crie o script ***create-cert.sh*** que vai gerar o certificado e senha para as configurações HTTPs

```text
#!/bin/bash

# Criando senha do certificado
passwd=$(cat /dev/urandom | tr -dc 'A-Za-z0-9' | head -c 32)

# Criando chaves privada/pública 
openssl req -x509 -newkey rsa:2048 -keyout key-private.pem -out key-public.pem -days 3650 -config openssl.cnf -nodes

# Criando certificado com senha com validade de 3650 dias (10 anos)
openssl pkcs12 -inkey key-private.pem -in key-public.pem -export -out certificate.pfx -passout pass:"$passwd"

# Salvando a senha do certificado
echo "Senha do certificado: $passwd" > password-cert.txt
echo "Certificado criado com sucesso!!!"
```

- Dê permissão de execução para o script ***create-cert.sh***

```bash
chmod +x create-cert.sh
```

- Crie o certificado e os demais arquivos executando o script

```bash
sh ./create-cert.sh
```

```text
...
Generating a RSA private key
..........+++++
....................................................................................................................................................................................................................+++++
writing new private key to 'key-private.pem'
-----
Certificado criado com sucesso!!!
...
```

> ***Observação:*** Esse certificado autoassinado tem validade de 3650 dias (10 anos).
    
    
### Configurando memória da JVM do Apache Nifi

- Edite o seguinte arquivo:

```bash
nano /opt/nifi/conf/bootstrap.conf
```

- Edite os seguintes parâmetros:

```bash
# JVM memory settings
# Xms2g memória inicial da JVM e Xmx2g memória máxima da JVM ambas com 2 GB
java.arg.2=-Xms2g
java.arg.3=-Xmx2g
```

### Outras configurações do Apache Nifi

- Adicione a variável ***export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64*** as variáveis do nifi

```bash
nano /opt/nifi/bin/nifi-env.sh
```

```text
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```

- Configurando o TLS/HTTPS no Apache Nifi

Com as informações do certificado, keystore, truststore e senha criados no diretório /opt/nifi/certs atualize o arquivo ***/opt/nifi/conf/nifi.properties.***

Atualize os seguintes parâmetros com os valores:

```

nifi.remote.input.host=
nifi.remote.input.secure=true
nifi.remote.input.socket.port=
nifi.remote.input.http.enabled=false
#############################################
nifi.web.http.host=
nifi.web.http.port=
#############################################
nifi.web.https.host=0.0.0.0  
nifi.web.https.port=8443
#############################################
nifi.security.autoreload.enabled=false
nifi.security.keystore=/opt/nifi/certs/certificate.pfx
nifi.security.keystoreType=PKCS12
nifi.security.keystorePasswd=<Senha do certificado...>
nifi.security.keyPasswd=<Senha do certificado...>
nifi.security.truststore=/opt/nifi/certs/certificate.pfx
nifi.security.truststoreType=PKCS12
nifi.security.truststorePasswd=<Senha do certificado...>

```

### Iniciando o servidor Apache Nifi

- Executando o install do Nifi

```bash
sudo /opt/nifi/bin/nifi.sh install
```

- Iniciando o serviço do Nifi

```bash
sudo service nifi start 
```

> ***Observação:*** Existem os parâmetros para gerenciar o Nifi: (start|stop|restart|status).

- Status do servidor Nifi

```bash
service nifi status
```

```text
...
● nifi.service - SYSV: Apache NiFi is a dataflow system based on the principles of Flow-Based Programming.
     Loaded: loaded (/etc/init.d/nifi; generated)
     Active: active (running) since Thu 2023-08-03 21:53:34 -03; 2min 1s ago
       Docs: man:systemd-sysv-generator(8)
    Process: 52590 ExecStart=/etc/init.d/nifi start (code=exited, status=0/SUCCESS)
      Tasks: 83 (limit: 4622)
     Memory: 2.3G
     CGroup: /system.slice/nifi.service
             ├─52609 /bin/sh /opt/nifi/bin/nifi.sh start
             ├─52611 /usr/lib/jvm/java-8-openjdk-amd64/bin/java -cp /opt/nifi/conf:/opt/nifi/lib/bootstrap/* -Xms48m -Xmx48m -Dorg.apache.nifi.bo>
             └─52625 /usr/lib/jvm/java-8-openjdk-amd64/bin/java -classpath /opt/nifi/./conf:/opt/nifi/./lib/nifi-runtime-1.22.0.jar:/opt/nifi/./l>
...
```

### Configurando usuário e senha para o Apache Nifi

- Crie o arquivo ***create-user.sh*** em ***/opt/nifi/bin***  e adicione o seguinte conteúdo:

```bash
cd /opt/nifi/bin
```

```bash
nano create-user.sh
```

```text
#!/bin/bash

# Criando uma senha para o usuário Nifi
PASSWD=$(cat /dev/urandom | tr -dc 'A-Za-z0-9' | head -c 32)

# Definindo o usuário e senha (O usuário que vai ser definido aqui vai ser o da sessão.)
./bin/nifi.sh set-single-user-credentials $USER $PASSWD

echo "Usuário: $USER\nSenha: $PASSWD" > credenciais-nifi.txt
echo "Usuário e senha Apache Nifi criados com Sucesso!!!"
```

- Dê permissão de execução para o arquivo

```bash
chmod +x create_user.sh
```

- Para criar usuário e senha execute o seguinte comando

```bash
sh ./create-user.sh
```

- Se tudo dê certo, será criado o arquivo ***credenciais-nifi.txt*** com as credenciais de acesso ao Nifi

```bash
cat credenciais-nifi.txt
```

```
...
Usuário: nifi
Senha: ******************************************************
...
```

### Testando o acesso ao Apache Nifi

Acesse o navegador e digite a seguinte URL: [https://localhost:8443/](https://localhost:8443/) e em seguida digite o usuário e senha do passo anterior, e só utilizar o Apache Nifi agora!!!


# Referências

Docs Nifi, **Apache Nifi**. Disponível em: <https://nifi.apache.org/docs/nifi-docs/>. Acesso em: 02 de ago. de 2023.

Single User Access and HTTPS in Apache NiFi, **ExceptionFactory**. Disponível em: <https://exceptionfactory.com/posts/2021/07/21/single-user-access-and-https-in-apache-nifi/>. Acesso em: 02 de ago. de 2023

Anselmo Borges, **TLS**. Disponível em:<https://github.com/AnselmoBorges/tls/tree/master>. Acesso em: 01 de ago. de 2023.
