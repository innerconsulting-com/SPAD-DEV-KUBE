![Image text](Logo_EMQ.png)
# Construyendo el clúster de EMQ desde cero
![Image text](Logo_Kube.png)

# Tabla de Contenidos
- [Construyendo el clúster de EMQ desde cero](#construyendo-el-clúster-de-emq-desde-cero)
- [Tabla de Contenidos](#tabla-de-contenidos)
- [Información General 📋](#información-general-)
- [Implementación de un nodo del agente EMQX MQTT en Kubernetes 🚀](#implementación-de-un-nodo-del-agente-emqx-mqtt-en-kubernetes-)
	- [deployment.yaml para la implementación del pod de EMQX 🔧](#deploymentyaml-para-la-implementación-del-pod-de-emqx-)
	- [service.yaml para la implementación del pod de EMQX ⚙️](#serviceyaml-para-la-implementación-del-pod-de-emqx-️)
	- [Clúster de persistencia EMQX Broker 🛠️](#clúster-de-persistencia-emqx-broker-️)
	- [configmap.yaml para la implementación de la configuracion de EMQX ⚙️](#configmapyaml-para-la-implementación-de-la-configuracion-de-emqx-️)

# Información General 📋

El equipo de EMQX proporciona un Helm Chart que facilita a los usuarios implementar con un clic el agente EMQX MQTT en el clúster de Kubernetes. Aqui se encontraran los pasos necesarios para comenzar desde cero utilizando el método de archivos YAML de escritura a mano para implementar un agente EMQX MQTT. Facilitará a los desarrolladores un uso flexible durante la implementación real.

# Implementación de un nodo del agente EMQX MQTT en Kubernetes 🚀
## deployment.yaml para la implementación del pod de EMQX 🔧

La implementación proporciona un método de definición declarativa para pod. Los escenarios de aplicación típicos incluyen:

- Definir la implementación para crear Pod.
- Desplázarce por las aplicaciones para actualización y reversión
- Expandir y encoger

Use el archivo deployment.yaml, para implementar un módulo del agente de EMQX:

```ssh
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: emqx-pvc
  labels:
    app: emqx
  namespace: emqx
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: emqx-deployment
  labels:
    app: emqx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: emqx
  template:
    metadata:
      labels:
        app: emqx
    spec:
      containers:
      - name: emqx
        image: emqx/emqx:4.3.7
        ports:
        - name: mqtt
          containerPort: 1883
        - name: mqttssl
          containerPort: 8883
        - name: mgmt
          containerPort: 8081
        - name: ws
          containerPort: 8083
        - name: wss
          containerPort: 8084
        - name: dashboard
          containerPort: 18083
        envFrom:
          - configMapRef:
              name: emqx-config
        volumeMounts:
        - name: emqx-persistent-storage
          mountPath: /mnt/data
      volumes:
      - name: emqx-persistent-storage
        persistentVolumeClaim:
          claimName: emqx-pvc
```
PersistentVolumeClaim (PVC) es el almacenamiento establecido por el administrador y es parte del clúster. Este objeto API incluye los detalles de la implementación del almacenamiento, es decir, NFS, iSCSI o sistemas de almacenamiento específicos de los proveedores de la nube.

```ssh
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: emqx-pvc
  labels:
    app: emqx
  namespace: emqx
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

En la primer parte del manifiesto se encuentra el codigo con el cual se declara el persistent volume claim (PVC) con sus caracteristicas como nombre, ruta de montaje, almacenamiento reservado y tipo de acceso permitido por la aplicación.

```ssh
volumeMounts:
        - name: emqx-persistent-storage
          mountPath: /mnt/data
      volumes:
      - name: emqx-persistent-storage
        persistentVolumeClaim:
          claimName: emqx-pvc
```

Es similar al Pod, el pod consume el recurso del nodo y el PVC consume los recursos fotovoltaicos. El Pod puede solicitar recursos a un nivel específico (CPU y RAM). La declaración ***storage*** puede solicitar el tamaño específico y la declaración ***accessModes*** el modo de acceso.

## service.yaml para la implementación del pod de EMQX ⚙️

Cada pod tiene su dirección IP, pero en implementación puede que los demas pods no se comuniquen con el, debido a que no estan sus puertos expuestos, para solucionar esto, se hace uso de un service de tipo load balancer.

El servicio es un método abstracto para exponer la aplicación que se ejecuta en un conjunto de Pods como servicio de red. En este caso, como se exponen los puertos para los demas pods y para ver el dashboard, se debe implementar un service load balancer el cual consiste en generar un

```ssh
apiVersion: v1
kind: Service
metadata:
  name: emqx-service
spec:
  selector:
    app: emqx
  ports:
    - name: mqtt
      port: 1883
      protocol: TCP
      targetPort: mqtt
    - name: mqttssl
      port: 8883
      protocol: TCP
      targetPort: mqttssl
    - name: mgmt
      port: 8081
      protocol: TCP
      targetPort: mgmt
    - name: ws
      port: 8083
      protocol: TCP
      targetPort: ws
    - name: wss
      port: 8084
      protocol: TCP
      targetPort: wss
    - name: dashboard
      port: 18083
      protocol: TCP
      targetPort: dashboard
  type: LoadBalancer
```
En los proveedores de nube que admiten balanceadores de carga externos, establecer el campo de tipo en LoadBalancer aprovisiona un balanceador de carga para su Servicio. La creación real del balanceador de carga se realiza de forma asincrónica y la información sobre el balanceador aprovisionado se publica en el campo ***.status.loadBalancer*** del servicio. El tráfico del balanceador de cargas externo se dirige a los pods de backend. El proveedor de la nube decide cómo se equilibra la carga.

## Clúster de persistencia EMQX Broker 🛠️

La implementación utilizada anteriormente para administrar Pod, pero la red de Pod se puede cambiar constantemente. Cuando el pod se destruye y se reconstruye, los datos y la configuración almacenados en EMQ X Broker también desaparecen, esto no se puede aceptar durante la producción. A continuación, intente la persistencia del clúster EMQ X Broker.
Incluso si el módulo se destruye y se reconstruye, los datos de EMQ X Broker pueden conservarse.

## configmap.yaml para la implementación de la configuracion de EMQX ⚙️

ConfigMap es un objeto API que se utiliza para almacenar datos no confidenciales en pares llave-valor. Se utilizará como variables de entorno, argumentos de línea de comandos o como archivos de configuración en un volumen.
ConfigMap desacopla la información de configuración de su entorno de la duplicación de contenedores, para que pueda modificar fácilmente la configuración de la aplicación.

```ssh
apiVersion: v1
kind: ConfigMap
metadata:
  name: emqx-config
data:
  EMQX_CLUSTER__K8S__ADDRESS_TYPE: "hostname"
  EMQX_CLUSTER__K8S__APISERVER: "https://kubernetes.default.svc:443"
  EMQX_CLUSTER__K8S__SUFFIX: "svc.cluster.local"
```

ConfigMap no proporciona cifrado. Si los datos que desea almacenar son confidenciales, use un ***secret*** en lugar de un ConfigMap, o use herramientas de terceros para mantener la privacidad de sus datos.