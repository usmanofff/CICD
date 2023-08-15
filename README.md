Спринт 2 

ЗАДАЧА

```
1. Клонируем репозиторий, собираем его на сервере srv.
Исходники простого приложения можно взять здесь. Это простое приложение на Django с уже написанным Dockerfile. 
Приложение работает с PostgreSQL, в самом репозитории уже есть реализация docker-compose — её можно брать за 
референс при написании Helm-чарта. Необходимо склонировать репозиторий выше к себе в Git и настроить пайплайн 
с этапом сборки образа и отправки его в любой docker registry. Для пайплайнов можно использовать GitLab, 
Jenkins или GitHub Actions — кому что нравится. Рекомендуем GitLab.

2. Описываем приложение в Helm-чарт.
Описываем приложение в виде конфигов в Helm-чарте. По сути, там только два контейнера — с базой и приложением, 
так что ничего сложного в этом нет. Стоит хранить данные в БД с помощью PVC в Kubernetes.

3. Описываем стадию деплоя в Helm.
Настраиваем деплой стадию пайплайна. Применяем Helm-чарт в наш кластер. Нужно сделать так, чтобы наше приложение 
разворачивалось после сборки в Kubernetes и было доступно по бесплатному домену или на IP-адресе с выбранным портом.
Для деплоя должен использоваться свежесобранный образ. По возможности нужно реализовать сборку из тегов в Git, где 
тег репозитория в Git будет равен тегу собираемого образа. Чтобы создание такого тега запускало пайплайн на сборку 
образа c таким именем hub.docker.com/skillfactory/testapp:2.0.3.
```

Создаем в корне проекта папку CICD и клонируем исходники приложения ```https://github.com/vinhlee95/django-pg-docker-tutorial```

Проверяем  Dockerfile исправляем ошибки, разрешаем подключения с хостов в settings.py 
```
ALLOWED_HOSTS = ['*']
DEBUG = True
```
Подсовываем enventory file: 
```
DB_HOST=
DB_NAME=
DB_USER=
DB_PASS=
POSTGRES_DB=
POSTGRES_USER=
POSTGRES_PASSWORD=
```
Проверяем запуск через ``` docker-compose up -d ```

![image](https://github.com/usmanofff/CICD/assets/74288450/51ecb3ef-8a9d-490d-a736-f7a10bd90086)

Проверем ```curl localhost:3003 ``` приложение должно отвечать.

![image](https://github.com/usmanofff/CICD/assets/74288450/064125a2-c453-476a-8c61-c317d44ab44e)

Если все запустилось останавливаем контейнеры ``` docker stop $(docker ps -qa) ``` 

Далее будем описывать CI-CD процесс сборки и деплоя в Doker-registry для этого будем использовать gitlab.

Создаем учетку на DockerHub и логинемся командой:

```Docker login``` 

Собираем приложение из Dockerfile и отправляем rigestry: 

```docker biuld -t usmcode/diplom:v.1 .``` 
```docker push usmcode/diplom:v.1```


![image](https://github.com/usmanofff/CICD/assets/74288450/c06121a4-5c15-4f26-a48f-218c16110918)


Создал папку APP-DOCKER и подключил к ней свой репозиторий в gitlab : ``` https://gitlab.com/sfc_diplom/app ```

команды подключения репозитория :
```
git init
git branch -M main
git remote add origin git@gitlab.com:sfc_diplom/app.git
git push -uf origin
```
Настройки gitlab :

- Разрешаем запуск gitlab-runner для нашего проекта
- Разрешаем push в ветку main
- Отключаем shared runners

![runners](https://github.com/usmanofff/CICD/assets/74288450/718ab76f-65da-460b-aa1a-f1c636b42074)

Проверяем работу gitlab-runner на SRV : ```systemctl status gitlab-runner```

![image](https://github.com/usmanofff/CICD/assets/74288450/e9e28200-c24b-4f63-bd01-8a3fe4a74533)


config.toml который создался при активации gitlab-runner положил в /etc/gitlab-runner для автоматического запуска.

<h1>Подготовка Helm Chart </h1>

  - Создаем папку для манифестов KUBE-MANIFEST:

Нам пондобятся:

kind: Secret:
- файл all_credentials.yaml Kind:Secret с чуствительными данными , которые необходимо предварительно зашифровать.

  ``` echo -n "" | base64 - шифруем , | base64 --decode так дешифруем ```

Kind:Deployment :
- файл app-deployment.yaml  с описанием запуска приложения  usmcode/diplom:v.1 из нашего Docker rigestry.
- файл db-deployment.yaml  c описание запуска базы данных postgres:13-alpine для нашего преложения. 

Kind:Service : (взаимодействие внутри кластера)
  - app_service-cluster-ip.yaml сервис для открытие порта 3003 приложения
  - db_service-cluster-ip.yaml сервис для открытие порта 5432 postgres 

Kind:ingress : (для вывода приложения в интернет )  вкачестве ingress контроллера исползовал ``` https://projectcontour.io/getting-started/ ```
  - app_ingress.yaml ingress правило для перенапровление трафика по выбранному селектору сервиса cluster_ip

kind: PersistentVolume (для postgress)
  - db_store.yaml Хранилище данных

Запускаем наши манифесты  ``` kubectl -n diplom apply -f . ```

Проверяем командой ``` kubectl -n diplom get all ```  ``` kubectl get po -A ```

![k_get_all](https://github.com/usmanofff/CICD/assets/74288450/5e6e0e28-91c4-4de1-81d8-2f61703319d1)

Если что то не заработало можно проверять логи командой   ```kubect -n dipom logs pods/имя пода```

так же можно проверить env внутри пода    ``` kubectl -n diplom exec -it pods/имя пода -с под -- env ```  

Если все запустилось успешно идем дальше...

 --- Создаем Helm Chat на основе созданных манифестов , заменем данные переменными из файла values.yaml

 Структура чарта: 

 ![image](https://github.com/usmanofff/CICD/assets/74288450/87447cee-93d1-45ac-940e-3c6e89ccf084)


Создаем файл .gitlab-ci.yml в котором описываем этапы сборки :

stage:
  - build:
  - deploy:

Для чуствительных данных необходимо создать переменные : 

Settings-CICD-Variables :

![image](https://github.com/usmanofff/CICD/assets/74288450/826eaae4-65ec-4fa4-9e5d-c1f41919c1f5)

- На стадии build описываем процесс сборки приложения из dockerfile и отправки его в Docker rigestry меткой $tag

Вывод:

![image](https://github.com/usmanofff/CICD/assets/74288450/e8ed9919-a326-4903-a77f-9ec672e3c461)

- На стадии deploy разварачиваем Helm Chart тригиром должно быть изменение тэга

Вывод:

![image](https://github.com/usmanofff/CICD/assets/74288450/7702c830-43da-464f-b062-8a4ff6225e77)


Вносим изменения в проект создаем тэг ``` git tag -a v.1.6 ``` git push --tag ```

Должен запустится pipeline и применится новая версия приложения. 

![image](https://github.com/usmanofff/CICD/assets/74288450/ee898f59-21f7-4ff3-b602-e1472e1eb917)

![image](https://github.com/usmanofff/CICD/assets/74288450/d89c963c-c75b-4fc2-ae65-34b8c98af180)

![image](https://github.com/usmanofff/CICD/assets/74288450/be8abb1e-fa2a-4d60-947a-822982ec8ab0)












