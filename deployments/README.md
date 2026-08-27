# Guía de Despliegue: Aplicación de Votación en AWS EKS (Auto Mode)

Esta guía documenta los pasos necesarios para desplegar la aplicación en un clúster de Amazon Elastic Kubernetes Service (EKS) utilizando **EKS Auto Mode** y exponiendo los servicios a internet mediante **Network Load Balancers (NLB)**.

---

## 1. Configuración Previa en AWS (Consola)

Antes de aplicar cualquier archivo `.yaml`, es obligatorio configurar la infraestructura base en AWS.

### Roles de IAM
Se deben crear dos roles en IAM con las siguientes políticas adjuntas:

**Cluster Role (Ej. `EKSRole`)**
Permite a EKS gestionar el clúster.
* `AmazonEKSBlockStoragePolicyV2`
* `AmazonEKSClusterPolicy`
* `AmazonEKSComputePolicy`
* `AmazonEKSLoadBalancingPolicy`
* `AmazonEKSNetworkingPolicy`

**Node Role (Ej. `EKSNodeRole`)**
Permite a los nodos conectarse al clúster y descargar imágenes.
* `AmazonEC2ContainerRegistryPullOnly`
* `AmazonEKS_CNI_Policy` (Crítica para la red de los pods)
* `AmazonEKSWorkerNodeMinimalPolicy`
* `AmazonEKSWorkerNodePolicy`
* `AmazonElasticContainerRegistryPublicReadOnly`
* `AmazonSSMManagedInstanceCore` (Opcional, para entrar por Session Manager)

### Etiquetas (Tags) en Subredes (VPC)
Para que el AWS Load Balancer Controller sepa dónde ubicar los balanceadores de carga públicos, debes ir a **VPC > Subnets**, seleccionar tus subredes públicas y añadirles estas dos etiquetas:

| Key | Value | Propósito |
| :--- | :--- | :--- |
| `kubernetes.io/role/elb` | `1` | Indica que la subred es pública y apta para Load Balancers. |
| `kubernetes.io/cluster/tu-nombre-de-cluster` | `shared` | Vincula la subred a tu clúster específico. |

---

## 2. Creación del Clúster EKS

El despliegue utiliza la función moderna de EKS para evitar la gestión manual de servidores:

1. Crear el clúster desde la consola de EKS.
2. Seleccionar el **Cluster Role** creado previamente.
3. Asegurarse de tener habilitado **EKS Auto Mode** durante la creación.
4. Asignar el **Node Role** en la sección correspondiente.
5. **IMPORTANTE:** Una vez activo, NO crear Node Groups manuales. El Auto Mode encenderá las instancias EC2 automáticamente cuando se desplieguen los Pods.

---

## 3. Modificaciones en los Archivos YAML

Para que las aplicaciones web (`vote` y `result`) sean accesibles desde internet, sus archivos de servicio (`Service`) deben modificarse. No basta con `NodePort`. 

Se debe cambiar el tipo a `LoadBalancer` y agregar las anotaciones específicas de AWS:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: result
  labels:
    app: result
  annotations:
    # Indica a AWS que use el nuevo NLB externo y lo haga público
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer # Cambiado de NodePort a LoadBalancer
  ports:
  - name: "result-service"
    port: 80 # Puerto público a escuchar
    targetPort: 80
  selector:
    app: result
