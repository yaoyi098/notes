# 自建GIT

决定搭建git后，对开源的webgit做了下调研，最后决定在gitlab和gitea中选一个。

- gitlab： 对标github，生态比较全，自带功能丰富。自带静态页面渲染，自带CI/CD。但是gitlab太重了，镜像拉下来接近1个G，k8s部署要起n个svc。
- [gitea](https://gitea.io/zh-cn/)： 新兴的gitserver，据说是国人写的。小巧轻便，运行开销小，但是CI/CD功能没有gitlab全。

考虑到我的云服务器只有2c8g,50G硬盘空间，最终还是决定部署gitea。如果需要CI/CD功能，正好也可以试验下和其他开源产品的对接。

## 部署 GITEA

不~翻墙~我家联通网居然打不开gitea首页，差评。 

gitea官方提供了各种安装方式。但作为坚定的云原生一员，肯定是docker 或者k8s二选一。

- docker简单方便， docker pull docker run就ok了。

- k8s稍微复杂些，不过官方也给了helm chart，可以一键部署。

纠结了一下，最红还是决定用k8s。一是本身打算转行SRE，想考CKA、CKD，提前熟悉环境肯定是有必要的，边玩边学才是最有效率的。二是我打算用K3s，这样服务器资源占用率也不会很高。

### 0. 准备工作

- k3s环境建立
- helm 安装
- 检查storageclass、ingress。
  
参考文档： chart gitea doc

### 1. 添加gitea chart

```shell
helm repo add gitea-charts https://dl.gitea.io/charts/
helm install gitea gitea-charts/gitea
```

### 2. 查看并修改gitea chart 配置

- 查看gitea chart配置：

```shell
helm show values gitea-charts/gitea 
```

- 更改gitea chart配置： [charts doc](https://gitea.com/gitea/helm-chart/)

在这里我们需要更改的是：域名，ingress配置（k3s默认安装了traefik Ingress），ssh的lb配置（ssh只能使用LB进行转发），默认用户名和密码，以及其他一些配置。

- 原始chatrs values配置：

```yaml
# Default values for gitea.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

clusterDomain: cluster.local

image:
  repository: gitea/gitea
  # Overrides the image tag whose default is the chart appVersion.
  tag: ""
  pullPolicy: Always
  rootless: false # only possible when running 1.14 or later

imagePullSecrets: []

# Security context is only usable with rootless image due to image design
podSecurityContext:
  fsGroup: 1000

containerSecurityContext: {}
#   allowPrivilegeEscalation: false
#   capabilities:
#     drop:
#       - ALL
#   # Add the SYS_CHROOT capability for root and rootless images if you intend to
#   # run pods on nodes that use the container runtime cri-o. Otherwise, you will
#   # get an error message from the SSH server that it is not possible to read from
#   # the repository.
#   # https://gitea.com/gitea/helm-chart/issues/161
#     add:
#       - SYS_CHROOT
#   privileged: false
#   readOnlyRootFilesystem: true
#   runAsGroup: 1000
#   runAsNonRoot: true
#   runAsUser: 1000

# DEPRECATED. The securityContext variable has been split two:
# - containerSecurityContext
# - podSecurityContext.
securityContext: {}

service:
  http:
    type: ClusterIP
    port: 3000
    clusterIP: None
    #loadBalancerIP:
    #nodePort:
    #externalTrafficPolicy:
    #externalIPs:
    #ipFamilyPolicy:
    #ipFamilies:
    loadBalancerSourceRanges: []
    annotations:
  ssh:
    type: ClusterIP
    port: 22
    clusterIP: None
    #loadBalancerIP:
    #nodePort:
    #externalTrafficPolicy:
    #externalIPs:
    #ipFamilyPolicy:
    #ipFamilies:
    #hostPort:
    loadBalancerSourceRanges: []
    annotations:

ingress:
  enabled: false
  # className: nginx
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: git.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - git.example.com
  # Mostly for argocd or any other CI that uses `helm template | kubectl apply` or similar
  # If helm doesn't correctly detect your ingress API version you can set it here.
  # apiVersion: networking.k8s.io/v1

resources: {}
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi

## Use an alternate scheduler, e.g. "stork".
## ref: https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/
##
# schedulerName:

nodeSelector: {}

tolerations: []

affinity: {}

statefulset:
  env: []
    # - name: VARIABLE
    #   value: my-value
  terminationGracePeriodSeconds: 60
  labels: {}
  annotations: {}

persistence:
  enabled: true
  # existingClaim:
  size: 10Gi
  accessModes:
    - ReadWriteOnce
  labels: {}
  annotations: {}
  # storageClass:
  # subPath:

# additional volumes to add to the Gitea statefulset.
extraVolumes:
# - name: postgres-ssl-vol
#   secret:
#     secretName: gitea-postgres-ssl


# additional volumes to mount, both to the init container and to the main
# container. As an example, can be used to mount a client cert when connecting
# to an external Postgres server.
extraVolumeMounts:
# - name: postgres-ssl-vol
#   readOnly: true
#   mountPath: "/pg-ssl"

# bash shell script copied verbatim to the start of the init-container.
initPreScript: ""
#
# initPreScript: |
#   mkdir -p /data/git/.postgresql
#   cp /pg-ssl/* /data/git/.postgresql/
#   chown -R git:git /data/git/.postgresql/
#   chmod 400 /data/git/.postgresql/postgresql.key

# Configure commit/action signing prerequisites
signing:
  enabled: false
  gpgHome: /data/git/.gnupg

gitea:
  admin:
    #existingSecret: gitea-admin-secret
    username: gitea_admin
    password: r8sA8CPHD9!bt6d
    email: "gitea@local.domain"

  metrics:
    enabled: false
    serviceMonitor:
      enabled: false
      #  additionalLabels:
      #    prometheus-release: prom1

  ldap: []
    # - name: "LDAP 1"
    #  existingSecret:
    #  securityProtocol:
    #  host:
    #  port:
    #  userSearchBase:
    #  userFilter:
    #  adminFilter:
    #  emailAttribute:
    #  bindDn:
    #  bindPassword:
    #  usernameAttribute:
    #  publicSSHKeyAttribute:

  # Either specify inline `key` and `secret` or refer to them via `existingSecret`
  oauth: []
    # - name: 'OAuth 1'
    #   provider:
    #   key:
    #   secret:
    #   existingSecret:
    #   autoDiscoverUrl:
    #   useCustomUrls:
    #   customAuthUrl:
    #   customTokenUrl:
    #   customProfileUrl:
    #   customEmailUrl:

  config: {}
  #  APP_NAME: "Gitea: Git with a cup of tea"
  #  RUN_MODE: dev
  #
  #  server:
  #    SSH_PORT: 22
  #
  #  security:
  #    PASSWORD_COMPLEXITY: spec

  additionalConfigSources: []
  #   - secret:
  #       secretName: gitea-app-ini-oauth
  #   - configMap:
  #       name: gitea-app-ini-plaintext

  additionalConfigFromEnvs: []

  podAnnotations: {}

  # Modify the liveness probe for your needs or completely disable it by commenting out.
  livenessProbe:
    tcpSocket:
      port: http
    initialDelaySeconds: 200
    timeoutSeconds: 1
    periodSeconds: 10
    successThreshold: 1
    failureThreshold: 10

  # Modify the readiness probe for your needs or completely disable it by commenting out.
  readinessProbe:
    tcpSocket:
      port: http
    initialDelaySeconds: 5
    timeoutSeconds: 1
    periodSeconds: 10
    successThreshold: 1
    failureThreshold: 3

  # # Uncomment the startup probe to enable and modify it for your needs.
  # startupProbe:
  #   tcpSocket:
  #     port: http
  #   initialDelaySeconds: 60
  #   timeoutSeconds: 1
  #   periodSeconds: 10
  #   successThreshold: 1
  #   failureThreshold: 10

memcached:
  enabled: true
  service:
    port: 11211

postgresql:
  enabled: true
  global:
    postgresql:
      postgresqlDatabase: gitea
      postgresqlUsername: gitea
      postgresqlPassword: gitea
      servicePort: 5432
  persistence:
    size: 10Gi

mysql:
  enabled: false
  root:
    password: gitea
  db:
    user: gitea
    password: gitea
    name: gitea
  service:
    port: 3306
  persistence:
    size: 10Gi

mariadb:
  enabled: false
  auth:
    database: gitea
    username: gitea
    password: gitea
    rootPassword: gitea
  primary:
    service:
      port: 3306
    persistence:
      size: 10Gi

# By default, removed or moved settings that still remain in a user defined values.yaml will cause Helm to fail running the install/update.
# Set it to false to skip this basic validation check.
checkDeprecation: true

```

- 修改Ingress配置，使用traefik作为Ingress。

```yaml
ingress:
  enabled: true
  className: traefik
  annotations: {}
    kubernetes.io/ingress.class: traefik
    kubernetes.io/tls-acme: "true"
  hosts:
    - host: git.yaoyimeng.cn
    paths:
        - path: /
        pathType: Prefix
  tls: []
    - secretName: git-tls
      hosts:
        - git.yaoyimeng.cn
```

-  修改Gitea配置，管理员账号。

```yaml
gitea:
  admin:
    username: xxx
    password: xxx
    email: "yimengyao.cn@gmail.com"
```    
- 修改PGSQL配置，使用PostgreSQL作为数据库。

```yaml
postgresql:
  enabled: true
  global:
    postgresql:
      postgresqlDatabase: gitea
      postgresqlUsername: gitea
      postgresqlPassword: xxx
      servicePort: 5432
  persistence:
    size: 10Gi
mysql:
  enabled: false
mariadb:
  enabled: false
```

- 修改ssh配置，并配置使用`2222`端口。增加LB配置。

```yaml
service:
  ssh:
    type: ClusterIP
    port: 2222
    clusterIP: None
    loadBalancerSourceRanges: []
    annotations:
    #TODO： 增加SSH的LB配置。

```
