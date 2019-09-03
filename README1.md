
###Tunnig Fluentd

#Planteo
El objetivo del presente documento es poder visualizar desde kibana solo los logs de los proyectos, pods, containers deseados. No usando filtros de kibana, sino filtrar el logging desde fluentd.

#Marco teórico
En el stack de logging de openshift tenemos tres componentes claves.

- Fluentd: Encargado de colectar los logs en los nodos de Openshift y realizar el reenvío de los datos de manera estandarizada a un repositorio central, elasticsearch.
- Elasticsearch: Repositorio de todos los logs que envía fluentd.
- Kibana: dashboard para visualizar los datos del repositorio elasticsearch.

Los logs son colectados por el demonio de docker de manera local en cada nodos. El demonio de docker posee distintos drivers de logging. A partir de la versión 3.9 de Openshift el driver de logging por defecto para docker es de ticket json-log. Este driver es necesario para poder hacer uso de los filtros de fluentd. 

Fluentd puede reenviar logs a distintos endpoints. Los endpoints soportados son elasticsearch y un demonio de fluentd-forwarder. Para el caso de elasticsearch puede ser interno al cluster de openshift o externo fuera del cluster de openshift.

En caso de que sea interna al cluster, un despliegue de stack de logging de openshift permite, si se lo desea, desplegar un elasticsearch para los datos de operación (OPS_HOST) y uno dedicado para aplicaciones (ES_HOST). Por defecto solo se despliega un cluster de elasticsearch (ES_HOST).

En el archivo de configuración de fluentd encontraremos tres secciones source (INPUT), filter (FILTER) y match (OUTPUT). Trabajaremos sobre la sección FILTER. Mas informacion de fluentd (Openshift Fluentd)

# Propuesta

# Filtros Custom 
Todos los archivos de configuración de fluentd (y todo el stack) están almacenados en su respectivo configmaps. Para poder realizar un filtro custom debemos modificar el configmaps con los valores de los filtros deseados y luego recargar la configuración en cada uno de los pods desplegados. Los pasos son los siguientes.

Creamos el directorio de trabajo.
```
$ mkdir -p workdir/(ori,mod) ; cd workdir/ori
```

Colocarse en el proyecto logging o openshift-logging, que son los proyectos donde está instalado el stack de logging. Cargar la variable fluentd_pod, la usaremos para hacer el backup de los archivos de configuración necesario.

```
fluentd_pod=$(oc get pods -o name | grep fluentd | sed 's/pods\///' | head -1)
```

NOTA: En caso de que haya dos despliegues de elasticsearch, uno para operaciones y otro para aplicaciones se deben realizar el backup de los archivos de configuración de fluentd output-applications.conf o output-operations.conf. En caso de que solo haya uno no es necesario este paso.

```
oc exec $fluentd_pod -- \
  cat /etc/fluent/configs.d/openshift/output-applications.conf > \
  output-applications.conf
```

```
oc exec $fluentd_pod -- \
  cat /etc/fluent/configs.d/openshift/output-operations.conf > \
  output-operations.conf
```

Backup de los archivos de configuracion alojados en el configmap de flluentd.

```
oc extract configmap/logging-fluentd --to=.
```

Los archivos de configuracion originales son los siguientes, son utiles para realizar un rollback a la conf original.

```
ls workdir/ori/
fluent.conf  secure-forward.conf  throttle-config.yaml
```

Copiamos el directorio ori para poder hacer uso de las modificaciones.
```
cd ..
cp -pvr ori/ mod/
cd mod
```

#Creamos el archivo de configuracion de filtros filter-post-retag-apps.conf con el siguiente contenido.

La primer parte <filter kubernetes.**> agrega tres campos de nivel superior, estos son necesarios porque los filtros de fluentd no pueden aplicacarse a campos anidados del tipo kubernets.namespace_name (con punto intermedio), debe ser solo uno kubernetes_namespace_name (con guion bajo intermedio). Para esto usamos el @type record_transformer de fluentd. Esto agrega los tres campos  kubernetes_namespace_name, kubernetes_pod_name y kubernetes_container_name pero podría agregarse el que uno quiera construir. 

vi filter-post-retag-apps.conf 
```
# Agregar campos para filtrado
<filter kubernetes.**>
  @type record_transformer
  enable_ruby
  <record>
    kubernetes_namespace_name ${record["kubernetes"] && record["kubernetes"]["namespace_name"] ? record["kubernetes"]["namespace_name"] : "undefined_namespace_name"}
    kubernetes_pod_name ${record["kubernetes"] && record["kubernetes"]["pod_name"] ? record["kubernetes"]["pod_name"] : "undefined_pod_name"}
    kubernetes_container_name ${record["kubernetes"] && record["kubernetes"]["container_name"] ? record["kubernetes"]["container_name"] : "undefined_container_name"}
  </record>
</filter>
# Crear filtros por namespace, pod o container. 
<filter **>
  @type grep
  #Always filter out the restricted namespaces
  regexp1 kubernetes_namespace_name ^(tercero)
  exclude1 kubernetes_namespace_name (null|devnull|logging|default|kube-public|kube-service-catalog|kube-system|logging|management-infra|openshift|openshift-ansible-service-broker|openshift-infra|openshift-metrics|openshift-node|primero|segundo)
  #exclude2 level (info)
</filter>
```

En este caso hacemos uso del filtro @type grep de fluentd para usar expresiones regulares para filtrar campos. regexp1 actúa sobre el campo kubernetes_namespace_name filtrando solo por aquello que comience con tercero (nombre del proyecto en openshift). En la sentencia exclude1 excluimos todos los proyectos sobre los cuales no queremos que sea enviado data a elasticsearch, proyectos de infra y dos agregados primero y segundo. En el último exclude2 ponemos como ejemplo otro campo de nivel superior que nos permite excluir todo lo que sea de tipo info. Mas información sobre filtros de fluentd aca (Fluentd grep)


Editamos el archivo de configuracion de fluentd.conf para agregar el archivo recien creado. Buscamos dentro del archivo estas lineas.

```
…
<label @INGRESS>
## filters
...
  @include configs.d/openshift/filter-viaq-data-model.conf
  @include configs.d/openshift/filter-post-*.conf
…

Y agregamos la linea del medio, debe quedar asi.
...
  @include configs.d/openshift/filter-viaq-data-model.conf
  @include configs.d/user/filter-post-retag-apps.conf
  @include configs.d/openshift/filter-post-*.conf
...
```

Resumiendo hasta el momento nos quedan estos archivos de configuración.

```
# ls 
filter-post-retag-apps.conf  fluent.conf  secure-forward.conf  throttle-config.yaml
```

Una vez creado el archivo nos queda subir la configuración y realizar el restart de los pods de fluentd. Seguimos los siguientes pasos.

Eliminamos el configmap de fluentd.
```
oc delete configmap logging-fluentd
```

En el directorio ./wordir/mod cargamos los archivos de configuración que listamos previamente en el configmap.
```
oc create configmap logging-fluentd --from-file=.
```

Fluentd corre como daemonset, esto quiere decir que los pods de fluentd se van a desplegar en aquellos nodos que estén etiquetamos como logging-infra-fluentd=true. En aquellos nodos que esté corriendo el pod pero el valor del tag sea cambiado a false, el pod de fluentd será eliminado. Por lo tanto para reiniciar su configuración cambiamos el valor de la variable.

Paramos los pods.
```
oc label nodes -l logging-infra-fluentd=true logging-infra-fluentd=false --overwrite
```


Aguardamos que bajen
```
watch -n 2 oc get pods -n logging
```

Una vez bajo los volvemos a subir.
```
oc label nodes -l logging-infra-fluentd=false logging-infra-fluentd=true --overwrite
```

Una vez que estén arriba podemos chequear la configuración entrando a uno de los pods, no es necesario entrar a todos, solo a uno. Chequeamos los dos archivos se han creado.

```
oc rsh $(oc get pods -o name | grep fluentd | sed 's/pods\///' | head -1)
sh-4.2# ls -ltr /etc/fluent/fluent.conf /etc/fluent/configs.d/user/filter-post-retag-apps.conf 
lrwxrwxrwx. 1 root root 38 Apr 25 20:03 /etc/fluent/fluent.conf -> /etc/fluent/configs.d/user/fluent.conf
lrwxrwxrwx. 1 root root 34 Sep  2 21:56 /etc/fluent/configs.d/user/filter-post-retag-apps.conf -> ..data/filter-post-retag-apps.conf
sh-4.2# 
```

En la imagen se puede ver que el filtro está para todos los logs y las ultimas entradas solo corresponden al filtro seleccionado en la prueba. En al recuadro de abajo se pueden ver los tres campos que hemos agregado para realizar el filtro.

