# Тестовый стенд через docker-compose и WSL

## WSL, активация, первичные настройки

### Установить windows terminal (можно это и не делать)

[Установить Windows Terminal](https://www.microsoft.com/store/productId/9N0DX20HK701)

Терминал позволяет открывать окна CMD, PowerShell, WSL-дистрибутивов, так что вполне может пригодиться.

### Активация WSL

1. Запустить терминал от имени администратора
   1. Либо правая кнопка и далее так
    ![Terminal elevated right click](./img/terminal_elevated.png)
   2. Либо открыть меню пуск и начать печатать "windowster..." или "ter.." и когда появится Windows Terminal нажать Ctrl+Shift+Enter.
2. Активация WSL: ```dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart```
3. Активация функционала виртуальной машины: ```dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart```.
4. Перезагрузка.
5. Обновить ядро WSL для перехода на WSL2:
   1. Скачать ``` https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi ```
   2. Установить.
6. Установить версию WSL2 по умолчанию.
   1. В терминале: ``` wsl --set-default-version 2 ```
7. Установить Linux distro
   1. Зайти в Microsoft Store
   2. В поиск вбить Linux
   3. Выбрать дистро и установить
8. Посмотрим, что у нас установлено из WSL
   1. В терминале: ```wsl --list --verbose```
   2. Примерный вывод в терминале:

```
    PS C:\Users\username> wsl --list --verbose
    NAME                   STATE           VERSION
    Ubuntu-20.04           Stopped         1
    Ubuntu-18.04           Stopped         2
```
9. Конвертируем дистро первой версии во вторую (Ubuntu-20.04).
   1.  ```wsl --set-version Ubuntu-20.04 2```

## Docker Desktop

[На сайте докера скачиваем](https://www.docker.com/products/docker-desktop) установочный файл, запускаем.

### Настройки Docker

#### Use the WSL2 based engine
![Use the WSL2 based engine](./img/docker-general-settings.png)

#### Resourses
![Use the WSL2 based engine](./img/docker-settings-resources.png)

С такими настройками докер будет работать внутри Linux дистро и дополнительно ничего устанавливать не потребуется.

## Работа с Linux distro

### Запуск

Запускаем дистро из меню пуск или через терминал (картинка):

![Use the WSL2 based engine](./img/linux-distro-start-terminal.png)

### Первичные настройки

1. Задать пользователя (для локальных тестов можно покороче)
2. Задать пароль (для локальных тестов можно покороче)

### Ставим JDK

Проверяем, есть ли у нас Java:

```java --version```

Терминал нам ответит что-то похожее на это, если Java не установлена:
```
user@qa-test-bed:~$ java --version
Command 'java' not found, but can be installed with:
sudo apt install default-jre
sudo apt install openjdk-11-jre-headless
sudo apt install openjdk-8-jre-headless
```
### Обновляемся

```sudo apt-get update && sudo apt-get upgrade```

### Устанавливаем JDK

```sudo apt-get install default-jdk```

Соглашаемся на установку и потом ждем, пока вся эта радость установится.

Ещё раз проверяем версию Java:

```java --version```

```
    PS C:\Users\username> java --version
    java 11.0.7 2020-04-14 LTS
    Java(TM) SE Runtime Environment 18.9 (build 11.0.7+8-LTS)
    Java HotSpot(TM) 64-Bit Server VM 18.9 (build 11.0.7+8-LTS, mixed mode)
```

### Прописываем переменную окружения JAVA_HOME (понадобится)

Джава вероятнее всего установилась вот так:

```/usr/lib/jvm/java-11-openjdk-amd64``` в JAVA_HOME необходимо прописывать именно этот путь, который не включает папку `bin`.

#### Редактируем файл с переменными окружения

```sudo nano /etc/environment```

добавляем на последнюю строку:

```JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64```

Нажимаем последовательно Ctrl+S (сохранить), Ctrl+X (выйти из редактора nano).

#### Обновляем сведения о переменных окружения

```source /etc/environment```

#### Проверяем, прописалась ли переменная окружения

```echo $JAVA_HOME```

```
    username@pc:$ source /etc/environment
    username@pc:$ echo $JAVA_HOME
    /usr/lib/jvm/java-11-openjdk-amd64
```
Всё ок.

## Ставим все через docker-compose

### Selenoid

Описано здесь: https://aerokube.com/selenoid/latest/#_selenoid_with_docker_compose

#### Готовим конфиг для docker-compose

В папке проекта создаем файл `docker-compose.yml`

```yaml
version: '3.4'

services:
  selenoid:
    image: aerokube/selenoid:latest-release
    volumes:
      - "${PWD}/init/selenoid:/etc/selenoid"
      - "${PWD}/work/selenoid/video:/opt/selenoid/video"
      - "${PWD}/work/selenoid/logs:/opt/selenoid/logs"
      - "/var/run/docker.sock:/var/run/docker.sock"
    environment:
      - OVERRIDE_VIDEO_OUTPUT_DIR=work/selenoid/video
    command: ["-conf", "/etc/selenoid/browsers.json", "-video-output-dir", "/opt/selenoid/video", "-log-output-dir", "/opt/selenoid/logs"]
    ports:
      - "4444:4444"
    network_mode: bridge
```

${PWD} - текущая директория

`volumes` позволяет монтировать папки файловой системы локальной машины в контейнер.

Для строки типа `"${PWD}/init/selenoid:/etc/selenoid"` слева от двоеточия находится папка, которая лежит на локальной машине (в Linux distro), справа — на файловой системе контейнера.

##### В папке проекта создаём подпапки
- init/selenoid
- work/selenoid/video
- work/selenoid/logs

#### Файлы внутри init/selenoid

Создаём `browsers.json`

```json
{
    "chrome": {
      "default": "88.0",
        "88.0": {
          "env" : ["LANG=ru_RU.UTF-8", "LANGUAGE=ru:en", "LC_ALL=ru_RU.UTF-8", "TZ=Europe/Moscow"],
          "image": "selenoid/vnc:chrome_88.0",
          "tmpfs": {"/tmp": "size=512m"},
          "port": "4444"
        }
      }
    }
  }
```
#### Образы браузеров

Образы браузеров необходимо скачивать самостоятельно. Для примера выше потребуется образ браузера `selenoid/vnc:chrome_88.0`, он скачивается следующим образом:

```bash
docker pull selenoid/vnc:chrome_88.0
```
После этого, можно попробовать запустить то, что получилось.

#### Запуск через docker-compose

В папке проекта выполняем:

```bash
    docker-compose up
```
В консоли будет что-то вроде вот этого:

```
user@pc:/path/project$ docker-compose up
Starting project_selenoid_1 ... done
Attaching to project_selenoid_1
selenoid_1     | 2021/03/03 20:33:19 [-] [INIT] [Loading configuration files...]
selenoid_1     | 2021/03/03 20:33:19 [-] [INIT] [Loaded configuration from /etc/selenoid/browsers.json]
selenoid_1     | 2021/03/03 20:33:19 [-] [INIT] [Video Dir: /opt/selenoid/video]
selenoid_1     | 2021/03/03 20:33:19 [-] [INIT] [Logs Dir: /opt/selenoid/logs]
selenoid_1     | 2021/03/03 20:33:19 [-] [INIT] [Your Docker API version is 1.41]
selenoid_1     | 2021/03/03 20:33:19 [-] [INIT] [Timezone: UTC]
selenoid_1     | 2021/03/03 20:33:19 [-] [INIT] [Listening on :4444]
```

В браузере заходим `http://your-ip-address:4444/status`

Должна быть видна JSON строка `{"total":5,"used":0,"queued":0,"pending":0,"browsers":{"chrome":{"75.0":{},"76.0":{},"87.0":{},"88.0":{}}}}`

Выключаем при помощи Ctrl+C, переходим к след. шагу.

### Selenoid UI

Описано здесь: http://aerokube.com/selenoid-ui/latest/#_with_docker_compose

Добавляем в файл `docker-compose.yml` необходимое для запуска selenoid-ui:

```yaml
  selenoid-ui:
    image: "aerokube/selenoid-ui:latest-release"
    network_mode: bridge
    links:
      - selenoid
    ports:
      - "8080:8080"
    command: ["--selenoid-uri", "http://selenoid:4444"]
```
#### Проверяем работоспособность

```bash
    docker-compose up
```
В браузере заходим `http://your-ip-address:8080`

Мы должны видеть веб-страницу selenoid-ui.

 ![Selenoid-ui up and running](./img/selenoid-ui-main.png)

1. Переходим на вкладку Capabilities
2. Выбираем браузер
3. Запускаем ручную сессию

Чтобы иметь возможность управлять браузером внутри контейнера, нужно его разблокировать (1) и лучше раскрыть VNC окно на максимум (2).

 ![Selenoid-ui up and running](./img/selenoid-manual-session.png)

Если все работает, то останавливаем docker-compose и переходим к установке Jenkins.  

### Jenkins

гуглим конфиг: https://is.gd/LKXZ9k

1. Добавляем к `docker-compose.yml`

```yaml
  jenkins:
    image: jenkins/jenkins:lts
    user: root
    volumes:
      - ${PWD}/work/jenkins:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - 8888:8080
    network_mode: bridge
```

2. Запускаем docker-compose

```bash
  docker-compose up
```
Ждём, когда закачается образ и Jenkins стартует. В логах должно появиться следующее сообщение:

```bash
jenkins_1      | Jenkins initial setup is required. An admin user has been created and a password generated.
jenkins_1      | Please use the following password to proceed to installation:
jenkins_1      |
jenkins_1      | 32d64223e62945fe8c4c1ed050360700
jenkins_1      |
jenkins_1      | This may also be found at: /var/jenkins_home/secrets/initialAdminPassword
jenkins_1      |
jenkins_1      | *************************************************************
jenkins_1      | *************************************************************
jenkins_1      | *************************************************************
```
Этот пароль понадобится для самого первого входа Jenkins, потом про него можно забыть.

3. Устанавливаем плагины по умолчанию (потом можно удалить то, что ненужно).
4. Создаём администратора и переходим к начальной конфигурации Jenkins.

#### Начальная конфигурация

##### Global tool configuration :: JDK > Add JDK

1. Убираем галку Install Automatically.
2. Name: можно написать что угодно, но лучше правду: OpenJDK 11.
3. JAVA_HOME вставляем ```$JAVA_HOME```.
4. Жмём APPLY внизу страницы.


##### Global tool configuration :: Gradle
1. Add gradle
2. Отмечаем Install Automatically.
3. Name: что угодно
4. Install from Gradle.org и выбрать нужную версию.

### Запуск стенда в detached режиме.

Этот режим нужен для того, чтобы вы не видели весь-весь консольный вывод приложений, которые у вас сейчас запущены в докере.

Останавливаем docker-compose Ctrl+C.

#### Запускаем в detached

```bash
    docker-compose up -d
```
Вы увидите в консоли только вот это:

```
user@pc:/path$ docker-compose up -d
Starting cm-local-test-bed-wsl_jenkins_1     ... done
Starting cm-local-test-bed-wsl_selenoid_1 ... done
Starting cm-local-test-bed-wsl_selenoid-ui_1 ... done
```
Далее вы вернётесь в консоль, а docker-compose продолжит свою работу.

#### Остановка docker-compose

Чтобы остановить все запущенные приложения, необходимо выполнить следующую команду:

```bash
  docker-compose down
```

Вот и всё.