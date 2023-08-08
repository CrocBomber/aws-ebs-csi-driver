# Вывод логов

```sh
tail -f -n 20 $(find /var/log/pods/kube-system_ebs-csi-controller-* -type l | xargs)
```

# Тестирование кастомных сборок

Для тестирования своей сборки нужно настроить стенд на свой реестр образов и S3 бакет.
Ниже - перечень шагов для подготовки стенда для тестирования.

1. В реестре образов https://registry.hosting.croc.ru должен быть свой неймспейс, например awesomedev, сделать его публичным.

2. Скопировать из неймспейса kaas в свой неймспейс все образы.

3. Из файла charts/aws-ebs-csi-driver/values.yaml выписать все образы с версиями и скопировать их в свой неймспейс используя утилиту scopeo.
Пример для версии v1.21:
```sh
skopeo copy --dcreds 'awesomedev:secretpassword' docker://public.ecr.aws/eks-distro/kubernetes-csi/node-driver-registrar:v2.8.0-eks-1-27-7 docker://registry.cloud.croc.ru/awesomedev/node-driver-registrar:v2.8.0-eks-1-27-7
skopeo copy --dcreds 'awesomedev:secretpassword' docker://public.ecr.aws/eks-distro/kubernetes-csi/livenessprobe:v2.10.0-eks-1-27-7 docker://registry.cloud.croc.ru/awesomedev/livenessprobe:v2.10.0-eks-1-27-7
skopeo copy --dcreds 'awesomedev:secretpassword' docker://public.ecr.aws/eks-distro/kubernetes-csi/external-attacher:v4.3.0-eks-1-27-7 docker://registry.cloud.croc.ru/awesomedev/external-attacher:v4.3.0-eks-1-27-7
skopeo copy --dcreds 'awesomedev:secretpassword' docker://public.ecr.aws/eks-distro/kubernetes-csi/external-provisioner:v3.5.0-eks-1-27-7 docker://registry.cloud.croc.ru/awesomedev/external-provisioner:v3.5.0-eks-1-27-7
skopeo copy --dcreds 'awesomedev:secretpassword' docker://public.ecr.aws/eks-distro/kubernetes-csi/external-resizer:v1.8.0-eks-1-27-7 docker://registry.cloud.croc.ru/awesomedev/external-resizer:v1.8.0-eks-1-27-7
skopeo copy --dcreds 'awesomedev:secretpassword' docker://public.ecr.aws/eks-distro/kubernetes-csi/external-snapshotter/csi-snapshotter:v6.2.2-eks-1-27-7 docker://registry.cloud.croc.ru/awesomedev/csi-snapshotter:v6.2.2-eks-1-27-7
skopeo copy --dcreds 'awesomedev:secretpassword' docker://public.ecr.aws/ebs-csi-driver/volume-modifier-for-k8s:v0.1.1 docker://registry.cloud.croc.ru/awesomedev/volume-modifier-for-k8s:v0.1.1
```

4. В файлах Makefile, charts/aws-ebs-csi-driver/values.yaml, deploy/kubernetes/overlays/stable/ecr/kustomization.yaml и deploy/kubernetes/overlays/stable/gcr/kustomization.yaml везде заменить неймспейс kaas на свой (в примере на awesomedev).

5. Сгенерировать деплоймент файлы по шаблонам командой:
```sh
make generate-kustomize
```

6. Сгенерировать файл k_bundle.yaml командой:
```sh
kubectl kustomize ./deploy/kubernetes/overlays/stable/ > ./deploy/kubernetes/overlays/stable/k_bundle.yaml
```

7. Скопировать сгенерированный k_bundle.yaml в склонированный репозиторий [kaas-resource-initializer](https://ghe.cloud.croc.ru/c2/kaas-resource-initializer) в каждую версию кубернетиса: deployment/<версия кубера>/ebs/ebs.yaml

8. Находясь в каталоге со склонированным репозиторием [kaas-resource-initializer](https://ghe.cloud.croc.ru/c2/kaas-resource-initializer), сгенерировать новые конфиги и rpm пакеты командами:
```sh
external_repo=http://172.25.6.71/pub/repos/slices/23.7/7.8/addons/ make
external_repo=http://172.25.6.71/pub/repos/slices/23.7/7.8/addons/ make create
```

9. В своём **продовском** облаке создать новый S3 бакет, например с именем awesomekaas

10. Залить сгенерированные конфиги и rpm пакеты в бакет из kaas-resource-initializer с помощью утилиты aws (реквизиты профиля default к облаку прописаны в ~/.aws/credentials):
```sh
aws --no-verify-ssl  --profile default --endpoint-url https://storage.cloud.croc.ru s3 cp --acl public-read --recursive ./output/ s3://awesomekaas/
```

11. Находясь в каталоге с проектом aws-ebs-csi-driver собрать тестируемый образ, протегировать и залить в свой неймспейс реестра образов.
Пример командя для версии v1.21.0-CROC1:
```sh
docker login registry.cloud.croc.ru/awesomedev
docker buildx build -t aws-ebs-csi-driver .
docker tag aws-ebs-csi-driver registry.cloud.croc.ru/awesomedev/aws-ebs-csi-driver:v1.21.0-CROC1
docker push registry.cloud.croc.ru/awesomedev/aws-ebs-csi-driver:v1.21.0-CROC1
```

12. В конфиге тестового стенда облака /etc/c2.deployment.conf, прописать свои регистри и s3 бакет.
- в секции KUBERNETES поменять docker_registry_namespace с kaas на awesomedev
- в секции KUBERNETES поменять bucket_address с kaas/17 на awesomekaas

13. В MongoDB в коллекции kubernetes.versions поменять настройки на свои:
```js
db["kubernetes.versions"].update({"_id": "base"}, {"$set": {"bucket_address": "https://storage.cloud.croc.ru/awesomekaas", "docker_registry_namespace": "awesomedev"}})
```
Восстановить как было:
```js
db["kubernetes.versions"].update({"_id": "base"}, {"$set": {"bucket_address": "https://storage.cloud.croc.ru/kaas/v17", "docker_registry_namespace": "kaas"}})
```

14. Перезапустить сервисы стенда c2-deploy и все c2-ks-*

15. На стенде в консоли облака создать пользователя для EBS провайдера.

16. На стенде в консоли облака создать новый кубернетес кластер, присвоив SSH ключ и Elastic IP, и активирвоать EBS провайдер с указанием созданного пользователя для EBS провайдера.

17. Если стенд не железный, а dev, то на нём не будет резолвится хост AWS_EC2_ENDPOINT, и его надо прописать вручную.
Для этого надо:
- зайти по ssh на мастер ноду
- изменить файл /tmp/ebs/api_enpoint.yaml добавив секцию с hostAliases и внешним IP адресов dev стенда, чтобы получилось так (этот файл генерируется в коде из репозитория [cloud-init-configs](https://ghe.cloud.croc.ru/c2/cloud-init-configs)):
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: ebs-csi-controller
  namespace: kube-system
spec:
  template:
    spec:
      hostAliases:
        - ip: "here must be external IP address of dev stand"
          hostnames:
            - "api.dev.c2.croc.ru"
      containers:
        - name: ebs-plugin
          env:
            - name: AWS_EC2_ENDPOINT
              value: https://api.dev.c2.croc.ru:8443
            - name: AWS_EC2_ENDPOINT_UNSECURE
              value: "true"
```
- передеплоить ebs провайдер командой:
```sh
kubectl apply -k /tmp/ebs
```

18. На этом всё, кубернетес должен будет подниматься с кастомной сборкой aws-ebs-csi-driver.


# Активация отладчика

Для использования go отладчика delve нужно внести ряд изменений в сборочные и конфигурационные файлы.
Ниже - перечень шагов по изменению и сборки образа с отладчиком. Эти шаги подразумевают, что были выполнены предыдущие необходимые шаги для кастомной сборки.

1. В Dockerfile на этап сборки добавить шаги по сборке и установки отладчика delve и запуска драйвера через отладчик.
На этап сборки добавить строки для скачивания отладчика delve:
```Dockerfile
ENV CGO_ENABLED 0
RUN go install github.com/go-delve/delve/cmd/dlv@latest
```
Где переменная окружение CGO_ENABLED в значении 0 указывает скачать статически собранный delve, что требуется т.к. в docker образе отсутствуют нужные динамические библиотеки.

Изменить строку ENTRYPOINT указав запуск aws-ebs-csi-driver через отладчик:
```Dockerfile
ENTRYPOINT ["/dlv", "--listen=:40000", "--headless", "--accept-multiclient", "--continue", "--api-version=2", "--log", "exec", "/bin/aws-ebs-csi-driver", "--"]
```
Где:
- `--listen=:40000` - указывает запустить отладчик на порту 40000.
- `--headless` - указывает запустить только сервеную часть отладчика.
- `--accept-multiclient` - разрешает принимать множество клиентских соединений.
- `--continue` - разрешает запускаемому процессу выполняться без подключенных клиентов отладчика.
- `--log` - логгирует события отладчика.
- `--api-version=2` - указывает использовать новую версию JSON-RPC API.
- `exec /bin/aws-ebs-csi-driver` - указывает исполняемый файл EBS драйвера для исполнения под отладчиком.
- `--` - разделитель используемый для аргументов исполняемого файла EBS драйвера, которые передаются в docker контейнер.
Пример итогового Dockerfile:
```Dockerfile
FROM --platform=$BUILDPLATFORM golang:1.20 AS builder

ENV CGO_ENABLED 0
RUN go install github.com/go-delve/delve/cmd/dlv@latest

WORKDIR /go/src/github.com/c2devel/aws-ebs-csi-driver
COPY go.* .
ARG GOPROXY
RUN go mod download
COPY . .
ARG TARGETOS
ARG TARGETARCH
ARG VERSION
RUN OS=$TARGETOS ARCH=$TARGETARCH make $TARGETOS/$TARGETARCH

FROM public.ecr.aws/eks-distro-build-tooling/eks-distro-minimal-base-csi-ebs:latest.2 AS linux-amazon
COPY --from=builder /go/src/github.com/c2devel/aws-ebs-csi-driver/bin/aws-ebs-csi-driver /bin/aws-ebs-csi-driver

COPY --from=builder /go/bin/dlv /

ENTRYPOINT ["/dlv", "--listen=:40000", "--headless", "--continue", "--accept-multiclient", "--api-version=2", "--log", "exec", "/bin/aws-ebs-csi-driver", "--"]
```

2. В Makefile убрать флаги сборки `-s -w`, которые убирают отладочные символы из собранного исполняемого файла.
Пример строки сборочных флагов с этими флагами:
```sh
LDFLAGS?="-X ${PKG}/pkg/driver.driverVersion=${VERSION} -X ${PKG}/pkg/cloud.driverVersion=${VERSION} -X ${PKG}/pkg/driver.gitCommit=${GIT_COMMIT} -X ${PKG}/pkg/driver.buildDate=${BUILD_DATE} -s -w"
```
Пример без них (целевой пример):
```sh
LDFLAGS?="-X ${PKG}/pkg/driver.driverVersion=${VERSION} -X ${PKG}/pkg/cloud.driverVersion=${VERSION} -X ${PKG}/pkg/driver.gitCommit=${GIT_COMMIT} -X ${PKG}/pkg/driver.buildDate=${BUILD_DATE}"
```

3. В сгенерированном файле k_bundle.yaml поправить Deployment приложения ebs-csi-controller:
- Указать количество реплик `replicas` равное 1, чтобы отладчик всегда был подключен к той реплике, на которой выполняются действия.
- В аргументах запуска контейнера `ebs-plugin` изменить значение аргумента `--v` с 2 на 10, чтобы повысить детальность вывода в лог.
- В секцию `ports` контейнера `ebs-plugin` добавить порт отладчика 40000: `- containerPort: 40000`
- Увеличить количество оперативной памяти в запрашиваемых ресурсах до 256Mi и в лимитах до 512Mi, т.к. приложение под отладчиком потрбляет больше ресурсов, а с ресурсами по умолчанию выходит за пределы лимитов и убивается out of memory киллером.

Пример секции Deployment целиком:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: aws-ebs-csi-driver
  name: ebs-csi-controller
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ebs-csi-controller
      app.kubernetes.io/name: aws-ebs-csi-driver
  strategy:
    rollingUpdate:
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: ebs-csi-controller
        app.kubernetes.io/name: aws-ebs-csi-driver
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - preference:
              matchExpressions:
              - key: eks.amazonaws.com/compute-type
                operator: NotIn
                values:
                - fargate
            weight: 1
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - ebs-csi-controller
              topologyKey: kubernetes.io/hostname
            weight: 100
      containers:
      - args:
        - --endpoint=$(CSI_ENDPOINT)
        - --logging-format=text
        - --user-agent-extra=kustomize
        - --v=10
        env:
        - name: CSI_ENDPOINT
          value: unix:///var/lib/csi/sockets/pluginproxy/csi.sock
        - name: CSI_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              key: key_id
              name: aws-secret
              optional: true
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              key: access_key
              name: aws-secret
              optional: true
        - name: AWS_EC2_ENDPOINT
          value: https://api.cloud.croc.ru
        - name: AWS_REGION
          value: croc
        envFrom: null
        image: registry.cloud.croc.ru/vladkuznetsov/aws-ebs-csi-driver:v1.19.0-CROC1
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /healthz
            port: healthz
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 3
        name: ebs-plugin
        ports:
        - containerPort: 40000
        - containerPort: 9808
          name: healthz
          protocol: TCP
        readinessProbe:
          failureThreshold: 5
          httpGet:
            path: /healthz
            port: healthz
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 3
        resources:
          limits:
            memory: 512Mi
          requests:
            cpu: 10m
            memory: 256Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
        volumeMounts:
        - mountPath: /var/lib/csi/sockets/pluginproxy/
          name: socket-dir
      - args:
        - --csi-address=$(ADDRESS)
        - --v=2
        - --feature-gates=Topology=true
        - --extra-create-metadata
        - --leader-election=true
        - --default-fstype=ext4
        env:
        - name: ADDRESS
          value: /var/lib/csi/sockets/pluginproxy/csi.sock
        envFrom: null
        image: registry.cloud.croc.ru/vladkuznetsov/external-provisioner:v3.5.0-eks-1-27-3
        imagePullPolicy: IfNotPresent
        name: csi-provisioner
        resources:
          limits:
            memory: 256Mi
          requests:
            cpu: 10m
            memory: 40Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
        volumeMounts:
        - mountPath: /var/lib/csi/sockets/pluginproxy/
          name: socket-dir
      - args:
        - --csi-address=$(ADDRESS)
        - --v=2
        - --leader-election=true
        env:
        - name: ADDRESS
          value: /var/lib/csi/sockets/pluginproxy/csi.sock
        envFrom: null
        image: registry.cloud.croc.ru/vladkuznetsov/external-attacher:v4.3.0-eks-1-27-3
        imagePullPolicy: IfNotPresent
        name: csi-attacher
        resources:
          limits:
            memory: 256Mi
          requests:
            cpu: 10m
            memory: 40Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
        volumeMounts:
        - mountPath: /var/lib/csi/sockets/pluginproxy/
          name: socket-dir
      - args:
        - --csi-address=$(ADDRESS)
        - --leader-election=true
        - --extra-create-metadata
        env:
        - name: ADDRESS
          value: /var/lib/csi/sockets/pluginproxy/csi.sock
        envFrom: null
        image: registry.cloud.croc.ru/vladkuznetsov/csi-snapshotter:v6.2.1-eks-1-27-3
        imagePullPolicy: IfNotPresent
        name: csi-snapshotter
        resources:
          limits:
            memory: 256Mi
          requests:
            cpu: 10m
            memory: 40Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
        volumeMounts:
        - mountPath: /var/lib/csi/sockets/pluginproxy/
          name: socket-dir
      - args:
        - --csi-address=$(ADDRESS)
        - --v=2
        - --handle-volume-inuse-error=false
        env:
        - name: ADDRESS
          value: /var/lib/csi/sockets/pluginproxy/csi.sock
        envFrom: null
        image: registry.cloud.croc.ru/vladkuznetsov/external-resizer:v1.8.0-eks-1-27-3
        imagePullPolicy: IfNotPresent
        name: csi-resizer
        resources:
          limits:
            memory: 256Mi
          requests:
            cpu: 10m
            memory: 40Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
        volumeMounts:
        - mountPath: /var/lib/csi/sockets/pluginproxy/
          name: socket-dir
      - args:
        - --csi-address=/csi/csi.sock
        envFrom: null
        image: registry.cloud.croc.ru/vladkuznetsov/livenessprobe:v2.10.0-eks-1-27-3
        imagePullPolicy: IfNotPresent
        name: liveness-probe
        resources:
          limits:
            memory: 256Mi
          requests:
            cpu: 10m
            memory: 40Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
        volumeMounts:
        - mountPath: /csi
          name: socket-dir
      nodeSelector:
        kubernetes.io/os: linux
        node-role.kubernetes.io/master: ""
      priorityClassName: system-cluster-critical
      securityContext:
        fsGroup: 1000
        runAsGroup: 1000
        runAsNonRoot: true
        runAsUser: 1000
      serviceAccountName: ebs-csi-controller-sa
      tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
      - effect: NoExecute
        operator: Exists
        tolerationSeconds: 300
      - operator: Exists
      volumes:
      - emptyDir: {}
        name: socket-dir
```
Скопировать k_bundle.yaml в склонированный репозиторий [kaas-resource-initializer](https://ghe.cloud.croc.ru/c2/kaas-resource-initializer) в каждую версию кубернетиса: deployment/<версия кубера>/ebs/ebs.yaml, сгенерировать конфигурационные файлы и выполнить заливку в S3 бакет в соответствии с шагами по тестированию кастомной сборки.

4. На стенде в консоли облака в Security Groups открыть порт отладчика 40000

5. В соответствии с шагами по тестированию кастомной сборки выполнить сборку и заливку образа.

6. В соответствии с шагами по тестированию кастомной сборки создать новый kubernetes калстер.

7. Зайти по SSH на мастер ноду кубернетеса и выполнять коману для пробасывания порта отладчика на хост машину:
```sh
kubectl port-forward --address 0.0.0.0 deployment/ebs-csi-controller 40000:40000 --namespace=kube-system
```

8. В IDE vscode в проекте и исходным кодом aws-ebs-csi-driver настроить `launch.json` указав Elastiс IP мастер ноды kubernetes:
```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Remote debug",
            "type": "go",
            "debugAdapter": "dlv-dap",
            "request": "attach",
            "mode": "remote",
            "port": 40000,
            "host": "Elastiс IP мастер ноды kubernetes",
            "substitutePath": [
                { "from": "${workspaceFolder}", "to": "/go/src/github.com/c2devel/aws-ebs-csi-driver" },
            ]
        }
    ],
}
```

9. Запустить отладку в vscode.

10. Предполагаемая входная точка аттачмента диска будет в файле pkg/driver/controller.go в функции ControllerPublishVolume.
В её первой строке можно установить breakpoint для начала отладки аттачмента диска.
