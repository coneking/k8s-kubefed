# Kubefed

## Instalar Kubefed

Instalación vía HELM (Instalación en uno de los cluster k8s)

```sh
$ kubectl create ns kube-federation-system
$ helm repo add kubefed-charts https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts
$ helm repo update
$ helm install kubefed-charts/kubefed --name kubefed --version=0.3.0 --namespace kube-federation-system
```

<br>

Verificamos que los pods de kubefed estén activos

```sh
$ kubectl -n kube-federation-system get pods
```

<br>

## Descargar kubefedctl

Kubefedctl es un binario que se utiliza para crear un grupo de clusters k8s y transformar recursos k8s (deployment, service, secret, etc) para el uso de kubefed.


```sh
$ wget https://github.com/kubernetes-sigs/kubefed/releases/download/v0.5.1/kubefedctl-0.5.1-linux-amd64.tgz -O - |tar -xz
$ sudo mv kubefedctl /usr/local/bin
```

<br>

# Registrar Clusters k8s

Es importante tener en cuenta que el registro de clusters depende del archivo kubeconfig, el cual debe tener los clusters necesarios que se quieran federar. Para este ejemplo usaremos un archivo kubeconfig con dos contextos.

```sh
$ kubectl config get-contexts
CURRENT   NAME    CLUSTER         AUTHINFO           NAMESPACE
          kafka   cluster.kafka   kubernetes-kafka   
*         test    cluster.local   kubernetes-admin   
```

<br>

Para agregar contextos de k8s a un cluster Federado ejecutamos lo siguiente

```sh
$ kubefedctl join test --cluster-context test --host-cluster-context test --v=2
$ kubefedctl join kafka --cluster-context kafka --host-cluster-context test --v=2
```

En este ejemplo se crea un cluster federado llamado "test" que se compone del contexto k8s test y el contexto k8s kafka
Verificamos si los cluster k8s están federados

```sh
$ kubectl -n kube-federation-system get kubefedclusters
```

<br>

# Pruebas

## Namespace

Inicialmente crearemos un recurso Namespace y lo federaremos, posteriormente lo que despleguemos en este Namespace se replicará en los clusters federados.

```sh
$ kubectl create ns prueba-fed
```

<br>

Una vez creado el namespace lo debemos replicar en los demás clusters, para eso crearemos un recurso `FederatedNamespace`.

```sh
kubectl apply -f -<<EOF
apiVersion: types.kubefed.io/v1beta1
kind: FederatedNamespace
metadata:
  name: prueba-fed
  namespace: prueba-fed
spec:
  placement:
    clusters:
    - name: test
    - name: kafka
EOF
```

Ahora verificamos si el namespace en ambos cluster k8s.

```sh
$ kubectl get ns prueba-fed --context=test
$ kubectl get ns prueba-fed --context=kafka
```

<br>

# Deployment y Service

En este ejemplo crearemos dos recursos, deployment y service los cuales posteriormente se transformarán para el uso de kubefed.
Crearemos el archivo `deployment.yaml` con la siguiente información

```sh
kind: Deployment
metadata:
   name: echo
spec:
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: echo
        image: k8s.gcr.io/echoserver:1.10
        ports:
        - name: http
          containerPort: 8080
        - name: https
          containerPort: 8443
```
<br>

Crearemos el archivo service.yaml con la siguiente información

```sh
apiVersion: v1
kind: Service
metadata:
  name: echo-svc-lb
spec:
  ports:
  - name: http
    port: 8080
    protocol: TCP
    targetPort: 8080
  - name: https
    port: 8443
    protocol: TCP
    targetPort: 8443
  selector:
    app: echo
  type: LoadBalancer
```

<br>

## Transformación de recursos

Los dos archivos anteriormente creados se deben modificar para que se apliquen entre los clusters federados. <br>
Con la ayuda del binario anteriormente descargado (kubefedctl) transformaremos los dos archivos ejecutando lo siguiente.

```sh
$ kubefedctl federate -f deployment.yaml
---
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: echo
spec:
  placement:
    clusterSelector:
      matchLabels: {}
  template:
    spec:
      selector:
        matchLabels:
          app: echo
      template:
        metadata:
          labels:
            app: echo
        spec:
          containers:
          - image: k8s.gcr.io/echoserver:1.10
            name: echo
            ports:
            - containerPort: 8080
              name: http
            - containerPort: 8443
              name: https
```
Como se muestra al ejecutar el comando devolverá un archivo nuevo de tipo FederatedDeployment agregando un apartado "placement" donde se podrán indicar los cluster a los cuales aplicar este archivo, por defecto se podrá aplicar a todos los cluster federados que tengamos registrados.

<br>

Aplicamos el nuevo archivo de la siguiente manera.

```sh
$ kubefedctl federate -f deployment.yaml |kubectl -n prueba-fed apply -f -
```

<br>

Verificamos que el recurso FederatedDeployment esté creado y posteriormente revisamos si el Deployment y Pod se crearon correctamente en ambos clusters federados.

```sh
$ kubectl -n prueba-fed get federateddeployment 
NAME   AGE
echo   42s

$ kubectl -n prueba-fed get federateddeployment echo -o yaml
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"types.kubefed.io/v1beta1","kind":"FederatedDeployment","metadata":{"annotations":{},"name":"echo","namespace":"prueba-fed"},"spec":{"placement":{"clusterSelector":{"matchLabels":{}}},"template":{"spec":{"selector":{"matchLabels":{"app":"echo"}},"template":{"metadata":{"labels":{"app":"echo"}},"spec":{"containers":[{"image":"k8s.gcr.io/echoserver:1.10","name":"echo","ports":[{"containerPort":8080,"name":"http"},{"containerPort":8443,"name":"https"}]}]}}}}}}
  creationTimestamp: "2020-11-30T19:03:16Z"
  finalizers:
  - kubefed.io/sync-controller
  generation: 1
  name: echo
  namespace: prueba-fed
  resourceVersion: "5187023"
  selfLink: /apis/types.kubefed.io/v1beta1/namespaces/prueba-fed/federateddeployments/echo
  uid: 57de0f0a-64bd-48b6-a64c-f75461109d5b
spec:
  placement:
    clusterSelector:
      matchLabels: {}
  template:
    spec:
      selector:
        matchLabels:
          app: echo
      template:
        metadata:
          labels:
            app: echo
        spec:
          containers:
          - image: k8s.gcr.io/echoserver:1.10
            name: echo
            ports:
            - containerPort: 8080
              name: http
            - containerPort: 8443
              name: https
status:
  clusters:
  - name: test
  - name: kafka
  conditions:
  - lastTransitionTime: "2020-11-30T19:03:16Z"
    lastUpdateTime: "2020-11-30T19:03:16Z"
    status: "True"
    type: Propagation
  observedGeneration: 1
```

<br>

```sh

$ kubectl -n prueba-fed get deployment,pod --context=test
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/echo   1/1     1            1           64s
NAME                        READY   STATUS    RESTARTS   AGE
pod/echo-5b749dd4df-59z94   1/1     Running   0          64s


$ kubectl -n prueba-fed get deployment,pod --context=kafka
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/echo   1/1     1            1           68s
NAME                        READY   STATUS    RESTARTS   AGE
pod/echo-5b749dd4df-sx5vn   1/1     Running   0          69s
```

<br>

De la misma forma desplegaremos el recurso Service del archivo `service.yaml`.

```sh
$ kubefedctl federate -f services.yaml |kubectl -n prueba-fed apply -f -
```

Revisamos que se haya aplicado correctamente en ambos cluster.

```sh
$ kubectl -n prueba-fed get federatedservice
NAME          AGE
echo-svc-lb   69s
```

<br>

```sh
$ kubectl -n prueba-fed get svc --context=test
NAME          TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                         AGE
echo-svc-lb   LoadBalancer   10.43.63.106   <pending>     8080:30603/TCP,8443:30471/TCP   79s


$ kubectl -n prueba-fed get svc --context=kafka
NAME          TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)                         AGE
echo-svc-lb   LoadBalancer   10.43.33.42   <pending>     8080:30234/TCP,8443:30380/TCP   87s
```

Con esto podemos certificar que lo que se despliegue en un cluster se replicará automáticamente a cualquier cluster federado.

<br>

---

# Prueba2 - Modificación de recursos.

Como mostramos anteriormente al desplegar recursos como `FederatedDeployment` o `FederatedService`, estos logran que se repliquen en los cluster federados que tenemos registrados, a su vez, mantiene un estado deseado de los recursos, esto quiere decir que si realizamos alguna modificación a uno de los recursos Deployment o Service de alguno de los clusters, estos volverán a su estado original.

<br>

## Scale deployment

En este ejemplo realizaremos un scale del deployment `echo` previamente aplicado al namespace prueba-fed

```sh
$ kubectl -n prueba-fed scale deploy echo --replicas=2
deployment.apps/echo scaled

$ kubectl -n prueba-fed get deploy
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
echo   2/2     2            2           37m
```

<br>

El deployment se escala correctamente pero en un par de segundos vuelve a su estado original

```sh
$ kubectl -n prueba-fed get pod
NAME                    READY   STATUS        RESTARTS   AGE
echo-5b749dd4df-59z94   1/1     Running       0          37m
echo-5b749dd4df-bs88b   1/1     Terminating   0          13s

$ kubectl -n prueba-fed get deploy
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
echo   1/1     1            1           38m
```

<br>

## Delete Service

Lo mismo sucede al intentar eliminar un recurso, en este ejemplo eliminaremos el recurso Service `echo-svc-lb`.

```sh
$ kubectl -n prueba-fed get svc
NAME          TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                         AGE
echo-svc-lb   LoadBalancer   10.43.63.106   <pending>     8080:30603/TCP,8443:30471/TCP   36m


$ kubectl -n prueba-fed delete svc echo-svc-lb
service "echo-svc-lb" deleted


$ kubectl -n prueba-fed get svc
No resources found in prueba-fed namespace.


$ kubectl -n prueba-fed get svc
NAME          TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                         AGE
echo-svc-lb   LoadBalancer   10.43.39.156   <pending>     8080:31155/TCP,8443:31091/TCP   4s
```

Inicialmente el recurso Service se pudo eliminar pero pasados unos segundos se creó nuevamente