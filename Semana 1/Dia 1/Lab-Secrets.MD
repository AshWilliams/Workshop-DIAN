# Secrets in Kubernetes Demo

### Creating Secrets Using Kubectl and Mounting Secrets as Volume Mounts

We can either create a secret from a literal value or a file. In this case we have put our secret information in a file named access.txt. 

```
$cat access.txt
APP_AUTH_TOKEN=WEj4VmNF755uc9vZdz98zvPXB6DkHp

$ kubectl create secret generic auth-token --from-file=./access.txt
secret "auth-token" created

$kubectl get secrets
NAME                  TYPE                                  DATA      AGE
auth-token            Opaque                                1         14s
default-token-k7vmv   kubernetes.io/service-account-token   3         17m

$ kubectl describe secret auth-token
Name:         auth-token
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
access.txt:  46 bytes
```
Let’s now use this secret in your pod as a mounted volume.
First, we create a demo pod and apply the manifest.
```
$ cat pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
  - name: demo-pod
    image: ubuntu
    command: ["/bin/bash", "-ec", "while :; do echo '.'; sleep 5 ; done"]
    volumeMounts:
    - name: myvolume
      mountPath: "/tmp"
      readOnly: true
  volumes:
  - name: myvolume
    secret:
      secretName: auth-token

$ kubectl apply -f pod.yaml
pod "demo-pod" created

$ kubectl get pods
NAME       READY     STATUS    RESTARTS   AGE
demo-pod   1/1       Running   0          1m
```


Now, if we exec inside this pod, we should be able to find our secret mounted to /tmp directory.
```
$ kubectl exec -it demo-pod /bin/bash
root@demo-pod:/# ls /tmp
access.txt
root@demo-pod:/# cat /tmp/access.txt
APP_AUTH_TOKEN=WEj4VmNF755uc9vZdz98zvPXB6DkHp
root@demo-pod:/# 
```
### Making Secrets Available as Pod Environment Variables

In order to make secrets available as env var, we will create a secret object manifest and apply it. We will first encode our data before putting it in the secret file as plain text. 
```
$ echo WEj4VmNF755uc9vZdz98zvPXB6DkHp | base64
V0VqNFZtTkY3NTV1Yzl2WmR6OTh6dlBYQjZEa0hwCg==

$ cat auth_secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  auth_token: V0VqNFZtTkY3NTV1Yzl2WmR6OTh6dlBYQjZEa0hwCg==

$ kubectl apply -f auth_secret.yaml
secret "mysecret" created

$ kubectl describe secret mysecret
Name:         mysecret
Namespace:    default
Labels:       <none>
Annotations: 
Type:         Opaque

Data
====
auth_token:  31 bytes
```

In order to inject the secret as an env var we will have to access the secret using the key at which it is stored and map it to an env var for the pod. In the following example we do this with another demo pod, demo-pod-2.
```
$ cat pod-2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod-2
spec:
  containers:
  - name: demo-pod-2
    image: ubuntu
    command: ["/bin/bash", "-ec", "while :; do echo '.'; sleep 5 ; done"]
    env:
      - name: APP_AUTH_TOKEN
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: auth_token

$ kubectl apply -f pod-2.yaml
pod "demo-pod-2" created
```

Now, if we exec into demo-pod-2 and check the value for APP_AUTH_TOKEN env var, we should see the decoded value of our secret. 

```
$ kubectl exec -it demo-pod-2 /bin/bash
root@demo-pod-2:/# echo $APP_AUTH_TOKEN
WEj4VmNF755uc9vZdz98zvPXB6DkHp
root@demo-pod-2:/# 
```