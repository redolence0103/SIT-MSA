## AWS serviceAccount 생성

### iam-policy.json
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": "arn:aws:s3:::atid-tdb"
        },
        {
            "Sid": "List",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion"
            ],
            "Resource": "arn:aws:s3:::atid-tdb/*"
        }
    ]
}
```
### pod-copy-s3 라는 policy 생성
```bash
aws iam create-policy --policy-name pod-copy-s3 --policy-document file://iam-policy.json
```
### eksctl을 사용하여 서비스 계정에 대한 IAM 역할을 생성합니다.
```bash
eksctl create iamserviceaccount \
  --name pod-s3 \
  --namespace analysis \
  --cluster mrn-cluster \
  --attach-policy-arn arn:aws:iam::855234652597:policy/pod-copy-s3 \
  --approve \
```
```text
2022-10-23 12:44:59 [ℹ]  7 existing iamserviceaccount(s) (default/iam-pod,default/iam-s3-role,default/influxdb-s3-role,default/pod-role,default/restore-s3-role,default/s3-restore-role,default/s3read-role) will be excluded
2022-10-23 12:44:59 [ℹ]  1 iamserviceaccount (analysis/app) was included (based on the include/exclude rules)
2022-10-23 12:44:59 [!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
2022-10-23 12:44:59 [ℹ]  1 task: { 
    2 sequential sub-tasks: { 
        create IAM role for serviceaccount "analysis/app",
        create serviceaccount "analysis/app",
    } }2022-10-23 12:44:59 [ℹ]  building iamserviceaccount stack "eksctl-mrn-cluster-addon-iamserviceaccount-analysis-app"
2022-10-23 12:44:59 [ℹ]  deploying stack "eksctl-mrn-cluster-addon-iamserviceaccount-analysis-app"
2022-10-23 12:44:59 [ℹ]  waiting for CloudFormation stack "eksctl-mrn-cluster-addon-iamserviceaccount-analysis-app"
2022-10-23 12:45:29 [ℹ]  waiting for CloudFormation stack "eksctl-mrn-cluster-addon-iamserviceaccount-analysis-app"
2022-10-23 12:46:12 [ℹ]  waiting for CloudFormation stack "eksctl-mrn-cluster-addon-iamserviceaccount-analysis-app"
2022-10-23 12:46:12 [ℹ]  created serviceaccount "analysis/app"
```
## EKS에서 serviceAccount 확인
```bash
kubectl get serviceaccount -n analysis
```
```text
NAME      SECRETS   AGE
app       1         21s
default   1         19h
```
```bash
kubectl get serviceaccount app -o yaml
```
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::855234652597:role/eksctl-mrn-cluster-addon-iamserviceaccount-a-Role1-1WO5V20CPEQ48
  creationTimestamp: "2022-10-23T12:46:12Z"
  labels:
    app.kubernetes.io/managed-by: eksctl
  name: app
  namespace: analysis
  resourceVersion: "15610786"
  uid: d5459f8a-9f14-492e-a432-032eb25961d3
secrets:
- name: app-token-b2rkg
```
## influxDB image에 aws cli 설치
### Dockerfile 생성
```Dockerfile
FROM influxdb:1.8.10
COPY aws.tgz aws.tgz
RUN  tar xzf aws.tgz
RUN  aws/install
RUN  rm aws.tgz
ENV  TZ=Asia/Seoul
RUN  ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
```
### aws cli 실행 파일을 다운 받아 .zip 파일을 .tgz로 변경 (influxdb image에 zip이 설치되어있지 않음)
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscli2.zip"
unzip awscli2.zip
tar czf aws.tgz aws/
docker build -t zasmin/influxdb:sit .
docker push zasmin/influxdb:sit
```
## influxdb를 pod로 실행하기 위한 yaml file
- influxdb-pod.yaml
  command field 부분의 aws s3 cp 뒤 부분에 대상 파일로 교체하고, -db 뒤 부분은 백업된 database 이름으로 변경합니다.
```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: analysis
  name: influx-analysis-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
  storageClassName: gp2
---
apiVersion: v1
kind: Service
metadata:
  namespace: analysis
  name: influxdb
spec:
  ports:
  - port: 8086
    name: server
  selector:
    run: influxdb
---
apiVersion: v1
kind: Pod
metadata:
  namespace: analysis
  labels:
    run: influxdb
  name: influxdb
spec:
  serviceAccount: app
  containers:
  - image: zasmin/influxdb:sit
    name: influxdb
    lifecycle:
      postStart:
        exec:
          command:
            - /bin/sh
            - "-c"
            - until curl -s http://localhost:8086/ping; do sleep 1; done; aws s3 cp s3://atid-tdb/noadb.tgz noadb.tgz; tar xzf noadb.tgz -C /backup; sleep 1; influxd restore -online -db NOAA_water_database /backup
    env:
    - name: INFLUXDB_USERNAME
      value: admin
    - name: INFLUXDB_PASSWORD
      value: admin
    volumeMounts:
    - mountPath: /backup
      name: backup-volume
    resources:
      limits:
        memory: "2048Mi"
        cpu: "2"
    ports:
    - containerPort: 8086
  volumes:
  - name: backup-volume
    persistentVolumeClaim:
      claimName: influx-analysis-data
```
## influxdb 시각화툴
### chronograf 설치
- chronograf.yaml
```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: analysis
  name: chronograf
  labels:
    app: chronograf
    component: chronograf
data:
  monitor.src: |-
    {
      "id": "5000",
      "name": "internal",
      "url": "http://influxdb:8086",
      "type": "influx",
      "insecureSkipVerify": false,
      "default": true,
      "organization": "influx"
    }
---
apiVersion: v1
kind: Service
metadata:
  namespace: analysis
  name: chronograf
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8888
    name: server
  selector:
    app: chronograf
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: analysis
  name: chronograf
  labels:
    app: chronograf
    component: chronograf
spec:
  strategy:
    type: "Recreate"
  selector:
    matchLabels:
      app: chronograf
  replicas: 1
  template:
    metadata:
      name: chronograf
      labels:
        app: chronograf
        component: chronograf
    spec:
      containers:
      - name: chronograf
        image:  quay.io/influxdb/chronograf:nightly
        env:
          - name: RESOURCES_PATH
            value: "/usr/share/chronograf/resources"
          - name: LOG_LEVEL
            value: "error"
        ports:
          - containerPort: 8888
            name: server
        volumeMounts:
          - name: data
            mountPath: /var/lib/chronograf
          - name: config
            mountPath: /usr/share/chronograf/resources
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: chronograf
      - name: config
        configMap:
          name: chronograf
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  namespace: analysis
  name: chronograf
  labels:
    app: chronograf
    component: chronograf
spec:
  storageClassName: gp2
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: 1Gi
---
```
### grafana 설치(datasource 설정 필요)
- grafana.yaml
```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana
  labels:
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.2.1"
data:
  allow-snippet-annotations: "false"
  grafana.ini: |
    [analytics]
    check_for_updates = true
    [grafana_net]
    url = https://grafana.net
    [log]
    mode = console
    [paths]
    data = /var/lib/grafana/
    logs = /var/log/grafana
    plugins = /var/lib/grafana/plugins
    provisioning = /etc/grafana/provisioning

  datasources.yaml: |
    apiVersion: 1
    datasources:
    - access: proxy
      editable: true
      isDefault: true
      jsonData:
        timeInterval: 5s
      name: InfluxDB
      orgId: 1
      type: influxdb
      url: http://influxdb:8086
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  labels:
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.2.1"
spec:
  type: LoadBalancer
  ports:
    - name: service
      port: 3000
      protocol: TCP
      targetPort: 3000

  selector:
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  labels:
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.2.1"
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/name: grafana
      app.kubernetes.io/instance: grafana
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app.kubernetes.io/name: grafana
        app.kubernetes.io/instance: grafana
        app: grafana
      annotations:
    spec:
      securityContext:
        fsGroup: 472
        runAsGroup: 472
        runAsUser: 472
      enableServiceLinks: true
      containers:
        - name: grafana
          image: "grafana/grafana:9.2.1"
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: config
              mountPath: "/etc/grafana/grafana.ini"
              subPath: grafana.ini
            - name: storage
              mountPath: "/var/lib/grafana"
          ports:
            - name: service
              containerPort: 3000
              protocol: TCP
            - name: grafana
              containerPort: 3000
              protocol: TCP
          env:
            - name: GF_PATHS_DATA
              value: /var/lib/grafana/
            - name: GF_PATHS_LOGS
              value: /var/log/grafana
            - name: GF_PATHS_PLUGINS
              value: /var/lib/grafana/plugins
            - name: GF_PATHS_PROVISIONING
              value: /etc/grafana/provisioning
            - name: "GF_AUTH_ANONYMOUS_ENABLED"
              value: "true"
            - name: "GF_AUTH_ANONYMOUS_ORG_ROLE"
              value: "Admin"
            - name: "GF_AUTH_BASIC_ENABLED"
              value: "false"
            - name: "GF_SECURITY_ADMIN_PASSWORD"
              value: "-"
            - name: "GF_SECURITY_ADMIN_USER"
              value: "-"
          livenessProbe:
            failureThreshold: 10
            httpGet:
              path: /api/health
              port: 3000
            initialDelaySeconds: 60
            timeoutSeconds: 30
          readinessProbe:
            httpGet:
              path: /api/health
              port: 3000
          resources:
            {}
      volumes:
        - name: config
          configMap:
            name: grafana
        - name: storage
          emptyDir: {}
---
```
