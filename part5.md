## Введение: зачем тебе Kubernetes как Java‑разработчику

Представь, что у тебя есть несколько микросервисов на Spring Boot, база данных PostgreSQL, брокеры сообщений RabbitMQ и Kafka, а всё это крутится в Docker. Пока сервисов один‑два, проект можно поднимать локально docker compose и вручную раскатывать на сервер. Но как только количество сервисов растёт, начинаются проблемы: где какой сервис запущен, на каком порту он слушает, как переразвернуть новую версию без даунтайма, как перезапустить упавший контейнер, как масштабировать только один сервис, а не все сразу. Здесь появляется Kubernetes. Это система, которая берёт на себя запуск, перезапуск, масштабирование и сетевую связность контейнеров, а ты описываешь нужное состояние в декларативных манифестах YAML.

В этой лекции мы пойдём от основ к практическим сценариям. Сначала разберём, что такое кластер, ноды и поды. Затем поднимем Minikube на локальной машине, научимся пользоваться kubectl, развернём простое приложение, затем PostgreSQL в виде StatefulSet с headless‑сервисом, и посмотрим, как добавлять в Kubernetes RabbitMQ и Kafka. По пути будем постоянно смотреть на реальные команды, YAML‑манифесты и типичные ошибки начинающих.

## Кластер, ноды и поды: как Kubernetes организует твои контейнеры

В терминах Kubernetes всё, что ты запускаешь, живёт внутри кластера. Кластер можно представить как группу машин, объединённых общим мозгом. Каждая машина в этом кластере называется нодой. Нода может быть физическим сервером, виртуальной машиной в облаке или локальной виртуалкой, как в случае с Minikube. В типичном проде есть управляющие ноды, где живут компоненты control plane (API‑сервер, планировщик, контроллеры), и рабочие ноды, на которых крутятся твои приложения.

Kubernetes сам по себе не запускает контейнеры напрямую. Он оперирует сущностью под. Под — это минимальная единица развертывания в Kubernetes. В одном поде могут жить один или несколько контейнеров, которые всегда запускаются и умирают вместе. Они разделяют сеть и диски. Если смотреть глазами Docker, под — это как группа связанных контейнеров с общим IP внутри кластера. На практике для большинства микросервисов используется один контейнер на под, но сама абстракция важна: ты никогда не создаёшь Deployment из контейнеров напрямую, только из подов.

### Почему под — это не просто контейнер

Главная идея пода в том, что Kubernetes управляет не конкретным контейнером, а логической единицей. Если контейнер внутри пода упал, Kubernetes создаёт новый под, а не перезапускает старый в лоб. У пода есть своё имя, IP‑адрес и жизненный цикл. Важно понимать, что под — это эфемерная сущность. Сегодня он называется postgres‑0, а завтра при перезапуске может появиться другой под с тем же именем, но уже с другим внутренним состоянием, если ты не закрепил его за постоянным хранилищем.

Чтобы гарантировать, что нужное количество подов всегда запущено, Kubernetes использует контроллеры более высокого уровня, например Deployment и StatefulSet. Они отслеживают желаемое количество реплик и автоматически создают или удаляют поды, чтобы привести реальное состояние к желаемому.

## Minikube: локальный Kubernetes‑кластер для практики

Для обучения и локальной разработки удобно использовать Minikube. Это инструмент, который поднимает одноузловой кластер Kubernetes на твоей машине. Внутри он запускает виртуальную машину или набор контейнеров с нужными компонентами кластера. Для macOS команда установки Minikube может выглядеть так же, как у тебя в заметках.

Пример установки Minikube на macOS через скачивание бинарника и установку в системный путь:

```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-darwin-amd64
sudo install minikube-darwin-amd64 /usr/local/bin/minikube
```

Эта команда скачивает бинарник для macOS и устанавливает его как исполняемый файл minikube в каталог bin. После этого можно запускать локальный кластер. Частая ошибка начинающих — забыть выдать права на выполнение или положить бинарник не в тот каталог, который есть в PATH. Команда install делает это за тебя, поэтому дополнительных chmod делать не нужно.

Запустим кластер c увеличенным объёмом памяти, чтобы нормально запускать базы и брокеры:

```bash
minikube start --memory=4g
```

Minikube создаёт кластер, настраивает control plane и регистрирует единственную ноду, которая одновременно играет роль управляющей и рабочей. Если при старте ты видишь ошибки про виртуализацию, обычно причина в том, что не включен hypervisor (например, в BIOS или в настройках macOS), либо Minikube не может подобрать драйвер виртуализации. В этом случае в документации Minikube есть отдельный раздел с выбором драйвера, например docker‑драйвер вместо виртуалки.

Чтобы заглянуть внутрь кластера на уровне операционной системы, у Minikube есть возможность подключения по SSH:

```bash
minikube ssh
```

Эта команда запускает shell внутри виртуальной машины Minikube. Там можно увидеть запущенные процессы, файлы и даже внутренние IP‑адреса подов, но в повседневной работе Java‑разработчику хватает kubectl.

## kubectl: основной инструмент общения с кластером

kubectl — это CLI‑клиент для общения с Kubernetes API‑сервером. Через него ты создаёшь, обновляешь и удаляешь ресурсы кластера: поды, деплойменты, сервисы, конфиги, секреты, StatefulSet, Ingress и так далее.

Установка kubectl на macOS может выглядеть так:

```bash
brew install kubectl
```

Если brew недоступен или ты хочешь поставить бинарник вручную, можно воспользоваться командой из твоей заметки:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
sudo chown root: /usr/local/bin/kubectl
```

Сначала скачивается стабильная версия kubectl, затем бинарнику назначаются права на выполнение, после чего он перемещается в каталог bin и меняется владелец на root. Без chmod +x запуск приведёт к ошибке permission denied. Без перемещения в каталог, который входит в PATH, придётся всё время вызывать kubectl через относительный путь, что неудобно.

Проверим, что клиент установлен:

```bash
kubectl version --client
```

Если конфиг контекста не настроен, но уже запущен Minikube, можно позволить Minikube прописать текущий контекст автоматически. При первом запуске minikube start он обычно сам настраивает файл kubeconfig, поэтому kubectl сразу начинает работать с локальным кластером. Проверить доступ к кластеру можно командой:

```bash
kubectl get nodes
```

В ответ ты увидишь одну ноду, например minikube со статусом Ready. Если вместо этого получаешь ошибку connection refused к API‑серверу, скорее всего, кластер Minikube не запущен или контекст kubectl смотрит на другой кластер. В таких случаях помогает команда kubectl config current‑context и при необходимости переключение на minikube через kubectl config use‑context minikube.

## Первый под: простой Hello World

Самый быстрый способ увидеть результат своей работы — запустить одиночный под с уже готовым Docker‑образом. В заметках у тебя есть пример с образом amigoscode/kubernetes:hello‑world. Запуск через kubectl run выглядит так:

```bash
kubectl run hello-world \
  --image=amigoscode/kubernetes:hello-world \
  --port=80
```

Команда run создаёт ресурс Pod с именем hello‑world, у которого один контейнер на базе указанного образа и открытый порт 80 внутри пода. Важно понимать, что это командный способ создания пода. В продакшене и в нормальном проекте лучше описывать ресурсы в YAML‑манифестах, чтобы их можно было версионировать в Git и воспроизводить состояние кластера.

Проверим, что под запустился:

```bash
kubectl get pods
```

Здесь видно имя пода, статус, количество рестартов и возраст. Статус Running означает, что контейнер внутри пода запущен. Если ты видишь статус ImagePullBackOff, значит Kubernetes не может скачать образ. Причины обычно такие: неправильное имя образа, отсутствие доступа к приватному registry, неверный тег. В этом случае помогает kubectl describe pod hello‑world, по которому можно увидеть подробную ошибку.

### Доступ к поду через port‑forward

По умолчанию под живёт только внутри кластера, и его IP не доступен из твоей машины напрямую. Чтобы протестировать HTTP‑сервис, можно использовать port‑forward. В заметке у тебя есть пример:

```bash
kubectl port-forward pod/hello-world 8080:80
```

Эта команда говорит: открой локальный порт 8080 на твоём ноутбуке и пробрось его на порт 80 внутри пода hello‑world. Теперь можно открыть браузер или curl по адресу http://localhost:8080 и увидеть ответ сервиса, хотя он физически работает внутри кластера. Заблуждение новичков в том, что они ожидают, что под сразу же будет доступен снаружи по какому‑то IP и порту. В Kubernetes доступ наружу обычно делается через сервисы типа LoadBalancer или Ingress, а port‑forward — это более локальный инструмент отладки.

Когда под больше не нужен, его можно удалить:

```bash
kubectl delete pod hello-world
```

Под исчезнет из кластера. Если бы он был частью Deployment, контроллер автоматически создал бы новый под, чтобы сохранить желаемое количество реплик. В случае одиночного пода Kubernetes просто выполнит удаление.

## Deployment: как держать нужное количество реплик приложения

Создавать поды напрямую удобно только для экспериментов. В реальном приложении тебе нужен контроллер, который будет поддерживать нужное количество реплик и обновлять приложение без простоя. Для этого используется Deployment. Он описывает желаемое состояние: образ, количество реплик, порты, переменные окружения. Kubernetes создаёт ReplicaSet и поды на его основе, а при изменении Deployment аккуратно перекатывает новую версию.

Вот пример простого Deployment для приложения на базе готового Docker‑образа:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: amigoscode/kubernetes:hello-world
          ports:
            - containerPort: 80
```

В этом YAML описывается объект Deployment с именем hello‑deployment. Поле replicas равно двум, значит Kubernetes будет стараться держать две копии подов. В секции selector и в секции template.metadata.labels используются одинаковые метки app: hello. Это важно: селектор определяет, какие поды принадлежат этому Deployment. Если метки не совпадут, Deployment не увидит свои же поды, и статус будет некорректным. Внутри template описано тело пода: контейнер с именем hello, образом amigoscode/kubernetes:hello‑world и портом 80.

Чтобы применить этот манифест, его сохраняют в файл, например hello‑deployment.yaml, и запускают:

```bash
kubectl apply -f hello-deployment.yaml
```

Команда apply либо создаёт ресурс, либо обновляет его до нужной версии. После этого можно посмотреть список подов:

```bash
kubectl get pods -o wide
```

Количество подов с меткой app=hello будет равно двум, и у каждой реплики будет своё уникальное имя, например hello‑deployment‑7f5c4c7f9b‑kqj4q и hello‑deployment‑7f5c4c7f9b‑zq88p. Если один под упадёт, Deployment через ReplicaSet создаст новую копию. Это и есть автолечение, одна из важных фич Kubernetes.

Обновление версии образа делается просто: меняешь тег image в YAML и снова вызываешь kubectl apply. Kubernetes сам начнёт rolling update, постепенно создавая новые поды и удаляя старые. При ошибках образа или конфигурации можно откатиться с помощью kubectl rollout undo deployment hello‑deployment.

## Services: устойчивые адреса и балансировка трафика

Поды в Kubernetes эфемерные. Их IP‑адреса меняются при удалении и пересоздании. Нельзя жёстко прописать IP пода в конфиге другого сервиса. Для устойчивой сетевой адресации используется объект Service. Сервис обеспечивает стабильное DNS‑имя и балансирует трафик между подами, которые имеют нужные метки.

Простейший сервис для нашего Deployment может выглядеть так:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  selector:
    app: hello
  ports:
    - name: http
      port: 80
      targetPort: 80
  type: ClusterIP
```

Этот Service называется hello‑service и выбирает все поды с меткой app=hello. Поле type ClusterIP означает, что сервис создаёт внутренний виртуальный IP‑адрес, доступный только внутри кластера. Порт port задаёт номер порта, по которому другие поды будут подключаться к сервису. Параметр targetPort говорит, на какой порт внутри контейнера будет перенаправлен трафик. В нашем случае это совпадает и равно 80. Если targetPort не указан, Kubernetes использует значение из port.

После применения манифеста:

```bash
kubectl apply -f hello-service.yaml
kubectl get svc
```

В списке сервисов появится hello‑service с типом ClusterIP. Любой под внутри кластера теперь может обратиться к нашему приложению по DNS‑имени hello‑service (или hello‑service.default.svc.cluster.local, если говорить о полном имени). При этом Kubernetes сам распределяет запросы между всеми живыми подами, имеющими нужную метку. Начинающие часто пытаются подключиться к такому сервису прямо с ноутбука по ClusterIP, но этот IP существует только внутри кластера. Для доступа снаружи нужны другие типы сервисов, например NodePort или LoadBalancer, либо Ingress‑контроллер.

Для локальной разработки с Minikube часто применяют порт‑форвардинг уже к сервису, а не к конкретному поду:

```bash
kubectl port-forward svc/hello-service 8080:80
```

Теперь переходя по адресу http://localhost:8080, ты попадаешь в сервис, который балансирует трафик между несколькими подами. Если один под будет удалён или упадёт, сервис продолжит работать, потому что будет перенаправлять запросы на оставшиеся реплики.

## ConfigMap и Secret: конфигурация приложений в Kubernetes

В микросервисах на Spring Boot часто есть application.yml, где прописаны конфиги, и переменные окружения, которые переопределяют значения. В Kubernetes хорошей практикой считается не зашивать чувствительные данные и окруженческие настройки прямо в образ. Вместо этого создаются ConfigMap для обычной конфигурации и Secret для чувствительной информации.

ConfigMap хранит неконфиденциальные данные: URL внешних сервисов, флаги фич, нестандартные порты. Секрет хранит пароли, токены, ключи. Внутри кластера они могут быть смонтированы как файлы или проброшены как переменные окружения.

Пример ConfigMap для простого приложения:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  SPRING_PROFILES_ACTIVE: docker
  APP_LOG_LEVEL: INFO
```

Здесь создаётся ConfigMap с именем app‑config, содержащий две пары ключ‑значение. Эти значения можно использовать как переменные окружения внутри контейнера. Для этого Deployment нужно дополнить секцией envFrom:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: youdzhin/my-app:latest
          envFrom:
            - configMapRef:
                name: app-config
```

Эта версия Deployment берёт все пары из ConfigMap app‑config и автоматически выставляет их как переменные окружения внутри контейнера. Ошибка новичков здесь — перепутать имя ConfigMap или забыть применить его перед Deployment. В этом случае под может упасть с ошибкой, что нужный ConfigMap не найден. Диагностируется это через kubectl describe pod.

Теперь посмотрим на Secret. В нём данные хранятся в base64. Это не реальное шифрование, но хотя бы маскирует значения при простом просмотре. Настоящая безопасность достигается при использовании внешних хранилищ секретов, но на уровне Kubernetes так устроен базовый механизм.

Пример Secret для пароля к базе данных:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  POSTGRES_USER: cG9zdGdyZXM=
  POSTGRES_PASSWORD: cGFzc3dvcmQ=
```

Здесь значения POSTGRES_USER и POSTGRES_PASSWORD закодированы в base64. Раскодировать их можно командой echo cG9zdGdyZXM= | base64 --decode. Внутри пода они будут доступны как обычные строки postgres и password. Чтобы использовать этот Secret, в Deployment добавляют секцию env:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-with-db
  template:
    metadata:
      labels:
        app: app-with-db
    spec:
      containers:
        - name: app-with-db
          image: youdzhin/app-with-db:latest
          env:
            - name: SPRING_DATASOURCE_URL
              value: jdbc:postgresql://postgres-service:5432/people
            - name: SPRING_DATASOURCE_USERNAME
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: POSTGRES_USER
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: POSTGRES_PASSWORD
```

В этом манифесте Spring‑приложение получает URL базы напрямую через переменную окружения SPRING_DATASOURCE_URL, а имя пользователя и пароль — из Secret. Подключение к сервису postgres‑service описано с использованием DNS‑имени, которое будет выдано объектом Service для базы данных. Ошибка начинающих — пробовать использовать localhost для подключения к базе из контейнера, как привыкли в Docker Compose. В Kubernetes localhost всегда означает сам под, а не базу данных, живущую в другом поде.

## Stateful приложения: пример PostgreSQL со StatefulSet и headless‑сервисом

Статeless‑микросервисы можно свободно пересоздавать и масштабировать. Но базы данных, вроде PostgreSQL, требуют сохранения состояния на диске и часто полагаются на стабильные имена и хосты. Для таких случаев в Kubernetes есть контроллер StatefulSet. Он создаёт поды с фиксированными именами и порядковыми номерами, например postgres‑0, postgres‑1, и связывает их с постоянными томами хранения.

Для StatefulSet почти всегда создаётся headless‑сервис. В обычном ClusterIP‑сервисе есть виртуальный IP, и трафик балансируется между подами. В headless‑сервисе параметр clusterIP устанавливается в None, и к каждому поду создаются отдельные DNS‑записи. Это нужно, чтобы клиент мог обращаться к конкретному поду, например для настройки репликации или sharding.

Начнём с headless‑сервиса для PostgreSQL:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
```

Этот сервис называется postgres и не имеет отдельного виртуального IP (clusterIP: None). Вместо единого IP он публикует DNS‑имена вида postgres‑0.postgres.default.svc.cluster.local, postgres‑1.postgres.default.svc.cluster.local и так далее. Метка selector app: postgres должна совпадать с метками подов StatefulSet.

Теперь определим сам StatefulSet:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16.4
          ports:
            - containerPort: 5432
              name: postgres
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: POSTGRES_PASSWORD
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 5Gi
```

В этом манифесте StatefulSet с именем postgres использует сервис postgres как serviceName. Это связывает поды и headless‑сервис. Метка app: postgres совпадает с селектором сервиса и селектором StatefulSet. Контейнер запускается из образа postgres:16.4, экспортирует порт 5432 и получает пользователя и пароль из Secret db‑credentials. Переменная PGDATA перенастраивает путь, где PostgreSQL хранит свои данные, чтобы было проще монтировать том. В секции volumeMounts имя postgres‑data соответствует имени в volumeClaimTemplates. Каждому поду будет создан свой PersistentVolumeClaim с именем postgres‑data‑postgres‑<номер>, что обеспечит ему собственный диск.

Типовая ошибка начинающих — забыть настроить storage‑класс или использовать непривязанные PVC на кластере, где нет подходящего провайдера хранения. Minikube обычно предоставляет стандартный storage‑class по умолчанию, поэтому на локальной машине всё работает из коробки. Если PersistentVolumeClaim не может быть создан или привязан, под будет висеть в состоянии Pending.

После применения:

```bash
kubectl apply -f postgres-secret.yaml
kubectl apply -f postgres-service.yaml
kubectl apply -f postgres-statefulset.yaml
kubectl get pods
```

Видно, что появляется под postgres‑0. Подключиться к базе можно несколькими способами. Один из них — использовать port‑forward:

```bash
kubectl port-forward pod/postgres-0 5432:5432
```

Теперь любая клиентская программа на твоём ноутбуке, например psql, может подключаться к базе по адресу localhost:5432, но фактически будет работать с PostgreSQL внутри кластера. Другой способ — зайти внутрь пода и использовать psql прямо оттуда:

```bash
kubectl exec -it postgres-0 -- psql -U postgres
```

Флаг -it открывает интерактивную сессию, а команда после двойного дефиса запускается внутри контейнера. После подключения стандартными командами PostgreSQL (\l, \d, \c) можно просматривать базы и таблицы. Важно помнить, что при удалении пода его PersistentVolumeClaim и связанные данные останутся, поэтому при следующем создании postgres‑0 подцепит существующие данные.

## Порт‑форвардинг и диагностика: как понять, что пошло не так

При работе с Kubernetes основное средство диагностики — это команды describe и logs. Если под не запускается или падает, первым делом стоит посмотреть его события.

Чтобы посмотреть общую информацию о поде, включая состояние контейнеров и события, используется:

```bash
kubectl describe pod postgres-0
```

В выводе видно, какие контейнеры входят в под, какие переменные окружения они получили, какие тома монтируются и какие события происходили. Если, например, образ не найден, будет сообщение о проблеме ImagePullBackOff. Если контейнер завершается с ошибкой, будет статус CrashLoopBackOff с описанием кода выхода.

Логи контейнера можно посмотреть так:

```bash
kubectl logs postgres-0
```

Если в поде несколько контейнеров, нужно указать имя контейнера через параметр -c. Это особенно важно в сложных подах, где работают sidecar‑контейнеры, вроде прокси или логгеров. При отладке приключений Java‑приложения на Kubernetes kubectl logs — это твой стандартный заменитель вывода из docker logs.

Порт‑форвардинг используется не только для тестового hello‑world, но и для доступа к административным панелям. В заметках у тебя есть пример:

```bash
kubectl port-forward svc/rabbitmq 15672:15672
```

Эта команда привязывает локальный порт 15672 к порту такого же номера на сервисе rabbitmq. После этого можно открыть панель управления RabbitMQ в браузере по адресу http://localhost:15672. Важно понимать, что порт‑форвардинг привязан к конкретной сессии терминала. Как только команду остановишь, доступ пропадёт. Это нормально для локальной отладки, но не подходит для постоянного внешнего доступа.

Частая ошибка — пытаться пробросить порт на под, который не запущен или не слушает указанный порт. В этом случае kubectl port-forward завершится с ошибкой или будет висеть без отдачи данных. В таких ситуациях стоит проверить состояние пода и убедиться, что контейнер внутри действительно слушает нужный порт.

## Headless‑сервисы: когда тебе нужен прямой доступ к конкретным подам

Headless‑сервис отличается от обычного сервиса типом адресации. Вместо единого виртуального IP и балансировки он отдаёт DNS‑записи для каждого пода. Это полезно для stateful‑систем, вроде PostgreSQL‑кластера, RabbitMQ‑кластера или Kafka‑кластера, где компоненты должны видеть друг друга по предсказуемым хостнеймам.

Схематично можно представить так. Обычный сервис типа ClusterIP выглядит как один логический хост:

приложение отправляет запросы к имени hello‑service, а Kubernetes сам решает, в какой из подов их доставить. В случае headless‑сервиса, например postgres, DNS‑запрос postgres.default.svc.cluster.local вернёт список IP‑адресов всех подов StatefulSet, а запрос postgres‑0.postgres.default.svc.cluster.local вернёт IP конкретного пода. Это позволяет клиентским библиотекам, которые поддерживают репликацию и sharding, работать корректно.

Если забыть указать clusterIP: None, сервис станет обычным ClusterIP, и клиент не сможет адресовать конкретные поды по предсказуемым именам. Для баз с репликацией это критично, поэтому для них почти всегда используют headless‑сервисы.

## RabbitMQ в Kubernetes: от простого deployment до подключения из микросервисов

RabbitMQ уже знаком тебе по Docker Compose, где ты поднимаешь контейнер с web‑панелью и стандартными портами 5672 и 15672. Перенос этой логики в Kubernetes начинается с Deployment и Service. Сначала определим Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rabbitmq
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      containers:
        - name: rabbitmq
          image: rabbitmq:3.9.11-management-alpine
          ports:
            - name: amqp
              containerPort: 5672
            - name: http
              containerPort: 15672
          env:
            - name: RABBITMQ_DEFAULT_USER
              value: guest
            - name: RABBITMQ_DEFAULT_PASS
              value: guest
```

Здесь поднимается контейнер rabbitmq с management‑панелью. Порты 5672 и 15672 объявлены как доступные внутри пода. Имя образа совпадает с тем, что ты использовал в Docker Compose. Переменные окружения задают дефолтные логин и пароль.

Теперь создадим сервис для доступа к брокеру:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
spec:
  selector:
    app: rabbitmq
  ports:
    - name: amqp
      port: 5672
      targetPort: 5672
    - name: http
      port: 15672
      targetPort: 15672
  type: ClusterIP
```

Этот сервис отдаёт два порта: один для AMQP‑протокола, другой для веб‑интерфейса. Внутри кластера любой микросервис может подключиться к брокеру по адресу rabbitmq:5672. Снаружи, для локальной разработки, удобно использовать port‑forward к сервису:

```bash
kubectl port-forward svc/rabbitmq 15672:15672
```

После этого панель RabbitMQ открывается в браузере по адресу http://localhost:15672. Логин и пароль соответствуют значениям, указанным в Deployment. Важно помнить, что это временный доступ. Для постоянного внешнего доступа в проде обычно используют NodePort или Ingress.

Типичная ошибка начинающего Java‑разработчика в Kubernetes — прописывать адрес брокера как localhost:5672 внутри Spring‑конфигурации. В Kubernetes localhost — это сам под, а не внешний сервис. Вместо этого в application.yml нужно использовать адрес rabbitmq:5672 или, если сервис в другом namespace, полное DNS‑имя.

## Kafka в Kubernetes: базовый сценарий для разработки

Kafka сложнее RabbitMQ, потому что требует координации брокеров, зоопарка сервисов и хранилища. В последних версиях Kafka можно работать в режиме KRaft без ZooKeeper, что упрощает запуск. Локально это часто делают через Docker, как в твоей заметке, командами docker pull apache/kafka и docker run. В Kubernetes чаще используют готовые чарты Helm или операторы, но для понимания принципов полезно посмотреть на простой Deployment и Service.

Пример упрощённого Deployment для одного брокера Kafka (для учебных целей):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
        - name: kafka
          image: apache/kafka:3.9.0
          ports:
            - containerPort: 9092
              name: kafka
          env:
            - name: KAFKA_LISTENERS
              value: PLAINTEXT://:9092
            - name: KAFKA_ADVERTISED_LISTENERS
              value: PLAINTEXT://kafka:9092
            - name: KAFKA_PROCESS_ROLES
              value: broker
            - name: KAFKA_NODE_ID
              value: "1"
            - name: KAFKA_CONTROLLER_QUORUM_VOTERS
              value: "1@kafka:9093"
            - name: KAFKA_LISTENER_SECURITY_PROTOCOL_MAP
              value: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
            - name: KAFKA_INTER_BROKER_LISTENER_NAME
              value: PLAINTEXT
```

Этот пример показывает базовые переменные для работы в режиме KRaft. Переменная KAFKA_ADVERTISED_LISTENERS говорит клиентам, как обращаться к брокеру. Внутри кластера этот адрес kafka:9092 должен совпадать с именем сервиса, который мы создадим. Для учебных целей один брокер достаточно, но в реальной системе их будет несколько, и здесь уже нужен StatefulSet и headless‑сервис.

Сервис для доступа к брокеру может быть таким:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka
spec:
  selector:
    app: kafka
  ports:
    - name: kafka
      port: 9092
      targetPort: 9092
  type: ClusterIP
```

После применения микросервисы внутри кластера смогут подключаться к Kafka по адресу kafka:9092. Spring‑конфигурация, которую ты уже использовал локально для Kafka, в Kubernetes будет выглядеть примерно так: bootstrap‑servers=kafka:9092. Если использовать localhost:9092, приложение внутри пода не сможет достучаться до брокера.

При отладке Kafka внутри Kubernetes можно также использовать port‑форвардинг:

```bash
kubectl port-forward svc/kafka 9092:9092
```

Теперь локальный продюсер и консюмер из твоего ноутбука могут подключаться к брокеру, запущенному внутри кластера, по адресу localhost:9092. Важно помнить, что advertised listeners должны быть настроены так, чтобы брокер корректно работал как с внутренними, так и с внешними клиентами. Неправильная конфигурация этих параметров — одна из самых частых причин, по которой продюсер не может отправить сообщения, хотя порт формально открыт.

## Практический сценарий: развёртывание Spring Boot‑сервиса в Minikube с PostgreSQL

Теперь соберём всё воедино и рассмотрим сценарий, близкий к твоему стеку. Допустим, у тебя есть Spring Boot‑сервис, который использует PostgreSQL для хранения пользователей. Локально ты запускал его через Docker Compose с контейнером postgres:16.4 и приложением, которое подключается к базе по jdbc:postgresql://localhost:5432/people. Перенесём это в Kubernetes.

Сначала создаём Secret для базы:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  POSTGRES_USER: cG9zdGdyZXM=
  POSTGRES_PASSWORD: cGFzc3dvcmQ=
```

Затем поднимаем PostgreSQL через StatefulSet и headless‑сервис, как в примере выше. После этого создаём Deployment и сервис для Spring Boot‑приложения.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: customer-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: customer-service
  template:
    metadata:
      labels:
        app: customer-service
    spec:
      containers:
        - name: customer-service
          image: youdzhin/customer-service:latest
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_DATASOURCE_URL
              value: jdbc:postgresql://postgres-0.postgres:5432/people
            - name: SPRING_DATASOURCE_USERNAME
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: POSTGRES_USER
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: POSTGRES_PASSWORD
```

В этой конфигурации приложение подключается к базе данных по адресу postgres‑0.postgres:5432. Хост postgres‑0.postgres — это DNS‑имя первого пода StatefulSet postgres, зарегистрированное headless‑сервисом postgres. Именно поэтому нам был нужен такой тип сервиса. Если бы мы использовали обычный ClusterIP‑сервис, адресом подключения было бы что‑то вроде postgres.default.svc.cluster.local, и для одного экземпляра базы этого было бы достаточно. В реальных сценариях с репликацией headless‑сервис даёт больше гибкости.

Создадим сервис для доступа к нашему микросервису:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: customer-service
spec:
  selector:
    app: customer-service
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

Внутри кластера другие сервисы смогут обращаться к нашему приложению по адресу customer‑service:80. Для локальной отладки в Minikube проще всего пробросить порт:

```bash
kubectl port-forward svc/customer-service 8080:80
```

Теперь API Spring Boot‑сервиса доступен по адресу http://localhost:8080. Все миграции Flyway, настройки JPA и прочий Java‑стек будут работать так же, как и при запуске в Docker, но инфраструктурой теперь управляет Kubernetes.

Частые ошибки на этом шаге связаны с путаницей в адресах базы. Если в переменной SPRING_DATASOURCE_URL оставить localhost, приложение не сможет подключиться к PostgreSQL и будет падать с ошибкой connection refused. Ещё одна распространённая проблема — забытый профиль Spring или неверно проброшенные значения в ConfigMap и Secret. Команда kubectl logs для пода приложения обычно сразу показывает stacktrace и точную причину ошибки.

## Частые ошибки и типичные заблуждения начинающих

В процессе перехода от Docker Compose к Kubernetes многие разработчики совершают похожие ошибки. Первая и самая распространённая — попытка относиться к подам как к постоянным серверам. На самом деле под — это временный объект, который может быть удалён и создан заново в любой момент. Нельзя хранить состояние внутри файловой системы контейнера без использования томов, потому что при пересоздании пода все внутренние файлы исчезнут.

Вторая ошибка — использование localhost для подключения к другим сервисам. В Kubernetes localhost всегда указывает на текущий под. Чтобы связать микросервисы, нужно использовать имена сервисов, которые создаёт Kubernetes. Например, rabbitmq:5672 для RabbitMQ, kafka:9092 для Kafka, postgres‑0.postgres:5432 для PostgreSQL. Эти имена резолвятся во внутренние IP‑адреса через встроенный DNS.

Третья ошибка — хранение всех конфигов и паролей внутри образа. Такой подход опасен в продакшене и неудобен в управлении окружениями. Вместо этого нужно использовать ConfigMap и Secret, как мы рассмотрели выше. Это позволяет для одного и того же Docker‑образа задавать разные конфигурации и креды в зависимости от namespace и кластера.

Четвёртое заблуждение — ожидание, что kubectl сразу выдаст человекопонятную причину любой проблемы. На практике иногда нужно сочетать несколько инструментов: describe, logs, просмотр событий на уровне нод и даже вход по SSH в Minikube. Но для повседневной Java‑разработки почти всегда достаточно describe и logs.

Наконец, многие начинающие пытаются использовать kubectl run и kubectl expose как основной способ развертывания. Эти команды подходят для быстрого прототипирования, но для реальных проектов стоит всегда держать YAML‑манифесты в Git, как часть инфраструктурного кода. Тогда развёртывание становится повторяемым, и его можно автоматизировать через CI/CD, как ты уже делаешь для Docker‑образов и GitHub Actions.

## Заключение: что делать дальше для практики

После того как ты понял базовые сущности Kubernetes и научился поднимать Minikube, пользоваться kubectl, создавать Deployment, Service, ConfigMap, Secret и StatefulSet, логичный следующий шаг — обернуть весь свой микросервисный стек в набор манифестов. Каждый сервис, который у тебя уже работает через Docker Compose, можно перенести в Kubernetes одну за другой, начиная с простых stateless‑приложений и заканчивая PostgreSQL, RabbitMQ и Kafka.

Хорошая практическая тренировка — взять свой существующий проект с Eureka, Spring Cloud Gateway, RabbitMQ, Kafka и PostgreSQL и описать все эти компоненты в отдельном namespace Kubernetes. Для каждого сервиса определить Deployment и Service, для базы — StatefulSet и headless‑сервис, для конфигураций — ConfigMap и Secret. Затем подключить kubectl apply к твоему CI/CD, чтобы автоматически раскатывать новые версии образов, которые ты уже умеешь собирать с помощью Jib и GitHub Actions.

Важно помнить, что Kubernetes сам по себе не делает приложение надёжным и безопасным. Он даёт инструменты: автолечение, масштабирование, сетевую адресацию и управление конфигурацией. Но архитектурные решения, выбор стратегий деплоя, правильное использование брокеров сообщений и базы данных зависят от тебя. Если продолжать отрабатывать сценарии, разбирать типичные ошибки и смотреть, как твой стек ведёт себя в Kubernetes под нагрузкой, через какое‑то время кластер перестанет казаться магией и станет такой же привычной частью проекта, как pom.xml и application.yml.


