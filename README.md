# urfu

for test

**1**.На server2 добавляем vhost с проксированием к server1.На server1 запускаем backend приложение, пример привёл в папке node.js
```
$ node app.js
```
Запускаем nginx на server1 для работы со статикой.
Запросы поступающие на прокси-сервер, с указанием определённого доменного имени, будут перенаправляться
на server1 по соотвествующим URI.

Примеры конфигурационных файлов приведены в папке 1.
Если использовать кастомные сабдомены, то достаточно перенаправлять только 80ый порт(или 443 для https) из DMZ.

Так же всё можно развернуть используя Docker. Файлы конфигурация приведены в папках проектов. Конфигурационные файл Nginx копируется внуть образа, как и статический контент. Так же приведён docker-compose.

**2**.На Jenkins настраивается job, который мониторит определённую ветку в git(vcs). Устанавливается триггер срабатывающий при коммите. Job забирает исходные файлы с git, копирует их через SSH на нужный сервер. В момент копирования, файлы могут быть недоступны по запросам от пользователей. Пример Jenkinsfile для pipeline привёл в папке 2.

**3**.SQL Server должен использовать учётную запись с правами доступа к сетевой директории '\\server2\rk\'.
Через T-SQL запрос
```
BACKUP DATABASE database.
   TO DISK = '\\server2\rk\database.bak';  
GO
```
#Проверяем на наличие ошибок при передаче через сеть
```
   RESTORE VERIFYONLY FROM DISK = '\\server2\rk\database.bak';
GO  
```
Или через Managment Studio
Задачи ---> Создать резервную копию

**4**.Полный бэкап системы 150 Гб раз в 10 дней. Разностная копия 500 Мб за каждый день. Используется подход смешанного разностного копирования. Создаются цепочки разностных копий, из условия задачи их будет 10, по одной на каждый день. Первоначальная цепочка будет занимать порядка 177.5 Гб + ~0,5% за каждый день, что в сумме даст ~187Гб. При замене с удалением предыдущей цепочки на новую получается приращение ~ 10Гб за период в 10 дней. С учётом первоначального обёма резервных копий получается 520 дней. Так как расчёты имеют приблизительный характер, то лучше взять большую погрешность вычислений в 10%, что в итоге составит 468 дней.

**5**.Возможно развернуть на обширном семействе ОС Linux(Ubuntu, CentOS, ...) Потребуются, собственно, php 7.3 и MariaDB 10.5.1. Часто входят в состав дистрибутивов. Устанавливаются в зависимости от пакетного менеджера данной ОС. Для RHEL семейства требуется добавление репозитория Remi. Пример привёл в папке 5.
``` 
rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
dnf install -y https://rpms.remirepo.net/enterprise/remi-release-8.rpm
dnf module enable php:remi-7.3 -y
dnf install -y php php-cli php-common
#Поддержка MySQL в php
dnf install -y php-mysqlnd
```

Альтернативный способ разворачивания приложения через Docker. Примеры конфигураций привёл в GitHub.

**6**.Нужно создать точку монтирования с использованием протокола CIFS
```
apt-get install cifs-utils
```
#Монтируем
```
mount.cifs //RK/db /mnt -o user=mysql_bkp, pass=mypass
```
#Используем утилиту mysqldump
```
mysqldump -h localhost -u root -p mydb > /mnt/mydb.sql
```

**7**.Для запуска java-приложений требуется установленый пакет OpenJDK( Oracle JDK имее тограничения по лицензии).
#Для Ubuntu, Debian
```
sudo apt-get install openjdk-8-jdk
```
В докере можно воспользоваться готовым образом, пример Dockerfile
```
 FROM openjdk:8-jdk-alpine
 COPY ./ ./
 CMD ["java", "-jar", "app.jar"]
```
**8**.В данном случае лучше разворачивать приложение непосредственно в Docker, т.к. .net не ялвется родной платформой Linux, а сервера Windows обычно не годятся для высоконагруженных систем. В Visual Studio существует поддержка Docker для проектов C#, dockerfile формируется под конкретный проект.

#Пример
```
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build-env
WORKDIR /app

#Copy csproj and restore as distinct layers
COPY *.csproj ./
RUN dotnet restore

#Copy everything else and build
COPY . ./
RUN dotnet publish -c Release -o out

#Build runtime image
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1
WORKDIR /app
COPY --from=build-env /app/out .
ENTRYPOINT ["dotnet", "aspnetapp.dll"]
```
**9**.Самая простая конфигурация балансировщика. По умолчанию используется метод round-robin, нагрузка распределяется равномерно между серверами. Так же можно менять распределение нагрузки изменяя веса серверов или метод балансировки.

```
http {
    upstream myapp1 {

        server srv1.example.com weight=3;
        server srv2.example.com;
        server srv3.example.com;
        server srv4.example.com;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://myapp1;
        }
    }
}
```
**10**.В Debian 10 используется cron, обычно присутствует в дистрибутиве. Редактируется текстовый файл планировщика, добавляю задания вида:
```
0 5,17 * * * /scripts/script.sh
```
Скрипт будет выполняться два раза в день в 5 и 17 часов.

В Windows используется Task Scheduler, задания создаются в графическом интерфейсе. Так же есть командлет New-ScheduledTaskTrigger в PS.
```
# Настройки триггера
$Trigger= New-ScheduledTaskTrigger -At 10:00am –Daily 
# Учётная запись выполняющая задание
$User= "NT AUTHORITY\SYSTEM"  
#Выполняемый скрипт
$Action= New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "C:\PS\StartupScript.ps1" 
# Имя задачи
Register-ScheduledTask -TaskName "CheckingDBbackup" -Trigger $Trigger -User $User -Action $Action -RunLevel Highest –Force 
```