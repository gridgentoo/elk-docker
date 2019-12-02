# Elastic stack (ELK) on Docker

[![Join the chat at https://gitter.im/gridgentoo/elk-docker](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/gridgentoo/elk-docker?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![Elastic Stack version](https://img.shields.io/badge/ELK-7.4.1-blue.svg?style=flat)](https://github.com/gridgentoo/elk-docker/issues/441)
[![Build Status](https://api.travis-ci.org/gridgentoo/elk-docker.svg?branch=master)](https://travis-ci.org/gridgentoo/elk-docker)

Как запустить последнюю версию стека [Elastic stack][elk-stack] с помощью Docker и Docker Compose.

Это дает вам возможность анализировать любой набор данных, используя возможности searching/aggregatio Elasticsearch и возможности визуализации Kibana.

> :information_source: Docker images, поддерживающие этот стек, включают include [Stack Features][stack-features] (formerly X-Pack)
c [платными функциями][paid-features] включенными по умолчанию (Где смотреть [Как отключить платные функции](#how-to-disable-paid-features) как disable платные функции). [Пробная лицензия][trial-license] действительна в течение 30 дней..

Основано на официальных Docker images от Elastic:

* [Elasticsearch](https://github.com/elastic/elasticsearch/tree/master/distribution/docker)
* [Logstash](https://github.com/elastic/logstash/tree/master/docker)
* [Kibana](https://github.com/elastic/kibana/tree/master/src/dev/build/tasks/os_packages/docker_generator)

Другие доступные варианты стека::

* [`searchguard`](https://github.com/gridgentoo/elk-docker/tree/searchguard): Search Guard support

## Contents

1. [Requirements](#requirements)
   * [Host setup](#host-setup)
   * [SELinux](#selinux)
   * [Docker for Desktop](#docker-for-desktop)
2. [Usage](#usage)
   * [Bringing up the stack](#bringing-up-the-stack)
   * [Cleanup](#cleanup)
   * [Initial setup](#initial-setup)
     * [Setting up user authentication](#setting-up-user-authentication)
     * [Injecting data](#injecting-data)
     * [Default Kibana index pattern creation](#default-kibana-index-pattern-creation)
3. [Configuration](#configuration)
   * [How to configure Elasticsearch](#how-to-configure-elasticsearch)
   * [How to configure Kibana](#how-to-configure-kibana)
   * [How to configure Logstash](#how-to-configure-logstash)
   * [How to disable paid features](#how-to-disable-paid-features)
   * [How to scale out the Elasticsearch cluster](#how-to-scale-out-the-elasticsearch-cluster)
4. [Extensibility](#extensibility)
   * [How to add plugins](#how-to-add-plugins)
   * [How to enable the provided extensions](#how-to-enable-the-provided-extensions)
5. [JVM tuning](#jvm-tuning)
   * [How to specify the amount of memory used by a service](#how-to-specify-the-amount-of-memory-used-by-a-service)
   * [How to enable a remote JMX connection to a service](#how-to-enable-a-remote-jmx-connection-to-a-service)
6. [Going further](#going-further)
   * [Using a newer stack version](#using-a-newer-stack-version)
   * [Plugins and integrations](#plugins-and-integrations)
   * [Swarm mode](#swarm-mode)

## Requirements(Требования)

### Host setup

* [Docker Engine](https://docs.docker.com/install/) version **17.05+**
* [Docker Compose](https://docs.docker.com/compose/install/) version **1.12.0+**
* 1.5 GB of RAM

По умолчанию в стеке доступны следующие порты:
* 5000: Logstash TCP input
* 9200: Elasticsearch HTTP
* 9300: Elasticsearch TCP transport
* 5601: Kibana

Проверки начальной загрузки Elasticsearch были преднамеренно отключены, чтобы упростить настройку стека Elastic в средах разработки.
> :information_source: Elasticsearch's [bootstrap checks][booststap-checks] были намеренно отключены, 
> чтобы облегчить настройка стека Elastic development environments. 
> Для производственных установок мы рекомендуем пользователям настроить свой host
> в соответствии с инструкциями из документации Elasticsearch: [Important System Configuration][es-sys-config].

### SELinux

В дистрибутивах, в которых SELinux включен «из коробки», вам потребуется либо re-context файлы, либо перевести SELinux в режим Permissive mode, чтобы правильно запустить docker-elk. Например, в Redhat и CentOS следующее будет применять правильный context:

```console
$ chcon -R system_u:object_r:admin_home_t:s0 docker-elk/
```

### Docker for Desktop

## Usage

### Bringing up the stack
### Поднимая stack(стек)  

Клонируйте этот репозиторий, затем запустите stack(стек), используя Docker Compose:  


```console
$ docker-compose up
```

Вы также можете запустить все службы в фоновом режиме (detached mode) (в автономном режиме), добавив `-d` flag вышеупомянутой команды.   

> :information_source: Cначала вы должны запускать сборку run `docker-compose build`  всякий раз, когда вы переключаете ветку или обновляете базовый image.  

Если вы запускаете stack(стек) впервые, пожалуйста, внимательно прочитайте раздел ниже.

### Cleanup

По умолчанию (Elasticsearch data) сохраняются внутри (volume)(тома).

Чтобы полностью отключить стек и удалить все сохраненные данные(remove all persisted data), используйте следующую команду Docker Compose:

```console
$ docker-compose down -v
```

## Initial setup

### Setting up user authentication
### Настройка аутентификации пользователя

> :information_source: Обратитесь к разделу (как отключить платные функции) [How to disable paid features](#how-to-disable-paid-features) для отключения аутентификации.  

(Stack)(Стек) предварительно настроен для следующего привилегированного пользователя **privileged** (bootstrap user):

* user: *elastic*
* password: *changeme*

Хотя все компоненты стека работают с этим пользователем «из коробки», мы настоятельно рекомендуем использовать непривилегированных встроенных пользователей [built-in users][builtin-users] вместо этого для повышения безопасности.  

1. Инициализировать пароли для встроенных пользователей (built-in users)  

```console
$ docker-compose exec -T elasticsearch bin/elasticsearch-setup-passwords auto --batch
```

Пароли для всех (6 built-in users) встроенных пользователей будут cгенерированы случайным образом. 

2. Удалить пароль начальной загрузки (bootstrap password) (_optional_)  

Удалите переменную среды `ELASTIC_PASSWORD` из службыa `elasticsearch` внутри файла Compose (docker-compose.yml). Он используется только для инициализации (keystore) хранилища ключей во время первоначального запуска Elasticsearch.  

3. Замените имена пользователей и пароли в файлах конфигурации  

Используйте пользователя `kibana` внутри файла конфигурации Kibana (`kibana/config/kibana.yml`)  и пользователя `logstash_system` внутри файла конфигурации Logstash (`logstash/config/logstash.yml`) вместо существующего `elastic` user.

Замените пароль для `elastic` user в файле конвейера Logstash pipeline (`logstash/pipeline/logstash.conf`).

Не используйте пользователя logstash_system внутри файла конвейера Logstash, у него недостаточно прав для создания индексов. Следуйте инструкциям в разделе «Настройка безопасности в Logstash», чтобы создать пользователя с подходящими ролями.

> :information_source: Не используйте пользователя `logstash_system` внутри файла конвейера Logstash *pipeline* file, у него недостаточно
> (permissions)(прав) для создания (индексов)(indices).  Следуйте инструкциям в разделе [Configuring Security in Logstash][ls-security]
> чтобы создать user с подходящими (ролями)(roles).

Смотрите также раздел [Configuration](#configuration) ниже.  

4. Перезапустите Kibana и Logstash, чтобы применить изменения

```console
$ docker-compose restart kibana logstash
```
> :information_source: Узнайте больше о безопасности стека Elastic stack на Tutorial [Tutorial: Getting started with
> security][sec-tutorial].  

### Injecting data
### Внедрение данных

Дайте Kibana около минуты для инициализации, затем откройте веб-интерфейс Kibana, нажав http: // localhost: 5601 с помощью веб-браузера, и введите следующие учетные данные по умолчанию для входа в систему:

Дайте Kibana около минуты для инициализации, затем откройте веб-интерфейс Kibana web UI нажав 
[http://localhost:5601](http://localhost:5601) с помощью веб-браузера, и введите следующие (сredentials)(учетные данные) по умолчанию для входа в систему:  

* user: *elastic*
* password: *\<your generated elastic password>*

Теперь, когда stack работает, вы можете пойти дальше и добавить несколько записей журнала inject some log entries. Поставленная конфигурация Logstash позволяет отправлять контент через TCP:  

```console
$ nc localhost 5000 < /path/to/logfile.log
```

Вы также можете загрузить пример данных, предоставленных вашей установкой Kibana.

### Default Kibana index pattern creation  
### Создание патерна индекса Kibana по умолчанию  

Когда Kibana запускается впервые, ни на один (индексный паттерн)(index pattern) еще не настроен.  

#### Via the Kibana web UI

Вам необходимо внедрить данные в Logstash, прежде чем вы сможете настроить шаблон индекса Logstash через веб-интерфейс Kibana. Тогда все, что вам нужно сделать, это нажать кнопку «Создать».
> :information_source: Вам необходимо (inject data) внутрь Logstash, прежде чем вы сможете настроить (Logstash index pattern) 
> через веб-интерфейс Kibana web UI. Тогда все, что вам нужно сделать, это нажать кнопку *Create*.

Обратитесь к [Connect Kibana with Elasticsearch][connect-kibana] за подробными инструкциями о конфигурации (index pattern).    

#### On the command line

Создайте (index pattern) с помощью Kibana API:

```console
$ curl -XPOST -D- 'http://localhost:5601/api/saved_objects/index-pattern' \
    -H 'Content-Type: application/json' \
    -H 'kbn-version: 7.4.1' \
    -u elastic:<your generated elastic password> \
    -d '{"attributes":{"title":"logstash-*","timeFieldName":"@timestamp"}}'
```

Созданный шаблон будет автоматически помечен как index pattern  по умолчанию, как только Kibana UI будет открыт в первый раз.  

## Configuration

> :information_source: Конфигурация не загружается динамически, вам нужно будет перезапускать отдельные компоненты после
каждого изменения конфигурации.  

### How to configure Elasticsearch

Конфигурация Elasticsearch хранится в [`elasticsearch/config/elasticsearch.yml`][config-es].

Вы также можете указать (options)(параметры), которые вы хотите переопределить, установив (environment variable) переменные среды внутри файла Compose: 

```yml
elasticsearch:

  environment:
    network.host: _non_loopback_
    cluster.name: my-cluster
```

Пожалуйста, обратитесь к следующей странице документации для получения более подробной информации о том, как настроить Elasticsearch внутри контейнеров Docker: [Install Elasticsearch with Docker][es-docker].

### How to configure Kibana

Конфигурация Kibana по умолчанию хранится в [`kibana/config/kibana.yml`][config-kbn].

Также возможно (map)(отобразить) весь каталог `config`  вместо одного файла.

Пожалуйста, обратитесь к следующей странице документации для получения более подробной информации о том, 
как настроить Kibana внутри контейнеров Docker: [Running Kibana on Docker][kbn-docker].

### How to configure Logstash

Конфигурация Logstash хранится в [`logstash/config/logstash.yml`][config-ls].

Также возможно (map)(отобразить) весь каталог конфигурации `config` вместо одного файла, однако вы должны знать,
что Logstash будет ожидать файл  [`log4j2.properties`][log4j-props] для своей собственной (logging)(регистрации).

Пожалуйста, обратитесь к следующей странице документации для получения более подробной информации о том, как настроить Logstash внутри контейнеров Docker: [Configuring Logstash for Docker][ls-docker].

### How to disable paid features
### Как отключить платные функции

Переключите значение (option)(параметра) `xpack.license.self_generated.type` в Elasticsearch с `trial` на `basic` (see [License
settings][trial-license]).  

### How to scale out the Elasticsearch cluster
### Как масштабировать кластер Elasticsearch

Следуйте инструкциям из вики: [Scaling out Elasticsearch](https://github.com/gridgentoo/elk-docker/wiki/Elasticsearch-cluster)

## Extensibility

### How to add plugins
### Как добавить плагины

Чтобы добавить plugins  к любому компоненту ELK, вы должны:

1. Добавить оператор `RUN` statement в соответствующий `Dockerfile` (eg. `RUN logstash-plugin install logstash-filter-json`)
2. Добавьте соответствующую (plugin code configuration) в конфигурацию сервиса (eg. Logstash input/output)
3. (Перестройте images)(Rebuild the images) используя команду сборки `docker-compose build` 

### How to enable the provided extensions
### Как включить предоставленные расширения

Несколько расширений доступны в каталоге расширений [`extensions`](extensions). Эти расширения предоставляют функции, которые не являются частью стандартного стека  Elastic stack, но могут использоваться для обогащения его дополнительными интеграциями.

Документация для этих (расширений)(extensions) предоставляется внутри каждого отдельной (subdirectory) для каждого расширения. Несколько
из них требуют ручного изменения конфигурации (default ELK configuration).

## JVM tuning
## Тюнинг JVM

### Как указать объем памяти, используемой (сервисом)(service)

По умолчанию Elasticsearch и Logstash начинаются с [1/4 of the total host
memory](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size), выделенной для размера  JVM Heap.

Сценарии запуска (startup scripts) для Elasticsearch и Logstash могут добавлять дополнительные JVM options из значения environment
variable, позволяющая пользователю регулировать объем памяти, который может использоваться каждым компонентом:

| Service       | Environment variable |
|---------------|----------------------|
| Elasticsearch | ES_JAVA_OPTS         |
| Logstash      | LS_JAVA_OPTS         |

Чтобы приспособиться к (средам)(environments), в которых не хватает памяти (по умолчанию в Docker для Mac доступно только 2 ГБ), выделение размера кучи (Heap Size) по умолчанию ограничено 256MB на службу в файле  `docker-compose.yml`. Если вы хотите переопределить конфигурацию JVM по умолчанию, отредактируйте соответствующие переменные среды в файле `docker-compose.yml`.

Например, чтобы увеличить максимальный размер кучи (JVM Heap Size) для Logstash:

```yml
logstash:

  environment:
    LS_JAVA_OPTS: -Xmx1g -Xms1g
```

### How to enable a remote JMX connection to a service
### Как включить удаленное соединение JMX с сервисом

Что касается памяти кучи (Java Heap memory) (см. Выше), вы можете указать JVM options, чтобы включить JMX и (map)(отобразить) JMX port на хосте Docker.

Обновите (nvironment variable)(переменную среды) `{ES,LS}_JAVA_OPTS` следующим содержанием (я сопоставил JMX service с портом 18080, вы можете это изменить). Не забудьте обновить параметр `-Djava.rmi.server.hostname` с помощью IP-адреса вашего хоста Docker(Docker host) (replace **DOCKER_HOST_IP**):

```yml
logstash:

  environment:
    LS_JAVA_OPTS: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.port=18080 -Dcom.sun.management.jmxremote.rmi.port=18080 -Djava.rmi.server.hostname=DOCKER_HOST_IP -Dcom.sun.management.jmxremote.local.only=false
```

## Going further
## Идти дальше

### Using a newer stack version
### Использование более новой версии стека

Чтобы использовать версию Elastic Stack, отличную от той, которая доступна в данный момент в репозитории, просто измените номер версии в файле `.env` и (перестройте)(rebuild) стек с помощью:

```console
$ docker-compose build
$ docker-compose up
```

> :information_source: Всегда обращайте внимание на [upgrade instructions][upgrade] для каждого отдельного компонента перед выполнением обновления  stack.

### Plugins and integrations
### Плагины и интеграции

Смотрите следующие вики-страницы::

* [External applications](https://github.com/gridgentoo/elk-docker/wiki/External-applications)
* [Popular integrations](https://github.com/gridgentoo/elk-docker/wiki/Popular-integrations)

### Swarm mode

Экспериментальная поддержка режима Docker [Swarm mode][swarm-mode] предоставляется в виде файла `docker-stack.yml` который можно развернуть в существующем кластере Swarm с помощью следующей команды:

```console
$ docker stack deploy -c docker-stack.yml elk
```

Если все компоненты будут (deployed)(развернуты) без каких-либо ошибок, следующая команда покажет 3 запущенных (службы)(services):

```console
$ docker stack services elk
```

> :information_source: Чтобы масштабировать Elasticsearch в режиме Swarm, configure *zen* на использование DNS-имени `tasks.elasticsearch`
вместо `elasticsearch`.


[elk-stack]: https://www.elastic.co/elk-stack
[stack-features]: https://www.elastic.co/products/stack
[paid-features]: https://www.elastic.co/subscriptions
[trial-license]: https://www.elastic.co/guide/en/elasticsearch/reference/current/license-settings.html

[booststap-checks]: https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
[es-sys-config]: https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html

[win-shareddrives]: https://docs.docker.com/docker-for-windows/#shared-drives
[mac-mounts]: https://docs.docker.com/docker-for-mac/osxfs/

[builtin-users]: https://www.elastic.co/guide/en/x-pack/current/setting-up-authentication.html#built-in-users
[ls-security]: https://www.elastic.co/guide/en/logstash/current/ls-security.html
[sec-tutorial]: https://www.elastic.co/guide/en/elastic-stack-overview/current/security-getting-started.html

[connect-kibana]: https://www.elastic.co/guide/en/kibana/current/connect-to-elasticsearch.html

[config-es]: ./elasticsearch/config/elasticsearch.yml
[config-kbn]: ./kibana/config/kibana.yml
[config-ls]: ./logstash/config/logstash.yml

[es-docker]: https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html
[kbn-docker]: https://www.elastic.co/guide/en/kibana/current/docker.html
[ls-docker]: https://www.elastic.co/guide/en/logstash/current/docker-config.html

[log4j-props]: https://github.com/elastic/logstash/tree/7.3/docker/data/logstash/config
[esuser]: https://github.com/elastic/elasticsearch/blob/7.3/distribution/docker/src/docker/Dockerfile#L18-L19

[upgrade]: https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html

[swarm-mode]: https://docs.docker.com/engine/swarm/
