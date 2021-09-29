# RabbitMQ Install & Config

## Create NFS Share

From p0:
```shell=
sudo salt 'p*' file.mkdir /data/nfs/rabbit
sudo salt 'p*' file.chown /data/nfs/rabbit ubuntu ubuntu

# local p0 filesystem
chmod 777 /data/nfs/rabbit/ # permissions too loose

# setting permissions not tested w/salt yet
# sudo salt 'p*' file.set_perms /data/nfs/rabbit "{...}" 

# add entry to /etc/exports
sudo echo "/data/nfs/rabbit		*(rw,sync,subtree_check)" >> /etc/exports

sudo exportfs -a
```

## Mount Share on Workers
From p0:
```shell=
sudo salt 'w*' file.mkdir /data/nfs/rabbit
sudo salt 'w*' file.chown /data/nfs/rabbit ubuntu ubuntu

sudo salt 'w*' mount.mount /data/nfs/rabbit 10.0.0.190:/data/nfs/rabbit
```

## Create PV
```shell=
cat << EOF > rabbit-pv.yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: rabbit-pv
spec:
  capacity:
    storage: 8Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /data/nfs/rabbit
    server: 10.0.0.190
EOF

kubectl apply -f rabbit-pv.yml
```

## Install via Helm
```shell=
export RABBITMQ_PASSWORD=$(kubectl get secret --namespace "rabbit" rabbit-rabbitmq -o jsonpath="{.data.rabbitmq-password}" | base64 --decode)

export RABBITMQ_ERLANG_COOKIE=$(kubectl get secret --namespace "rabbit" rabbit-rabbitmq -o jsonpath="{.data.rabbitmq-erlang-cookie}" | base64 --decode)

helm upgrade rabbit bitnami/rabbitmq \
  --create-namespace \
  --install \
  --namespace rabbit \
  -f ./custom-rabbit-values.yml \
  --debug
```

## Configure Plugins & Permissions

```shell=
kubectl exec -it rabbit-rabbitmq-0 -nrabbit -- sh

> rabbitmq-plugins enable rabbitmq_management
> rabbitmq-plugins list

> rabbitmqctl add_user <user> <passwd>
> rabbitmqctl set_permissions <user> configure write read
> rabbitmqctl set_user_tags <user> administrator
```

## UI
- The `rabbit-rabbitmq` service for the UI is created as a LoadBalancer via port `15672`

## Notes
Credentials:

```shell=
echo "Username      : user"

echo "Password      : $(kubectl get secret --namespace rabbit rabbit-rabbitmq -o jsonpath="{.data.rabbitmq-password}" | base64 --decode)"

echo "ErLang Cookie : $(kubectl get secret --namespace rabbit rabbit-rabbitmq -o jsonpath="{.data.rabbitmq-erlang-cookie}" | base64 --decode)"
```
