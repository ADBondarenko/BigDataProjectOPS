# Инструкция по деплою Apache Hadoop, Hive на кластере из 4-х удаленных машин

## 1. Установка и настройка Hadoop HDFS

### SSH-подключение

Для начала необходимо создать SSH-ключи для подключения к удаленной машине. По умолчанию мы считаем, что мы поключаемся к JumpNode - 1-ой из 4-х машин в кластере: 

1. **JumpNode**
2. NameNode
3. DataNode0
4. DataNode1

Для этого (при наличии ключей в директории, из которой осуществляется поключение) необходимо выполнить команду: 

```
ssh -i [./path/to/key] team@176.109.81.245 
```

По умолчанию, мы входим под SUDO'ером **team** на 22-порту.

После входа нам необходимо создать нового юзера (без sudo прав, если для команды нужно sudo - мы всегда можем прибегнуть к sudo su) hadoop, для которого мы и будем разворачивать кластер.

### Конфигурация

#### Пользователи и права
Создадим нового юзера без sudo прав:
 
```
sudo adduser hadoop
```
Введем пароль для sudo'ера (юзера, под которым мы вошли):
```
[sudo] password for team: 
```
и необходимую информацию. Теперь с этим же юзером и будем работать (когда не нужны права sudo):



#### Локальные соединения

Пропишем необходимые нам конфиги, для начала настроим евхаристическое общение между узлами. Пропишем alias для хостов: 

При помощи команды

```
sudo nano /etc/hosts
```

внесем в файл следующий текст: 

```
192.168.1.182 team-45-jn
192.168.1.183 team-45-nn namenode
192.168.1.184 team-45-dn-0
192.168.1.185 team-45-dn-1
176.109.81.245 public-address
```
Удалим оттуда текст вида:

```
127.0.1.1 team-45-nn
127.0.0.1 localhost
```

А также закомменчиваем IPv6-соединения (если они прописаны - потом может чудить postgres и коннектится через IPv6, придется дополнительно прописывать параметры соединения.

Итоговый файл выглядит вот так: 

```
127.0.0.1 localhost
192.168.1.182 team-45-jn
192.168.1.183 team-45-nn namenode
192.168.1.184 team-45-dn-0
192.168.1.185 team-45-dn-1
176.109.81.245 public-address

# The following lines are desirable for IPv6 capable hosts
#::1     ip6-localhost ip6-loopback
#fe00::0 ip6-localnet
#ff00::0 ip6-mcastprefix
#ff02::1 ip6-allnodes
#ff02::2 ip6-allrouters
```

Распространим файл hosts на все машины.Также создадим везде пользователя hadoop. Надежнее будет сделать это вручную. Также выпишем все созданные публичные ssh-ключи и добавим их впоследствии в конфиг.

```
ssh team@team-45-nn
[ВВОДИМ ПАРОЛЬ team]
sudo nano /etc/hosts
[ВВОДИМ ТЕКСТ ВЫШЕ]
sudo adduser hadoop
sudo -i -u hadoop
ssh-keygen
cat /home/hadoop/.ssh/id_ed25519.pub

ssh team@team-45-dn-0
[ВВОДИМ ПАРОЛЬ team]
sudo nano /etc/hosts
[ВВОДИМ ТЕКСТ ВЫШЕ]
sudo adduser hadoop
sudo -i -u hadoop
ssh-keygen
cat /home/hadoop/.ssh/id_ed25519.pub

ssh team@team-45-dn-1
[ВВОДИМ ПАРОЛЬ team]
sudo nano /etc/hosts
[ВВОДИМ ТЕКСТ ВЫШЕ]
sudo adduser hadoop
sudo -i -u hadoop
ssh-keygen
```

Для справки, ключики получились такие:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILcefr8jgiaNl6qOZ02j2YCnnBnok+oh7hP7spQ5T1J7 hadoop@team-45-jn

ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIvbvvtsf1/d90ft1ylgLnRHQKMjJrEQVTaoJH1GFIfN hadoop@team-45-nn

ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFwr44HZhTbnqV/6WGN74m6SEzSJUPyhuVAwGvcQ9vVg hadoop@team-45-dn-00

ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOhVPIhOmtQtMmS0d4I3xZTX1nIlUlcPv2K9ZBBofxlp hadoop@team-45-dn-01
```

После этого на одной из машин войдем на юзера хадуп. Пока для этого надо (если мы, например, на JumpNode под пользователем team) выполнить команды:

```

ssh [имя ноды из файла /etc/hosts/]
sudo -i -u hadoop
nano .ssh/authorized_keys
[ВВОДИМ ТЕКСТ С КЛЮЧАМИ ВЫШЕ И СОХРАНЯЕМ]

```


и откопируем его на все наши ноды (включая localhost): 

```
scp ~/.ssh/authorized_keys hadoop@team-45-nn:/home/hadoop/.ssh/
scp ~/.ssh/authorized_keys hadoop@team-45-dn-0:/home/hadoop/.ssh/
scp ~/.ssh/authorized_keys hadoop@team-45-dn-1:/home/hadoop/.ssh/
```

**ОПЦИОНАЛЬНО**, после этого можно проверить соединение под 


```
ssh hadoop@team-45-nn
ssh hadoop@team-45-dn-0
ssh hadoop@team-45-dn-1
[На последней ноде]
ssh hadoop@team-45-jn
```

#### Переменные окружения 

Зайдем в окружение .bashrc и пропишем необходимые конфиги: 

```
nano ~/.profile
```

Текст для вставки в файл: 
```
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64/bin/java
export HADOOP_HOME=/home/hadoop/hadoop-3.4.0
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
export HADOOP_MAPRED_HOME=$HADOOP_HOME
```
Активируем окружение: 
```
source ~/.profile
```


Затем раскопируем конфиг на все остальные ноды (предположим, что мы сейчас на NameNode - если нет, нужно подставить нужные имена из /etc/hosts/):

```
scp ~/.profile hadoop@team-45-jn:/home/hadoop/.profile
scp ~/.profile hadoop@team-45-dn-0:/home/hadoop/.profile
scp ~/.profile hadoop@team-45-dn-1:/home/hadoop/.profile
```
Зайдем на каждую из нод и также активируем профайл:

```
ssh hadoop@team-45-jn
source ~/.profile

ssh hadoop@team-45-dn-0
source ~/.profile

ssh hadoop@team-45-dn-1
source ~/.profile
```

### Скачаем необходимые дистрибутивы (Hadoop)

Запустим в tmux (предустановленный менеджер терминалов) скачивание дистрибутивов. Это можно сделать на любом из этапов (параллельно, при помощи tmux) или просто в терминале

```
tmux 

wget https://archive.apache.org/dist/hadoop/common/hadoop-3.4.0/hadoop-3.4.0.tar.gz
```
В нашем случае, мы скачаем дистрибутив с нашей локальной машины (из директории, где лежат и SSH-ключ, и дистрибутив Hadoop) при помощи: 
```
scp hadoop-3.4.0.tar.gz team@176.109.81.245
```
Если уже настроен беспарольный SSH, выполним: 

```
scp hadoop-3.4.0.tar.gz team-45-nn:/home/hadoop/
scp hadoop-3.4.0.tar.gz team-45-dn-0:/home/hadoop/
scp hadoop-3.4.0.tar.gz team-45-dn-1:/home/hadoop/
```


И для каждой ноды: 

``` 
#Подключаемся к ноде 
ssh [имя ноды]

#Распаковываем архив
tar -zvxf hadoop-3.4.0-src.tar.gz
```



### Конфигурация самого Hadoop

зайдем на NameNode (если не подключена - подключимся командой ```ssh namenode```)
и выполним следующие манипуляции:

```
nano $HADOOP_HOME/etc/hadoop/workers
```

внесем в открывшийся файл следующие записи:
 
```
192.168.1.183
192.168.1.184
192.168.1.185
```

Это и будут адреса наших DataNode, на которых мы разворачиваем DFS. Расшарим (по умолчанию - с NameNode) данный файл на все прочие хосты

```
scp $HADOOP_HOME/etc/hadoop/workers team-45-jn:$HADOOP_HOME/etc/hadoop/workers
scp $HADOOP_HOME/etc/hadoop/workers team-45-dn-0:$HADOOP_HOME/etc/hadoop/workers
scp $HADOOP_HOME/etc/hadoop/workers team-45-dn-1:$HADOOP_HOME/etc/hadoop/workers
```

Затем настроим переменные окружения, локальные для самого хадуп.

```
cd $HADOOP_HOME/etc/hadoop/
nano hadoop-env.sh
```

Пропишем в файлике: 

```
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64/bin/java
export HADOOP_MAPRED_HOME=/home/hadoop/hadoop-3.4.0
```

```
cd $HADOOP_HOME/etc/hadoop/
nano mapred-env.sh
```

Пропишем в файлике: 

```
export HADOOP_HOME=/home/hadoop/hadoop-3.4.0

```
Расшарим конфиг на все остальные хосты:

```
cd $HADOOP_HOME/etc/hadoop/

scp $HADOOP_HOME/etc/hadoop/hadoop-env.sh team-45-jn:/home/hadoop/hadoop-3.4.0/etc/hadoop/hadoop-env.sh
scp $HADOOP_HOME/etc/hadoop/hadoop-env.sh team-45-dn-0:/home/hadoop/hadoop-3.4.0/etc/hadoop/hadoop-env.sh
scp $HADOOP_HOME/etc/hadoop/hadoop-env.sh team-45-dn-1:/home/hadoop/hadoop-3.4.0/etc/hadoop/hadoop-env.sh

```
В директории $HADOOP_HOME/etc/hadoop/ пропишем необходимые конфиги: 

**в core-site.xml** - ``` nano core-site.xml```

```
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://192.168.1.183:9000</value>
    </property>
</configuration>
```

**в hdfs-site.xml** - ``` nano hdfs-site.xml```

```
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>

<property>
        <name>dfs.namenode.rpc-address</name>
        <value>team-45-nn:9000</value>
        <description>IP-адрес и порт для RPC-соединений NameNode.</description>
    </property>

    <!-- Адрес HTTP веб-интерфейса NameNode -->
    <property>
        <name>dfs.namenode.http-address</name>
        <value>team-45-nn:9870</value>
        <description>IP-адрес и порт для веб-интерфейса NameNode.</description>
    </property>

    <!-- Адрес HTTP веб-интерфейса DataNode -->
    <property>
        <name>dfs.datanode.http.address</name>
        <value>0.0.0.0:50100</value>
        <description>IP-адрес и порт для веб-интерфейса DataNode.</description>
    </property>
</configuration>
```

Расшариваем все конфиги (скопом на остальные ноды): 

```
cd $HADOOP_HOME/etc/hadoop/

scp *.xml team-45-jn:/home/hadoop/hadoop-3.4.0/etc/hadoop/
scp *.xml team-45-dn-0:/home/hadoop/hadoop-3.4.0/etc/hadoop/
scp *.xml team-45-dn-1:/home/hadoop/hadoop-3.4.0/etc/hadoop/
```

Теперь можем запустить хосты и заняться веб-интерфейсами (публикацией их через джамп ноду. 

Запустим хадуп скриптом starf-dfs.sh. Если .profile - правильный, скрипт должен запуститься из любой директории. Для совместимости можем перейти в директорию, но сначала отфоматируем конфиги. 

Без этого, например, не поднимутся веб-интерфейсы т.к. не будет создано хранилище метаданных и NameNode будет падать в ошибку при попытке к ним обратитс, проверено.

```
bin/hdfs namenode -format
```

```
cd $HADOOP_HOME/sbin
start-dfs.sh
```

Проверим процессы посредством jps, ожидаемый вывод на NameNode:

```
20880 DataNode
21076 SecondaryNameNode
21194 Jps
```

На Datanode0 и Datanode1:

```
10643 Jps
10527 DataNode
```

На JumpNode:
```
39283 Jps
```

Ожидаемые вывод для веб-интерфейсов (нам интересен порт 9870):

```
ss -tulnp | grep 9870
tcp   LISTEN 0      500    192.168.1.183:9870       0.0.0.0:*    users:(("java",pid=27532,fd=335))
```
### Конфигурация Nginx для интерфейсов HDFS (как реверс-прокси для SSH-тоннеля)

Перейдем на машину JumpNode под sudoer'ом team. Настроим проброс портов для веб-интерфейсов с авторизацией по логину и паролю (в личке у С.): 

Скопируем дефолтный конфиг в качестве шаблоны и пропишем туда авторизацию и проброс портов.
~~
```
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/nn
```
Нам потребуется также установить пароль. Сделаем это при помощи утилиты htpasswd

```
sudo apt update 
sudo apt install apache2-utils

sudo htpasswd -c /etc/nginx/.htpasswd team45
[Вводим пароль для пользователя team45, он в личке у Саттара]
```
Пропишем конфиг
Создадим символическу линку в директории sites-enabled на наш конфиг:

```
cd /etc/nginx/sites-available/

sudo nano nn

sudo ln -s /etc/nginx/sites-available/nn /etc/nginx/sites-enabled/nn
```

Пропишем туда такой конфиг:

```

server {
	listen 9870 default_server;
	server_name nn;
	location / {
		# First attempt to serve request as file, then
		# as directory, then fall back to displaying a 404.
		proxy_pass http://team-45-nn:9870;
		auth_basic "Restricted Access";
		auth_basic_user_file /etc/nginx/.htpasswd;
	}

}

```
Потом займемся также WebUI для каждой из DataNode:
```
Потом для **DataNode0 (NN)**:

```
sudo cp nn dn0
sudo ln -s /etc/nginx/sites-available/dn0 /etc/nginx/sites-enabled/dn0
sudo nano dn0

```

Прописываем туда такое:

```
server {
        listen 50100 default_server;



        server_name dn0;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                proxy_pass http://team-45-nn:50100;
                auth_basic "Restricted Access";
                auth_basic_user_file /etc/nginx/.htpasswd;
        }

}
```
Потом для **DataNode1 (DN-00)**:

```
sudo cp dn0 dn1
sudo ln -s /etc/nginx/sites-available/dn1 /etc/nginx/sites-enabled/dn1
sudo nano dn1

```

Прописываем туда такое:

```
server {
        listen 50101 default_server;



        server_name dn1;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                proxy_pass http://team-45-dn-0:50100;
                auth_basic "Restricted Access";
                auth_basic_user_file /etc/nginx/.htpasswd;
        }

}
```

Потом для **DataNode2 (DN-01)**:

```
sudo cp dn1 dn2
sudo ln -s /etc/nginx/sites-available/dn2 /etc/nginx/sites-enabled/dn2
sudo nano dn2

```

Прописываем туда такое:

```
server {
        listen 50101 default_server;



        server_name dn2;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                proxy_pass http://team-45-dn-0:50100;
                auth_basic "Restricted Access";
                auth_basic_user_file /etc/nginx/.htpasswd;
        }

}
```


И наконец перезапустим Nginx с новыми конфигами.

```
sudo systemctl restart nginx
```

### SSH-туннелирование для просмотра интерфейсов

Настроим через JumpNode туннель до NameNode и интерфейсов DataNode. Создадим на локальной машине новый ключик и подключимся по паролю к пользователю team:
```
ssh-keygen
ssh-copy-id team@176.109.81.245
```

На локально компьютере необходимо добавить в ~/.ssh/config:

```
nano ~/.ssh/config
```
следующие строки:
 
```
#Настройка для jumpnode
Host jumpnode
    HostName 176.109.81.245
    User team
    ForwardAgent yes
    #WebUI HistoryServer
    LocalForward 19888 127.0.0.1:19888
    #WebUI YarnResource Manager
    LocalForward 8088 127.0.0.1:8088
    #WebUI NameNode
    LocalForward 9870 127.0.0.1:9870
    #WebUI DataNode (сразу все
    LocalForward 50100 127.0.0.1:50100
    LocalForward 50101 127.0.0.1:50101
    LocalForward 50102 127.0.0.1:50102
    #WevUI NodeManager'ов
    LocalForward 8050 127.0.0.1:8050
    LocalForward 8051 127.0.0.1:8051
    LocalForward 8052 127.0.0.1:8052
    
   # Раскомментим потом
    

#Настройка для namenode через jumpnode
Host namenode
    HostName 192.168.1.183
    User hadoop
    ProxyJump jumpnode

#Настройка для datanode через jumpnode
Host datanode0
    HostName 192.168.1.184
    User hadoop
    ProxyJump jumpnode
    
Host datanode1
    HostName 192.168.1.185
    User hadoop
    ProxyJump jumpnode
    
``` 

На локальном компьютере выполним: 

```
cat ~/.ssh/id_rsa.pub
```

И добавим этот ключик в авторизованные ключи для пользователя team и hadoop на всех нодах.

Например, под юзером hadoop на jumpnode:

```
nano ~/.ssh/authorized_keys
[пишем ключ сюда]
```

и размножим на всех машинах: 

```
scp ~/.ssh/authorized_keys hadoop@team-45-nn:/home/hadoop/.ssh/
scp ~/.ssh/authorized_keys hadoop@team-45-dn-0:/home/hadoop/.ssh/
scp ~/.ssh/authorized_keys hadoop@team-45-dn-1:/home/hadoop/.ssh/
```

Затем на машине jn (пользователь team) нужно поменять настройки ssh-сервера:

```
sudo nano /etc/ssh/sshd_config
```

[ПРОПИСАТЬ]

```
AllowTcpForwarding yes
GatewayPorts yes
```
И перезапустить сервис:

```
sudo service ssh restart
```

и разможить данный конфиг на все остальные машины (пользователи team и hadoop его шерят), это придется сделать руками. 


Если по каким-то причинам прямое туннелирование не сработало, мы всегда можем использовать Nginx как реверс-прокси, и автоматически пробрасывать порты при подключении к jumpnode.



Заходим у себя на локальной машине на окно авторизации (которое выдает нам Nginx) и наслаждаемся. Теперь при любом ssh подключении к JumpNode она автоматом прокинет заданные порты на нашу локальную машину. 

```
localhost:9870
```

## 2. Установка и раскатка YARN

На NameNode'е выполним: 

```
cd $HADOOP_HOME/etc/hadoop
```

Откроем файлы конфигурации YARN'а и MapReduce. Пропишем в них следующее.

**в mapred-site.xml** - ``` nano mapred-site.xml```

```
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.application.classpath</name>
        <value>/home/hadoop/hadoop-3.4.0/share/hadoop/mapreduce/*:/home/hadoop/hadoop-3.4.0/share/hadoop/mapreduce/lib/*</value>
    </property>
    <property>
        <name>yarn.app.mapreduce.am.env</name>
        <value>HADOOP_MAPRED_HOME=/home/hadoop/hadoop-3.4.0</value>
        <description>Environment variables for the Application Master.</description>
    </property>
    <property>
        <name>mapreduce.map.env</name>
        <value>HADOOP_MAPRED_HOME=/home/hadoop/hadoop-3.4.0</value>
        <description>Environment variables for map tasks.</description>
    </property>
    <property>
        <name>mapreduce.reduce.env</name>
        <value>HADOOP_MAPRED_HOME=/home/hadoop/hadoop-3.4.0</value>
        <description>Environment variables for reduce tasks.</description>
    </property>
    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>team-45-nn:19888</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.done-dir</name>
        <value>/mapreduce/history/done</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.intermediate-done-dir</name>
        <value>/mapreduce/history/intermediate</value>
    </property>
</configuration>
```


**в yarn-site.xml** - ``` nano yarn-site.xml```:

```
<configuration>
<property>
    <name>yarn.resourcemanager.hostname</name>
    <value>team-45-nn</value>
</property>>
<property>
    <name>yarn.nodemanager.env-whitelist</name>
    <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value>
</property>
<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
</property>
<property>
    <name>yarn.nodemanager.webapp.https.address</name>
    <value>0.0.0.0:8090</value>
    <description>Адрес и порт для HTTPS веб-интерфейса NodeManager</description>
</property>
<!-- Site specific YARN configuration properties -->

</configuration>

```

Расшариваем все конфиги (скопом на остальные ноды): 

```
cd $HADOOP_HOME/etc/hadoop/

scp *.xml team-45-jn:/home/hadoop/hadoop-3.4.0/etc/hadoop/
scp *.xml team-45-dn-0:/home/hadoop/hadoop-3.4.0/etc/hadoop/
scp *.xml team-45-dn-1:/home/hadoop/hadoop-3.4.0/etc/hadoop/

scp hadoop-env.sh team-45-jn:/home/hadoop/hadoop-3.4.0/etc/hadoop/
scp hadoop-env.sh team-45-dn-0:/home/hadoop/hadoop-3.4.0/etc/hadoop/
scp hadoop-env.sh team-45-dn-1:/home/hadoop/hadoop-3.4.0/etc/hadoop/

scp mapred-env.sh team-45-jn:/home/hadoop/hadoop-3.4.0/etc/hadoop/
scp mapred-env.sh team-45-dn-0:/home/hadoop/hadoop-3.4.0/etc/hadoop/
scp mapred-env.sh team-45-dn-1:/home/hadoop/hadoop-3.4.0/etc/hadoop/
```
Запускаем YARN:

```
start-yarn.sh
```

и WebHistoryServer

```
mapred --daemon start historyserver
```
Посмотрим на выход ```ss -tulnp```:

```
Netid State  Recv-Q  Send-Q   Local Address:Port    Peer Address:Port Process                            
udp   UNCONN 0       0           127.0.0.54:53           0.0.0.0:*                                       
udp   UNCONN 0       0        127.0.0.53%lo:53           0.0.0.0:*                                       
tcp   LISTEN 0       500      192.168.1.183:9870         0.0.0.0:*     users:(("java",pid=27532,fd=335)) 
tcp   LISTEN 0       256      192.168.1.183:8033         0.0.0.0:*     users:(("java",pid=30302,fd=372)) 
tcp   LISTEN 0       256      192.168.1.183:8032         0.0.0.0:*     users:(("java",pid=30302,fd=402)) 
tcp   LISTEN 0       4096        127.0.0.54:53           0.0.0.0:*                                       
tcp   LISTEN 0       500          127.0.0.1:42837        0.0.0.0:*     users:(("java",pid=27715,fd=339)) 
tcp   LISTEN 0       256      192.168.1.183:8031         0.0.0.0:*     users:(("java",pid=30302,fd=382)) 
tcp   LISTEN 0       256      192.168.1.183:8030         0.0.0.0:*     users:(("java",pid=30302,fd=392)) 
tcp   LISTEN 0       500      192.168.1.183:8088         0.0.0.0:*     users:(("java",pid=30302,fd=362)) 
tcp   LISTEN 0       4096     127.0.0.53%lo:53           0.0.0.0:*                                       
tcp   LISTEN 0       511            0.0.0.0:80           0.0.0.0:*                                       
tcp   LISTEN 0       500      192.168.1.183:19888        0.0.0.0:*     users:(("java",pid=30852,fd=351)) 
tcp   LISTEN 0       256            0.0.0.0:8040         0.0.0.0:*     users:(("java",pid=30464,fd=401)) 
tcp   LISTEN 0       500            0.0.0.0:8042         0.0.0.0:*     users:(("java",pid=30464,fd=412)) 
tcp   LISTEN 0       256            0.0.0.0:10033        0.0.0.0:*     users:(("java",pid=30852,fd=340)) 
tcp   LISTEN 0       256            0.0.0.0:10020        0.0.0.0:*     users:(("java",pid=30852,fd=359)) 
tcp   LISTEN 0       256      192.168.1.183:9000         0.0.0.0:*     users:(("java",pid=27532,fd=346)) 
tcp   LISTEN 0       4096           0.0.0.0:9864         0.0.0.0:*     users:(("java",pid=27715,fd=392)) 
tcp   LISTEN 0       256            0.0.0.0:9866         0.0.0.0:*     users:(("java",pid=27715,fd=336)) 
tcp   LISTEN 0       256            0.0.0.0:9867         0.0.0.0:*     users:(("java",pid=27715,fd=393)) 
tcp   LISTEN 0       500            0.0.0.0:9868         0.0.0.0:*     users:(("java",pid=27927,fd=337)) 
tcp   LISTEN 0       256            0.0.0.0:44379        0.0.0.0:*     users:(("java",pid=30464,fd=390)) 
tcp   LISTEN 0       128            0.0.0.0:13562        0.0.0.0:*     users:(("java",pid=30464,fd=411)) 
tcp   LISTEN 0       511               [::]:80              [::]:*                                       
tcp   LISTEN 0       4096                 *:22                 *:*                                       
```

Видим, что демоны торчат на портах :8088 - **ResourceManager WebUI**, на :19888 - WebUI **HistoryManager WebUI**, на датанодах видно, что работают **NodeManager'ы** - они доступны на портах :


Затем открываем веб-интерфейсы при помощи Nginx:

Для ResouceManager'а: 

```
ssh jumpnode
su team
cd /etc/nginx/sites-available
sudo cp nn ya
sudo ln -s /etc/nginx/sites-available/ya /etc/nginx/sites-enabled/ya
sudo nano ya

```
Прописываем туда такое:

```
server {
        listen 8088 default_server;



        server_name ya;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                proxy_pass http://team-45-nn:8088;
                auth_basic "Restricted Access";
                auth_basic_user_file /etc/nginx/.htpasswd;
        }

}
```

Потом для WebHistoryServer'а:

```
sudo cp nn dh
sudo ln -s /etc/nginx/sites-available/dh /etc/nginx/sites-enabled/dh
sudo nano dh

```

Прописываем туда такое:

```
server {
        listen 19888 default_server;



        server_name dh;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                proxy_pass http://team-45-nn:19888;
                auth_basic "Restricted Access";
                auth_basic_user_file /etc/nginx/.htpasswd;
        }

}
```

Потом для NodeManager'ов:

```
sudo cp nn nm0
sudo ln -s /etc/nginx/sites-available/nm0 /etc/nginx/sites-enabled/nm0
sudo nano nm0

```

Прописываем туда такое:

```
server {
        listen 8050 default_server;



        server_name nm0;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                proxy_pass http://team-45-nn:8050;
                auth_basic "Restricted Access";
                auth_basic_user_file /etc/nginx/.htpasswd;
        }

}
```

Потом для NodeManager'ов:

```
sudo cp nn nm1
sudo ln -s /etc/nginx/sites-available/nm1 /etc/nginx/sites-enabled/nm1
sudo nano nm1

```

Прописываем туда такое:

```
server {
        listen 8051 default_server;



        server_name nm1;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                proxy_pass http://team-45-dn-0:8050;
                auth_basic "Restricted Access";
                auth_basic_user_file /etc/nginx/.htpasswd;
        }

}
```
Потом для NodeManager'ов:

```
sudo cp nm1 nm2
sudo ln -s /etc/nginx/sites-available/nm2 /etc/nginx/sites-enabled/nm2
sudo nano nm2

```

Прописываем туда такое:

```
server {
        listen 8052 default_server;



        server_name nm2;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                proxy_pass http://team-45-dn-1:8050;
                auth_basic "Restricted Access";
                auth_basic_user_file /etc/nginx/.htpasswd;
        }

}
```

После этого перезапускаем Nginx и наше ssh подключение с локальной машины. Интерфейсы будут доступны на соответствующих портах.

```
sudo systemctl nginx
```

# 3. Разворачиваем Hive и партицируем таблицы:

Развернем postgres на NameNode'е. 

```
ssh namenode
```
Работать будем из-под пользователя team.
```
su team
```
Установим postgres:
```
sudo apt install postgresql
```

Создадим и установим права на БД для MetaStore:

```
sudo -i -u postgres
```

Далее в консоли:

```
psql

CREATE USER hive with password 'hivepass';
CREATE DATABASE metastore WITH owner hive;
#Иногда, пользователь может падать в ошибку т.к. не имеет доступа к схеме public. Пропишем это отдельно 
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO hive;
GRANT ALL PRIVILEGES ON DATABASE "metastore" to hive;
\q 
exit
#выходим из консоли
```

Установим сетевой доступ к БД:

```
sudo nano /etc/postgresql/16/main/postgresql.conf
```

Пропишем там следующие конфиги: 

```
listen_addresses = 'team-45-nn'

```
а также, 

```
sudo nano /etc/postgresql/16/main/pg_hba.conf
```
там пропишем в разделе IPv4 connections:
```
host		metastore		hive		192.168.1.182/32 password
```
Перезапустим Postgres с обновленными конфигами:

```
sudo systemctl restart postgresql
```

Можно также протестировать подключение (с JumpNode и NameNode):

```
ssh team-45-jn
sudo apt install postgresql-client-16
psql -h team-45-nn -p 5432 -U hive -W -d metastore
# Выйдем из консоли 
\q
```
Переключимся на пользователя hadoop

```
sudo -i -u hadoop
wget https://dlcdn.apache.org/hive/hive-4.0.1/apache-hive-4.0.1-bin.tar.gz
tar -xvzf apache-hive-4.0.1-bin.tar.gz
```
Далее перейдем к настройке Hive:


```
cd apache-hive-4.0.1-bin
#Установим драйвер postgres
cd ./lib
wget https://jdbc.postgresql.org/download/postgresql-42.7.4.jar
```

Пропишем конфиги самого Hive:

```
nano hive-site.xml
```

В **hive-site.xml** пропишем:

```
<configuration>
  <property>
    <name>hive.server2.authentication</name>
    <value>NONE</value>
    <description>Authentication mode for HiveServer2</description>
  </property>
   <property>
    <name>hive.server2.thrift.bind.host</name>
    <value>0.0.0.0</value> <!-- Или используйте 0.0.0.0 для всех интерфейсов -->
    <description>Hostname to bind HiveServer2 Thrift service.</description>
  </property>
    <property>
    <name>hive.server2.thrift.port</name>
    <value>5433</value>
  </property>
  <property>
    <name>hive.metastore.warehouse.dir</name>
    <value>/user/hive/warehouse</value>
  </property>
  <property>
  <name>hive.metastore.port</name>
  <value>9084</value> <!-- Choose an alternative port -->
  <description>Port number for the Hive Metastore Server</description>
</property>
  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>hive</value>
    <description>Username to use against metastore database</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>hivepass</value>
    <description>password to use against metastore database</description>
  </property>
  
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:postgresql://team-45-nn:5432/metastore</value>
  </property>
<property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>org.postgresql.Driver</value>
  </property>
<property>
  <name>hive.server2.webui.enabled</name>
  <value>true</value>
  <description>Enable the Hive Web UI</description>
</property>

<property>
  <name>hive.server2.webui.port</name>
  <value>12344</value>
  <description>Port for the Hive Web UI</description>
</property>

<property>
  <name>hive.server2.webui.bind.host</name>
  <value>0.0.0.0</value> 
</property>
</configuration>
```

Переместим созданный файлик:
```
mv hive-site.xml ~/apache-hive-4.0.1-bin/conf/
```

Также изменим **.profile** для нашего Hive:

```
nano ~/.profile
[ПРОПИШЕМ]
export HIVE_HOME=/home/hadoop/apache-hive-4.0.1-bin
export HIVE_CONF_DIR=$HIVE_HOME/conf
export HIVE_AUX_JARS_PATH=$HIVE_HOME/lib/*
export PATH=$PATH:$HIVE_HOME/bin
```
Активируем профиль:
```
source ~/.profile
```
После этого настроим подключение с нашей локальной машины. Настроим авторизацию в nginx на JumpNode и добавим строчку в конфиг хоста jumpnode на нашей **ЛОКАЛЬНОЙ МАШИНЕ**.

```
su team
cd /etc/nginx/sites-available/
sudo cp dn0 hv
sudo ln -s /etc/nginx/sites-available/hv /etc/nginx/sites-enabled/hv
sudo nano hv
[ПИШЕМ ДАННЫЕ]
sudo systemctl restart nginx
```
Прописываем туда такое:

```
server {
        listen 12345 default_server;



        server_name hv;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                proxy_pass http://127.0.0.1:12344;
                auth_basic "Restricted Access";
                auth_basic_user_file /etc/nginx/.htpasswd;
        }

}
```
А также на всякий случай поменяем число worker-connections в конфигах nginx до 2048:

```
sudo nano /etc/nginx/nginx.conf
[ПРОПИСЫВАЕМ]
worker_rlimit_nofile 100000;
events {
        worker_connections 2048;
        # multi_accept on;
}

sudo nano /etc/security/limits.conf

[ПРОПИСЫВАЕМ]
www-data soft nofile 100000
www-data hard nofile 100000
```

На локальной машине выполним:

```
nano ~/.ssh/config
```
Прописываем туда:
```
Host jumpnode
    HostName 176.109.81.245
    User team
    ForwardAgent yes
    #WebUI HistoryServer
    LocalForward 19888 127.0.0.1:19888
    #WebUI YarnResource Manager
    LocalForward 8088 127.0.0.1:8088
    #WebUI NameNode
    LocalForward 9870 127.0.0.1:9870
    #WebUI DataNode (сразу все
    LocalForward 50100 127.0.0.1:50100
    LocalForward 50101 127.0.0.1:50101
    LocalForward 50102 127.0.0.1:50102
    #WevUI NodeManager'ов
    LocalForward 8050 127.0.0.1:8050
    LocalForward 8051 127.0.0.1:8051
    LocalForward 8052 127.0.0.1:8052
    #HiveUI
    LocalForward 12345 127.0.0.1:12345
```
Вернемся в пользователя hadoop

```
sudo -i -u hadoop
```
Создадим прописанную в конфигах директорию на HDFS:

```
hdfs dfs -mkdir -p /user/hive/warehouse
hdfs dfs -mkdir -p /tmp
```
Пропишем права для Hive:
```
hdfs dfs -chmod g+w /tmp
hdfs dfs -chmod g+w /user/hive
```

Запустим Hive:

```
cd $HIVE_HOME
./bin/schematool -dbType postgres -initSchema
hive --hiveconf hive.server2.enable.doAs=false \
     --hiveconf hive.security.authorization.enabled=false \
     --hiveconf hive.server2.thrift.bind.host=0.0.0.0 \
     --hiveconf hive.server2.thrift.port=5433 \
     --service hiveserver2 1>> /tmp/hs2.log 2>> /tmp/hs2.log &
```

В выводе jps должен появиться:

```
hadoop@team-45-jn:~/apache-hive-4.0.1-bin$ jps
8165 Jps
48122 RunJar
[1] 48060
```
Подключимся к консоли Hive и начнем работать.
```
beeline -u jdbc:hive2://localhost:5433/default -n hive -p hivepass hivepass
```

Создадим необходимую БД (в консоли beeline): 

```
CREATE DATABASE prod;
[ВЫЙДЕМ ИЗ КОНСОЛИ]
[CTRL+C]
```

Создадим директорию для нашего датасета:

```
hdfs dfs -mkdir /input
wget https://raw.githubusercontent.com/ADBondarenko/BigDataProjectOPS/refs/heads/main/housing.csv
hdfs dfs -put housing.csv /input

beeline -u jdbc:hive2://localhost:5433/prod -n hive -p hivepass
```

Установим настройки партицирования:

```
SET hive.exec.dynamic.partition = true;
SET hive.exec.dynamic.partition.mode = nonstrict;
```

```
CREATE EXTERNAL TABLE california_housing_partitioned_parquet (
    longitude DOUBLE,
    latitude DOUBLE,
    housing_median_age INT,
    total_rooms INT,
    total_bedrooms INT,
    population INT,
    households INT,
    median_income DOUBLE,
    median_house_value DOUBLE
)
PARTITIONED BY (income_partition STRING)
STORED AS PARQUET
LOCATION '/user/hive/warehouse/california_housing_partitioned_parquet/';
```

Соаздадим и выгрузим исходную таблицу из .csv:

```
CREATE TABLE california_housing_raw (
    longitude DOUBLE,
    latitude DOUBLE,
    housing_median_age INT,
    total_rooms INT,
    total_bedrooms INT,
    population INT,
    households INT,
    median_income DOUBLE,
    median_house_value DOUBLE,
    income_partition STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;
```

выгрузим: 

```
LOAD DATA INPATH '/input/housing.csv' INTO TABLE california_housing_raw;
```

Выгрузим с партицией по доходу через CASE WHEN:

```
INSERT INTO TABLE california_housing_partitioned_parquet
PARTITION (income_partition)
SELECT 
    longitude,
    latitude,
    housing_median_age,
    total_rooms,
    total_bedrooms,
    population,
    households,
    median_income,
    median_house_value,
    CASE
        WHEN median_income < 2.5 THEN 'low'
        WHEN median_income < 4.5 THEN 'medium'
        ELSE 'high'
    END AS income_partition
FROM california_housing_raw;
```

Посмотрим на то, верно ли были созданы партиции:


```
SHOW PARTITIONS california_housing_partitioned_parquet;

SELECT * FROM california_housing_partitioned_parquet LIMIT 10;
```

Ура, мы восхитительны: 

```
INFO  : Completed executing command(queryId=hadoop_20241104154228_18455f37-280c-469b-9e9d-df8a26253555); Time taken: 0.028 seconds
+--------------------------+
|        partition         |
+--------------------------+
| income_partition=high    |
| income_partition=low     |
| income_partition=medium  |
+--------------------------+
3 rows selected (0.127 seconds)
```

Done.
