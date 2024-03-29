# Description
The JTAN sensor uses the [Maltrail](https://github.com/stamparm/maltrail) malicious traffic detection to generate security events and send them to a central platformc for analysis and statistics.

For a fast and easy deployment the sensor is running in Docker containers orchestrated with docker compose and configured using environment variables from an env file. 

The docker compose stack can be run from low power systems like the Raspberry Pi.

The main services available are:
- maltrail
- filebeat
- redis
- exporter
- watchtower

# Component details
## Maltrail
The maltrail service uses a custom docker image that includes the Maltrail application configured to listen for the traffic from the network interface specified with the variable ```NETWORK_INTERFACE```. The variable ```SENSOR_LOCATION``` is used to set the location of the sensor in the field ```sensor``` fo the generated events.

The generated events are sent via udp to the port 5000.

## Filebeat
This service is used to receive events from the Maltrail service and change the format of the timestamp and update the structure of the events.

After this processing, the events are sent to redis service in the ```sensor``` list on ```localhost:6379```

## Redis
Used to receive the events from filebeat and store them for a short time until they are sent to the central server. The events are stored in a list data structure called ```sensor```.

## Exporter
This is a custom service created with Python that periodically reads from the redis list ```sensor``` (based on the value set in the variable ```EXPORT_INTERVAL_IN_MINUTES```).

The events are sent in batches as a json payload (of maximum 100 events) to a https endpoint specified in ```CENTRAL_SERVER_HOST``` which must also contain the protocol, the hostname, the port and the path (example: https://events.com:4443/api/ingest)

The request are authorized using a bearer token which is placed in the headers section of each request. This token is specified in the variable ```CENTRAL_SERVER_API_KEY```.

Also, in this service, the serial number of the machine is added to each event. The serial number must be specified in the variable ```SERIAL_NUMBER```.

Because the connection with the central api server is done using https we must also specify the certificate in pem format so that the requests are verified.

## Watchtower
A utility Docker image used to watch for any updates of the Docker images in the stack. This check is done every hour and if changes are detected, the new image is downloaded and the service is restarted.

Since some of the used images are stored in a private repository, Watchtower must authenticate to Dockerhub.


# Installation
Log into the raspberry pi using ssh and install docker
```
source <(curl -s https://get.docker.com/)

sudo usermod -aG docker $USER
```

Download the repository locally
```
wget https://github.com/dnscro/jtan-sensor/archive/refs/heads/main.zip

sudo apt update

sudo apt install docker-compose

sudo apt install unzip

unzip main.zip

cd jtan-sensor-main
```

Create a copy of the .env.example file
```
cp .env.example .env
```

Before starting the docker-compose stack we need to login the system to dockerhub. For this, a read-only token must be generated in the Dockerhub admin pan el. Then run the command:
```
docker login -u dnsc
```
After this, the token will be required.


Fill the variables in the .env file based on the details in the **Component details** chapter.

Make sure the timezone is set to UTC and the datetime is set.

Run the docker-compose stack:
```
docker-compose up -d
```
