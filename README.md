# Master Datalab IoT - Detección de anomalías en tiempo real

Juan José Cacho
Universidad de Cantabria (Mayo de 2018)


## Contenido
Creación de un flujo de datos en tiempo real para detección de anomalías.


La plataforma contiene los siguientes componentes principales:

- **[Fiware/Orion](https://fiware-orion.readthedocs.io/en/master/)** Context Broker
- **[Fiware/Perseo](http://fiware-iot-stack.readthedocs.io/en/latest/cep/)** Perseo-fe como Context Event Processing y Perseo-Core con el servidor de procesos ETL
- **[NodeRed](https://nodered.org/)** Desarrollo rápido de agentes para tareas de ETL (Extract Transform Load)
- **[NodeRed Dashboard](https://github.com/node-red/node-red-dashboard)** Dashboard de viasualización
- **[InfluxDb](https://www.influxdata.com/)** Base de datos especializada en series de tiempo
- **[Grafana](https://grafana.com/)** Análitica avanzada de datos históricos

## Requisitos

Se requiere tener instalado [Vagrant](https://www.vagrantup.com/downloads.html), VirtualBox y un cliente git.

## Despliegue

```shell
git clone https://github.com/cachuco/datalabiot_anomalias
cd datalabiot
vagrant up
```

Una vez tengamos la VM arrancada, accedemos a la shell:

```
vagrant ssh
```

y arrancamos los servicios:

```
cd /docker
sudo docker-compose up
```

Para eliminar la máquina virtual

```
vagrant destroy -f
```

### Crear suscripción ORION (CB) -> PERSEO (CEP)
```
(curl http://localhost:1026/v1/subscribeContext -s -S --header 'Content-Type: application/json' --header 'Accept: application/json' --header 'Fiware-Service: service' –header 'Fiware-ServicePath: subservice' -d @- | python -mjson.tool) <<EOF
{
    "entities": [
        {
            "type": "sensehat.simu",
            "isPattern": "true",
            "id": ".*"
        }
    ],
    "attributes": [
    ],
    "reference": "http://perseo:9090/notices",
    "duration": "P1Y",
    "notifyConditions": [
        {
            "type": "ONCHANGE",
            "condValues": [
                "temperature"
            ]
        }
    ],
    "throttling": "PT1S"
}
EOF
```

### Crear las reglas para PERSEO (CEP)
```
(curl http://localhost:9090/rules -s -S --header 'Content-Type: application/json' --header 'Accept: application/json' --header 'Fiware-Service: service' –header 'Fiware-ServicePath: subservice' -d @- | python -mjson.tool) <<EOF
{
    "name":"temprule",
    "text":"select *, \"temprule\" as ruleName from pattern [every ev=iotEvent(cast(cast(temperature?,String),float)>35 and type=\"sensehat.simu\")]",
    "action":{
        "type":"post",
        "parameters":{
            "url": "http://nodered:1880/api/alarm",
            "method": "POST",
            "headers": {
                "Content-type": "application/json"
            },
            "qs": {
                "${id}": "${id}",
                "${temperature}": "${temperature}"
            },
            "json": {
                "temperature": "${temperature}",
                "humidity": "${humidity}",
                "pressure": "${pressure}",
                "id": "${id}"
            }
        }
    }
}
EOF
```

```
(curl http://localhost:9090/rules -s -S --header 'Content-Type: application/json' --header 'Accept: application/json' --header 'Fiware-Service: service' –header 'Fiware-ServicePath: subservice' -d @- | python -mjson.tool) <<EOF
{
    "name":"alarm",
    "text":"select *,\"alarm\" as ruleName from pattern [every ev=iotEvent(cast(cast(temperature?,String),float)>35 and type=\"sensehat.simu\")]",
    "action":{
        "type":"update",
        "parameters":{
            "id":"${id}_mirror",
            "name":"alarm",
            "attrType":"boolean",
            "value":"true"
        }
    }
}
EOF
```

```
(curl http://localhost:9090/rules -s -S --header 'Content-Type: application/json' --header 'Accept: application/json' --header 'Fiware-Service: service' –header 'Fiware-ServicePath: subservice' -d @- | python -mjson.tool) <<EOF
{
    "name":"noalarm",
    "text":"select *,\"noalarm\" as ruleName from pattern [every ev=iotEvent(cast(cast(temperature?,String),float)<35 and type=\"sensehat.simu\")]",
    "action":{
        "type":"update",
        "parameters":{
            "id":"${id}_mirror",
            "name":"alarm",
            "attrType":"boolean",
            "value":"false"
        }
    }
}
EOF

```


### Mapa de puertos y servicios de la máquina virtual generada

- **NodeRed**  1880
- **Orion** 1026
- **Perseo** 9090
- **Perseo core** 8080
- **InfluxDb** 8086
- **Grafana** 3000

### Las URL's públicas de acceso son:

- **Nodered** http://localhost:1880
- **Nodered dashboard** http://localhost:1880/ui/
- **Orion** http://localhost:1026/
- **Métricas CEP** http://localhost:9090/admin/metrics


## Documentación

- [Documentación Fiware CEP API](http://fiware-iot-stack.readthedocs.io/en/latest/cep_api/index.html)
- [Creación de reglas EPL para Perseo](https://github.com/telefonicaid/perseo-fe/blob/master/documentation/plain_rules.md)
- [Ejemplos Perseo](https://github.com/telefonicaid/perseo-fe/tree/master/examples)

