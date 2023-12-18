# Implementación de un Service Mesh con k8s y Linkerd

## Paso 1: Instalar la CLI

Si es la primera vez en la que ejecuta Linkerd, será necesario que descargue la interfaz de línea de comandos de Linkerd (linkerdCLI) en su máquina local. Esta herramienta le facilitará la interacción con la implementación de Linkerd.

Para realizar la instalación de la interfaz de línea de comandos de manera manual, proceda ejecutando:

    curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh

Asegúrese de seguir detenidamente las instrucciones proporcionadas para agregarlo a la variable de entorno de su ruta:

    export PATH=$HOME/.linkerd2/bin:$PATH

Una vez completada la instalación, asegúrese de verificar que la interfaz de línea de comandos (CLI) se esté ejecutando correctamente mediante el siguiente comando:

    linkerd version

Al ejecutar el comando, debería observar la versión de la interfaz de línea de comandos (CLI), junto con la información del servidor: "Server version: unavailable". Este resultado se debe a que el plano de control aún no ha sido instalado en su clúster. No se preocupe, abordaremos esta cuestión prontamente.

## Paso 2: Valide su clúster de Kubernetes

Los clústeres de Kubernetes pueden ser configurados de diversas formas. Antes de proceder con la instalación del plano de control de Linkerd, es crucial verificar y validar que la configuración esté correctamente establecida. Para asegurarse de que su clúster esté preparado para la instalación de Linkerd, ejecute el siguiente comando:

    linkerd check --pre

## Paso 3: Instalar Linkerd en su clúster

Una vez que la interfaz de línea de comandos (CLI) se está ejecutando localmente y el clúster está listo para operar, es el momento propicio para instalar Linkerd en su clúster de Kubernetes. Para llevar a cabo esta instalación, por favor, ejecute el siguiente comando:

    linkerd install --crds | kubectl apply -f -

seguido por:

    linkerd install --set proxyInit.runAsRoot=true | kubectl apply -f -

Estos comandos generan manifiestos de Kubernetes con todos los recursos principales necesarios para Linkerd (si tiene curiosidad, no dude en inspeccionar este resultado). Al canalizar estos manifiestos, el comando `kubectl apply` le indica a Kubernetes que agregue esos recursos a su clúster. El comando `install --crds` instala las definiciones de recursos personalizados (CRD) de Linkerd, las cuales deben instalarse en primer lugar, mientras que el comando `install` instala el plano de control de Linkerd.

La duración de la instalación del plano de control puede variar según la velocidad de la conexión a Internet de su clúster. Le recomendamos esperar uno o dos minutos hasta que el plano de control haya concluido su instalación. Puede verificar la instalación ejecutando:

    linkerd check

## Paso 4: Explora Linkerd

Vamos a examinar más de cerca lo que realmente realiza Linkerd. Para llevar a cabo esta tarea, necesitaremos instalar una extensión. Dado que el plano de control central de Linkerd es sumamente mínimo, Linkerd se suministra con extensiones que añaden funcionalidades no críticas pero a menudo útiles, incluyendo una variedad de paneles.

Instalemos la extensión de visualización, la cual implementará una pila de métricas y un panel de control en el clúster.

Para instalar la extensión de visualización, ejecute el siguiente comando:

    linkerd viz install | kubectl apply -f -

Una vez que haya completado la instalación de la extensión, procedamos a validar todos los elementos por última vez:

    linkerd check

Con el plano de control y las extensiones instaladas y en funcionamiento, estamos ahora preparados para explorar Linkerd. Acceda al panel mediante:

    linkerd viz dashboard &

Deberías ver una pantalla como esta:

![image](https://github.com/oscarlucas22/Demo-PI/blob/main/imagenes/1.png)

## Paso 5: Instalar la aplicación de demostración

Vamos a instalar una aplicación de demostración llamada Emojivoto. Emojivoto es una aplicación independiente de Kubernetes que utiliza una combinación de llamadas gRPC y HTTP para permitir a los usuarios votar por sus emojis favoritos.

Para instalar Emojivoto en el espacio de nombres (namespace) "emojivoto", ejecute el siguiente comando:

    curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/emojivoto.yml \ | kubectl apply -f -

Este comando instala Emojivoto en su clúster; sin embargo, Linkerd aún no está activo. Necesitaremos "inyectar" la aplicación antes de que Linkerd pueda desplegar su funcionalidad.

Antes de explorar esta integración, examinemos el estado natural de Emojivoto. Realizaremos esto redirigiendo el tráfico hacia su servicio "web-svc", lo que nos permitirá acceder a él a través de nuestro navegador. Redireccione "web-svc" localmente al puerto 8088 ejecutando:

    kubectl -n emojivoto port-forward svc/web-svc 8088:80

Ahora, por favor, visite http://localhost:8088. ¡Voilà! Debería visualizar Emojivoto en todo su esplendor.

Si explora Emojivoto y hace clic en diversas opciones, notará que presenta algunos problemas. Por ejemplo, al intentar votar por el emoji de dona, obtendrá una página 404. No se preocupe, estos errores son intencionales. (En una guía posterior, le mostraremos cómo utilizar Linkerd para identificar y abordar estos problemas).

Con Emojivoto instalado y en ejecución, estamos listos para "inyectarlo", es decir, agregarle los servidores proxy del plano de datos de Linkerd. Podemos realizar esta acción en una aplicación en vivo sin tiempo de inactividad, gracias a las implementaciones continuas de Kubernetes. Inyecte Linkerd en su aplicación Emojivoto ejecutando:

    kubectl get -n emojivoto deploy -o yaml \ 
      | linkerd inject - \ 
      | kubectl apply -f -

Este comando recupera todas las implementaciones que se ejecutan en el espacio de nombres "emojivoto", ejecuta sus manifiestos a través de "linkerd inject", y luego los vuelve a aplicar al clúster. (El comando "linkerd inject" simplemente añade anotaciones a la especificación del pod, indicando a Linkerd que inyecte el proxy en los pods cuando se crean).

Al igual que con "install", "inject" es una operación de texto puro, lo que significa que puede inspeccionar la entrada y la salida antes de utilizarla. Una vez que se ha aplicado mediante "kubectl apply", Kubernetes realizará una implementación continua y actualizará cada módulo con los servidores proxy del plano de datos.

¡Felicidades! Ha añadido Linkerd a la aplicación. Al igual que con el plano de control, puede verificar que todo funcione correctamente en el lado del plano de datos. Consulte su plano de datos mediante:

    linkerd -n emojivoto check --proxy

Y, por supuesto, puede visitar http://localhost:8088 para ver nuevamente Emojivoto en todo su esplendor.

## Instalación de Dashboards de Prometheus y Grafana (Opcional)

Si la instalación de Prometheus y Grafana no la incluye nuestra versión de linkerd podemos instalarlos con Helm

### Instalación de Prometheus con Helm (en el namespace de linkerd)

Añadimos la repo de prometheus a Helm

        helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

### Paso 1: Instalamos prometheus con helm

        helm install prometheus prometheus-community/prometheus --namespace linkerd

### Paso 2: Exponemos el puerto de prometheus  

        kubectl expose service prometheus-server --type=NodePort --target-port=9090 --name=prometheus-server-np --namespace linkerd

### Paso 3: Arrancamos el servicio de prometheus y lanzamos el dashboard de prometheus

        minikube service prometheus-server-np -n linkerd &

Prometheus dashboard

![image](https://github.com/oscarlucas22/Demo-PI/blob/main/imagenes/2.png)

### Instalación de Grafana con Helm (en el namespace de linkerd)

### Paso 1: Agregamos la repo de grafana

        helm repo add grafana https://grafana.github.io/helm-charts

### Paso 2: Instalamos grafana con Helm

        helm install grafana grafana/grafana --namespace linkerd

### Paso 3: Exponemos el puerto de grafana

        kubectl expose service grafana --type=NodePort --target-port=3000 --name=grafana-np --namespace linkerd

### Paso 4: Obtenemos la clave de admin para grafana 

        kubectl get secret --namespace linkerd grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

### Paso 5: Iniciamos el servicio de grafana y lanzamos el dashboard

        minikube service grafana-np -n linkerd &

Grafana dashboard

Para utilizar Grafana con Prometheus, es necesario configurar un origen de datos de tipo Prometheus:

Para ello, accedemos al menú Configuration > Plugin y agregamos una nueva instancia de Prometheus.
En la sección HTTP URL, proporcionamos la dirección y el puerto donde está ubicado nuestro servidor Prometheus.

Posteriormente, para visualizar las métricas, es necesario importar un panel. Para hacerlo, seguimos estos pasos:

1. Navegamos a Dashboards.
2. Seleccionamos Import.
3. Optamos por Import via grafana.com y copiamos el código del panel específico que deseamos utilizar con Kubernetes que es `6417`.

Pulsamos en load ,luego importamos guardamos la configuracion y ya lo tendremos.

![image](https://github.com/oscarlucas22/Demo-PI/blob/main/imagenes/3.png)

El usuario de grafana inicial es: `admin`

Y la clave la obtenemos con: `kubectl get secret --namespace linkerd grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo`
