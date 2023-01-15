apiVersion: kubegres.reactive-tech.io/v1
kind: Kubegres
metadata:
name: mypostgres
namespace: default
annotations:
imageregistry: "https://hub.docker.com/"
nginx.ingress.kubernetes.io/affinity: cookie

spec:

replicas: 3
image: postgres:14.1
port: 5432

imagePullSecrets:
- name: my-private-docker-repo-secret

database:
size: 200Mi
storageClassName: standard
volumeMount: /var/lib/postgresql/data

customConfig: mypostgres-conf

failover:
isDisabled: false
promotePod: "mypostgres-2-0"

backup:
schedule: "* */1 * * *"
pvcName: my-backup-pvc
volumeMount: /var/lib/backup

securityContext:
runAsNonRoot: true
runAsUser: 999
fsGroup: 999
...

resources:
limits:
memory: "4Gi"
cpu: "1"
requests:
memory: "2Gi"
cpu: "1"

scheduler:
affinity:
podAntiAffinity:
preferredDuringSchedulingIgnoredDuringExecution:
- weight: 100
podAffinityTerm:
labelSelector:
matchExpressions:
- key: app
operator: In
values:
- postgres-name
topologyKey: "kubernetes.io/hostname"
tolerations:
- key: group
operator: Equal
value: critical

volume:
volumes:
- name: dshm
emptyDir:
medium: Memory
sizeLimit: "600Mi"

volumeMounts:
- mountPath: /dev/shm
name: dshm
- mountPath: /dev/anything
name: anyMount

volumeClaimTemplates:
name: anyMount
spec:
accessModes: [ "ReadWriteOnce" ]
storageClassName: standard
resources:
requests:
storage: 1Gi

probe:
livenessProbe:
exec:
command:
- sh
- -c
- exec pg_isready -U postgres -h $POD_IP
failureThreshold: 10
initialDelaySeconds: 60
periodSeconds: 20
successThreshold: 1
timeoutSeconds: 15

readinessProbe:
exec:
command:
- sh
- -c
- exec pg_isready -U postgres -h $POD_IP
failureThreshold: 3
initialDelaySeconds: 5
periodSeconds: 5
successThreshold: 1
timeoutSeconds: 3

env:
- name: POSTGRES_PASSWORD
valueFrom:
secretKeyRef:
name: mySecretResource
key: superUserPassword

- name: POSTGRES_REPLICATION_PASSWORD
valueFrom:
secretKeyRef:
name: mySecretResource
key: replicationUserPassword
