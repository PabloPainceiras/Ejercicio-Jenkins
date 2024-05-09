# PPS ZAP JENKINS

## Introducción

En esta guía montaremos un entorno automatizado de testeo mediante Jenkins. Escanearemos la web “juiceshop” a través de owasp-zap. Para ello crearemos un pipeline en docker, el cual levantara un contenedor de owasp y realizara el escaneo seleccionado.

**[Pipeline](https://github.com/IES-Rafael-Alberti/Ejercicio-Jenkins/blob/main/pipelinezap) que vamos a usar explicado en stages**

```jsx
pipeline {
    agent any
    stages {
        // Esta etapa verifica si el contenedor 'owasp' está en ejecución y lo detiene y elimina si es así.
        stage('Verificar y limpiar contenedor') {
            steps {
                script {
                    sh """
                        if [ \$(docker ps -q -f name=^owasp\$) ]; then
                            echo "El contenedor 'owasp' está corriendo. Deteniéndolo y eliminándolo..."
                            docker stop owasp
                            docker rm owasp
                        elif [ \$(docker ps -aq -f name=^owasp\$) ]; then
                            echo "El contenedor 'owasp' existe pero no está corriendo. Eliminándolo..."
                            docker rm owasp
                        else
                            echo "El contenedor 'owasp' no existe."
                        fi
                    """
                }
            }
        }
        // Esta etapa configura los parámetros para el escaneo, como el tipo de escaneo, el objetivo, la generación de informes, etc.
        stage('Configuro parámetros') {
            steps {
                script { 
                    properties([
                        parameters([
                            choice (
                                choices: ["Baselines", "APIS", "Full"],
                                description: "Tipo de escaneo",
                                name: "SCAN_TYPE"
                            ),
                            string (
                                defaultValue: "http://172.17.0.2:3000/",
                                description: "Objetivo del escaneo",
                                name: "TARGET"
                            ),
                            
                            booleanParam(
                                defaultValue: true, 
                                description: 'Genera reporte', 
                                name: 'GENERATE_REPORT'
                            ),
                            string (
                                defaultValue: "/tmp/reports/reportZAP",
                                description: "Ruta para guardar el informe",
                                name: "REPORT_PATH"
                            )
                            
                        ])
                    ])
                }
            }
        }
        // Esta etapa imprime la información de los parámetros utilizados en la ejecución del pipeline.
        stage("Pipeline Info") {
            steps {
                script {
                    echo"""
                    Parametros usados:
                        Tipo: $params.SCAN_TYPE
                        Objetivo: $params.TARGET
                        Informe: $params.GENERATE_REPORTS
                    """
                }
            }
        }
        // Esta etapa descarga y ejecuta el contenedor de ZAP necesario para realizar el escaneo.
        stage("Configuración del contenedor de ZAP") {
            steps {
                script {
                    echo "Pulling de la imagen de ZAP"
                    sh """
                        docker pull pixylweb/zap2docker-stable
                    """
                    echo "Arrancando el contenedor --> Start"
                    sh """
                        docker run -dt --name=owasp pixylweb/zap2docker-stable /bin/bash
                    """
                }
            }
        }
        // Esta etapa prepara el directorio de trabajo dentro del contenedor de ZAP.
        stage("Prepara el directorio de trabajo") {
            steps {
                script {
                    sh """
                        docker exec owasp mkdir /zap/wrk
                    """
                }
            }
        }
        // Esta etapa realiza el escaneo del objetivo utilizando el contenedor de ZAP según el tipo de escaneo especificado.
        stage("Escanenando el objetivo usando el contenedor") {
            steps {
                script {
                    scan_type="${params.SCAN_TYPE}"
                    echo "---> scan_type_ $scan_type"
                    target = "${params.TARGET}"
                    if(scan_type =="Baselines"){
                        sh """
                            docker exec owasp zap-baseline.py -t $target -x report.xml -i
                        """
                    }
                    else {
                        if(scan_type =="Full"){
                          sh """
                              docker exec owasp zap-full-scan.py -t $target -x report.xml -i
                          """
                         }
                         else {
                        echo "Se ha especificado un tipo de escaneo invalido."
                    }
                    }
                }
            }
        }
        // Esta etapa copia el informe generado dentro del contenedor de ZAP al directorio de trabajo del Jenkins.
        stage("Copiando el informe al directorio de trabajo") {
            steps {
                script{
                    sh """
                        docker cp owasp:/zap/wrk/report.xml ${WORKSPACE}/report.xml
                    """
                }
            }
        }
        // Esta etapa verifica si el contenedor 'owasp' está en ejecución y lo detiene y elimina si es así.
        stage('Eliminar Owasp') {
            steps {
                script {
                    sh """
                        if [ \$(docker ps -q -f name=^owasp\$) ]; then
                            echo "El contenedor 'owasp' está corriendo. Deteniéndolo y eliminándolo..."
                            docker stop owasp
                            docker rm owasp
                        elif [ \$(docker ps -aq -f name=^owasp\$) ]; then
                            echo "El contenedor 'owasp' existe pero no está corriendo. Eliminándolo..."
                            docker rm owasp
                        else
                            echo "El contenedor 'owasp' no existe."
                        fi
                    """
                }
            }
        }
    }  
}
```

## Preparación del entorno

Primero levantaremos la web juice-shop

Usaremos el comando `docker run -d -it --name=juiceshop --rm -p 0.0.0.0:5000:3000 bkimminich/juice-shop` 

NOTA: En caso de no tener una imagen del mismo se nos descargara.

![Untitled](img/Untitled%200.png)

A continuación, con el comando `docker-compose -f rutaendondeseencuentra/jenkins.yml up -d` levantaremos el contenedor jenkins

Para poder hacer ejecutar el comando tendremos que descargar el [jenkins.yml](https://github.com/IES-Rafael-Alberti/Ejercicio-Jenkins/blob/main/jenkins.yml) y un [dockerfile](https://github.com/IES-Rafael-Alberti/Ejercicio-Jenkins/blob/main/Dockerfile), el cual esta enlazado al yml para poder ejecutar docker dentro de jenkins. Importante que ambos ficheros esten en la misma ruta

NOTA: En caso de no tener una imagen del jenkins se nos descargara.

![Untitled](img/Untitled%201.png)

Contenedores creados en docker desktop

![Untitled](img/Untitled%202.png)

Nos conectaremos a la URL: http://localhost:8080 para acceder a jenkins

NOTA: Si es la primera vez que lo inicias tendrás que hacer la instalación y creación de usuario.

## Creación del trabajo

Iniciaremos una nueva tarea

![Untitled](img/Untitled%203.png)

Asignaremos el nombre que queramos y seleccionaremos la opción “Pipeline”

![Untitled](img/Untitled%204.png)

Pasaremos a la configuración del pipeline.

Clicaremos en la sección “Pipeline.

Seleccionamos Pipeline script e introduciremos el código del pipeline anteriormente proporcionado

![Untitled](img/Untitled%205.png)

Guardamos y arrancaremos el pipeline

![Untitled](img/Untitled%206.png)

Una vez arrancado nos aparecerá la siguiente, el cual nos hace una línea de las etapas

![Untitled](img/Untitled%207.png)

Esto nos creara un reporte el cual podemos encontrar en la ruta

`/var/jenkins_home/workspace/”nombredelpipeline”` dentro de la propia linea de comandos de docker.

o

dentro del trabajo creado, nos iremos a “Workspaces y lo encontraremos clicando en la ruta .

![Untitled](img/Untitled%208.png)

![Untitled](img/Untitled%209.png)

Report.xml

![Untitled](img/Untitled%2010.png)

## Futuros escaneos

Para futuros trabajos, la opción “Contruir Ahora” nos aparecerá de la siguiente manera.

![Untitled](img/Untitled%2011.png)

Esta nos permitirá seleccionar el tipo de escaneo que queramos; baselines, APIS y Full, que son los que contiene el owasp zap.

![Untitled](img/Untitled%2012.png)

Tambien nos permitira cambiar la ip del target, indicar si queremos que nos cree el reporte y la ruta en donde queremos guardarlo.

![Untitled](img/Untitled%2013.png)
