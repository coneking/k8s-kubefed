# Habilitar otros recursos para federar

Al realizar la instalación de kubefed se habilitan algunos recursos k8s para que se puedan federar pero si es necesario federar algo como un recurso Role, Rolebinding o PVC se debe especificar que esté habilitado para su transformación.

<br>

## Ejemplo

Tenemos el siguiente recurso de tipo Role en el archivo `role.yaml`

```sh
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: role-test
rules:
- apiGroups: ["*"]
  resources:
  - pods/log
  - pods/status
  - pods/top
  verbs:
  - get
  - list
  - watch
```

<br>

Si intentamos transformar el archivo y aplicarlo tendremos un el siguiente error.

```sh
kubefedctl federate -f role.yaml  |kubectl -n prueba-fed apply -f -
error: unable to recognize "STDIN": no matches for kind "FederatedRole" in version "types.kubefed.io/v1beta1"
```

<br>

En este caso se ejecuta `kubefedctl enable` se debe especificar el tipo de API del recurso a habilitar y el nombre del contexto del cluster federado que se creó inicialmente (para más información `kubefedctl --help`).

```sh
$ kubefedctl enable roles.rbac.authorization.k8s.io --host-cluster-context=test
customresourcedefinition.apiextensions.k8s.io/federatedroles.types.kubefed.io created
federatedtypeconfig.core.kubefed.io/roles.rbac.authorization.k8s.io created in namespace kube-federation-system
```

<br>

Posterior a esto se podrá crear el recurso federado sin problemas

```sh
$ kubefedctl federate -f role.yaml |kubectl -n prueba-fed apply -f -
federatedrole.types.kubefed.io/role-test created

 
$ kubectl -n prueba-fed get federatedrole
NAME        AGE
role-test   12s


$ kubectl -n prueba-fed get role --context=test
NAME        AGE
role-test   22s

$ kubectl -n prueba-fed get role --context=kafka
NAME        AGE
role-test   30s
```

<br>

## Ejemplo 2 - PVC

Para los recursos de tipo PVC es el mismo caso, se debe habilitar el recuso para posteriormente federarlo.

Contenido del archivo `pvc.yaml`

```sh
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-test
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

<br>

```sh
$ kubefedctl enable persistentvolumeclaim --host-cluster-context=test
customresourcedefinition.apiextensions.k8s.io/federatedpersistentvolumeclaims.types.kubefed.io created
federatedtypeconfig.core.kubefed.io/persistentvolumeclaims created in namespace kube-federation-system


$ kubefedctl federate -f pvc.yaml  |kubectl -n prueba-fed apply -f -
federatedpersistentvolumeclaim.types.kubefed.io/pvc-test created


$ kubectl -n prueba-fed get pvc --context=test
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-test   Bound    pvc-a2c9fe05-6aae-40cf-802f-aa6520f7d98b   1Gi        RWO            fast           95s


$ kubectl -n prueba-fed get pvc --context=kafka
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-test   Bound    pvc-ad4d6ed5-d3c3-4247-b6fb-0646596bcd6f   1Gi        RWO            fast           105s
```

Se debe tener especial cuidado con este recurso puesto que en los cluster federados debe existir el mismo tipo de StorageClass, esto si es que se llega a especificar en el archivo de creación del PVC.


<br>

## Ejemplo 3 - PVC 2

De acuerdo a lo advertido anteriormente respecto de los recursos PVC, pongamos la siguiente situación.
El cluster k8s `test` tiene un StorageClass llamado `fast`.
El cluster k8s `kafka` tiene un StorageClass llamado `fast2`

<br>

Si tenemos el siguiente archivo `new-pvc.yaml`

```sh
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: new-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClass: fast
```

Al transformar este archivo con el binario `kubefedctl` quedaría de la situiente manera

```sh
$ kubefedctl federate -f new-pvc.yaml 
---
apiVersion: types.kubefed.io/v1beta1
kind: FederatedPersistentVolumeClaim
metadata:
  name: new-pvc
spec:
  placement:
    clusterSelector:
      matchLabels: {}
  template:
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClass: fast
```

<br>

Si aplicamos este archivo se podrá crear un PVC de 1Gi en el cluster k8s `test` pero no en el `kafka`, puesto que en este último no existe el `storageClass fast`.
Para solucionar esto tendremos que crear el recurso `FederatedPersistentVolumeClaim` indicándole que en el cluster `kafka` el storageClass se llama `fast2` de la siguiente forma.

```sh
apiVersion: types.kubefed.io/v1beta1
kind: FederatedPersistentVolumeClaim
metadata:
  name: new-pvc
spec:
  placement:
    clusters:
    - name: test
    - name: kafka
  overrides:
  - clusterName: kafka
    clusterOverrides:
    - path: "/spec/storageClassName"
      value: "fast2"
  template:
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClass: fast
```

En este ejemplo se le indicó que sobrescribiera la información `spec.storageClass` del recurso PVC por `fast2` en el cluster k8s kafka usando la opción `overrides`.

<br>

Con esto el recurso PVC se creará en el `cluster k8s test` usando el `storageClass fast` indicado en el archivo y en el `cluster k8s kafka` se creará usando el `storageClass fast2` el cual se sobrescribió.

```sh
$ kubectl -n prueba-fed get pvc --context=test
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
new-pvc      Bound    pvc-8d095fc7-e89a-45c1-a310-a800d0063ed6   1Gi        RWO            fast           17m


$ kubectl -n prueba-fed get pvc --context=kafka
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
new-pvc      Bound    pvc-6eb92e96-cace-42de-ac6d-654547515958   1Gi        RWO            fast2          17m
```

<br>

La opción `overrides` se puede utilizar con cualquier recurso que se quiera modificar.