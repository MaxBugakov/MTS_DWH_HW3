# ДЗ 3 - Развертывание и загрузка данных в Hive
Бугаков Максим БПИ226
---
### 1. Подключение к серверу

```bash
ssh team@176.109.91.25
```
---
### 2. Установка и инициализация Postqres
Установка
```bash
ssh tmpl-nn
sudo apt install postgresql
```

---

### 3. Создание нового пользователя.
```bash
sudo adduser postgres
sudo -i -u postgres
```
(пароль от нового пользователя: postgres)

---

### 4. Инициализация БД

Подключается К БД
```bash
psql
```
---
Создание таблицы, пользователя и прав пользователя
```bash
CREATE DATABASE metastore;
CREATE USER hive WITH PASSWORD '123123';
GRANT ALL PRIVILEGES ON DATABASE metastore TO hive;
ALTER DATABASE metastore OWNER TO hive;
```
---
```bash
\q
exit
```

---
### 5. Настройка Postgres на NN 
```bash
sudo vim /etc/postgresql/16/main/postgresql.conf
```
----
Изменяем прослушиваемый адрес и порт
```bash
listen_addresses = 'tmpl-nn'
port = 5433
```
---
```bash
sudo vim /etc/postgresql/16/main/pg_hba.conf
```
---
Добавляем строчки
```bash
host    metastore       hive            192.168.1.1/32          password
host    metastore       hive            192.168.1.94/32         password
```
192.168.1.94/32 - JN

---
Перезагрузка Postqres
```bash
sudo systemctl restart postgresql
sudo systemctl status postgresql
```
--- 

### 6. Установка Postgres на JN 
```bash
ssh tmpl-jn
sudo -i -u hadoop
sudo apt install postgresql-client-16
```

---
### 7. Установка и настройка Hive

Установка и распаковка
```bash
wget https://archive.apache.org/dist/hive/hive-4.0.0-alpha-2/apache-hive-4.0.0-alpha-2-bin.tar.gz
tar -xzvf apache-hive-4.0.0-alpha-2-bin.tar.gz
```
---
Установка Postqres
```bash
cd apache-hive-4.0.0-alpha-2-bin/lib
wget https://jdbc.postgresql.org/download/postgresql-42.7.4.jar
```
---

Настройка Hive
```bash
vim ~/apache-hive-4.0.0-alpha-2-bin/conf/hive-site.xml
```

Добавить следующие строчки
```bash
<configuration> 
    <property> 
        <name>hive.server2.authentication</name> 
        <value>NONE</value> 
    </property> 
    <property> 
        <name>hive.metastore.warehouse.dir</name> 
        <value>/user/hive/warehouse</value> 
    </property> 
    <property> 
        <name>hive.server2.thrift.port</name> 
        <value>5432</value> 
    </property> 
    <property> 
        <name>javax.jdo.option.ConnectionURL</name> 
        <value>jdbc:postgresql://tmpl-nn:5433/metastore</value> 
    </property> 
    <property> 
        <name>javax.jdo.option.ConnectionDriverName</name> 
        <value>org.postgresql.Driver</value> 
    </property> 
    <property> 
        <name>javax.jdo.option.ConnectionUserName</name> 
        <value>hive</value> 
    </property> 
    <property> 
        <name>javax.jdo.option.ConnectionPassword</name> 
        <value>hiveMegaPass</value> 
    </property> 
</configuration> 
```
---

Редакторирование 
```bash
 vim ~/.profile 
```
---
Добавить следующие строчки
```bash
export HIVE_HOME=/home/hadoop/apache-hive-4.0.0-alpha-2-bin 
export HIVE_CONF_DIR=$HIVE_HOME/conf 
export HIVE_AUX_JARS_PATH=$HIVE_HOME/lib/* 
export PATH=$PATH:$HIVE_HOME/bin
```
---
Выолнить
```bash
 source ~/.profile 
```

---
### 8. Настройка Hadoop

Создаем новый каталог
```bash
hdfs dfs -mkdir -p /user/hive/warehouse 
hdfs dfs -chmod g+w /tmp 
hdfs dfs -chmod g+w /user/hive/warehouse
```
---
### 8. Запуск Hive

Старт 
```bash
 cd ~/apache-hive-4.0.0-alpha-2-bin 
bin/schematool -dbType postgres -initSchema
```

---

Запуск 
```bash
nohup hive --service hiveserver2   --hiveconf hive.server2.enable.doAs=false   --hiveconf hive.security.authorization.enabled=false   >> /tmp/hs2.log 2>&1 &
```
---
Подключение 
```bash
 beeline -u jdbc:hive2://tmpl-jn:5432 -n scott -p tiger
```

---
### 9. Загрузка данных в Hive

1. Генерация локального TSV-файла
```bash
cd ~
# Создание файла pairs.tsv (фрукт	цвет)
printf "apple\tred\nbanana\tyellow\ngrape\tpurple\nmelon\tgreen\npear\tgreen\n" > pairs.tsv
```

---

2. Загрузка файла в HDFS

```bash
# Создание дириктории
hdfs dfs -mkdir -p /test

# Копирование файла
hdfs dfs -put -f pairs.tsv /test

# Проверяем содержимое директории /test
hdfs dfs -ls /test
```

---

3. Создание таблицы в Hive
```sql
CREATE DATABASE IF NOT EXISTS test2;

CREATE TABLE IF NOT EXISTS test2.fruit_colors (
  fruit STRING,
  color STRING
)
COMMENT 'Fruit to color mapping'
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY '\t';

USE test2;
```

---

## 4. Загрузка данных и проверка

```sql
## Загрузка данных из HDFS
LOAD DATA INPATH '/test/pairs.tsv' INTO TABLE test2.fruit_colors;

## Вывод двнных![image](https://github.com/user-attachments/assets/1a5ddf96-db07-48a3-9f81-8b54a2f60437)
![image](https://github.com/user-attachments/assets/8bdc765c-a7ef-431b-8582-0f8f1d494653)
![image](https://github.com/user-attachments/assets/ddd0076a-d7d4-4ad3-91f8-4fa6b8dae5d1)

SELECT * FROM test2.fruit_colors LIMIT 5;
```

**Результат:**
![Данные](Снимок экрана 2025-04-25 225140.png)

---









































