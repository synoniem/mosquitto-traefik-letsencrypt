# mosquitto-traefik-letsencrypt

An Docker compose script that integrates the [Mosquitto MQTT server](https://mosquitto.org/) with [Traefik The Cloud Native Application Proxy](https://traefik.io/traefik/) generating and maintaining [Letsencrypt](https://letsencrypt.org/) TLS certificates.

# TL;DR

For a quick start clone this repo and copy example.env to .env. Change the environment variables in .env to your needs. Make sure that ports 443, 8883 and 8083 are accessible from the outside. Supposing you use a newer version from docker with compose plugin otherwise use docker-compose. Start the script with:

```
docker compose up -d
```
Add user and password to the configuration:

```
docker exec -it mqtt mosquitto_passwd -b /mosquitto/config/passwd user password
```
And restart Mosquitto:

```
docker restart mqtt
```

After this MQTT should be available on ports 8883 (mqtt with TLS) and 8083 (websocket with TLS).


# Longer version

## Why this Docker compose script?

As the Internet of Things (IoT) world rapidly grows and evolves, developers need a simple and secure way to implement peer-to-peer and peer-to-server (backend) communications.  MQTT is a relatively simple message/queue-based protocol that provides exactly that. 
Unfortunately, there are a ton of Docker images available for brokers, e.g. eclipse-mosquitto; but, nearly all of them leave it up to the user to figure out how to secure the platform.  This results in many developers simply ignoring security altogether for the sake of simplicity.  Even more dangerous, many semi-technical home owners are now dabbling in the home automation space and due to the complexity of securing it, they expose IoT/automation devices on the internet completely unsecured.

This docker compose script attempts to make it easier to implement a secure MQTT broker, either in the cloud or on premise.  The secured broker can be used with home automation platforms like [Home Assistant](https://home-assistant.io/) or simply as a means of enabling secure IoT device communications.


## Setting up a MQTT server

In this case, three ports are exposed. The first two ports are associated with Mosquitto, the third port mapping (443:443) allows LetsEncrypt to verify the supplied domainname.


## Environment Variables

There are four environment variables used.
```
MQTT_DOCKER_TAG=latest
MQTT_LETSENCRYPT_EMAIL=mqtt@example.com
MQTT_VIRTUAL_HOST=www.example.com
TRAEFIK_DOCKER_TAG=v2.8
```


**MQTT_VIRTUAL_HOST** - This should be defined as your fully qualified domain name, i.e. mqtt.myserver.com.  The domain name needs to point to your server as LetsEncrypt will verify such when obtaining certificates.

**MQTT_LETSENCRYPT_EMAIL** - This simply needs to be an email address. It's required by LetsEncrypt to obtain certificates.

**TRAEFIK_DOCKER_TAG** - This is the latest stable version Traefik this script has been tested with. If you like to live on the edge you can change it to latest.


## Volumes (persistence)

The needed files associated with this image assume a standard directory structure for Mosquitto configuration and Traefik.  It is possible to deviate from the below defined standard, but doing so should be left to those more familiar with Docker, Traefik and Mosquitto.

```
/mosquitto/config/
	mosquitto.conf
	passwd
/mosquitto/log/
/mosquitto/data/
/traefik-acme/
/traefik-configs/
```

The docker-compose.yml file maps local (persistent) directories to the relevant container volumes:

**/mosquitto/config/** - this directory is where Mosquitto will look for the mosquitto.conf file.

**/mosquitto/config/mosquitto.conf** - this file is user supplied.  The generated container will look for exactly this file in exactly this directory. The added mosquitto.conf has defined **allow_anonymous false** and **password_file /mosquitto/config/passwd**. Both are optional, but allowing anonymous access to MQTT somewhat defeats the purpose of running this docker compose script.

**/mosquitto/config/passwd** - this file is the standard location for Mosquitto users/passwords. An alternate file/location can be specified in mosquitto.conf, but it must be in a location persisted through docker volume mapping. With supplied mosquitto.conf the password file must be present otherwise the container will crash with error 13. A empty passwd file has been added.

**/mosquitto/log/** - This directory is the location where mosquitto will place log file(s). Like passwd defined above, its use is optional and can be controlled based on the contents of mosquitto.conf.

**/acme/** - This directory is where Traefik will place the retrieved certificate(s) in a acme.json file.


## Traefik/LetsEncrypt integration

At container startup, Traefik will look for a certificate for MQTT_VIRTUAL_HOST to exist in /acme/acme.json. If it doesn't find a certificate, it will attempt to obtain them. Traefik will periodically check to see if the certificate need renewal.

## mosquitto.conf

Documentation for Mosquitto should be consulted for details on how to further configure this file if needed. The for reference added **default_mosquitto.conf** in /mosquitto/config is the default config file supplied by Eclipse and contains a lot of information too.

Logging is enabled and the directory for storing log files is defined as /mosquitto/log. Consult [Mosquitto documentation](https://mosquitto.org/documentation/) for the logging parameters if you want another level of logging turned on.

```
# Config file for mosquitto
#
# See mosquitto.conf(5) for more information.
#

# =================================================================
# General configuration
# =================================================================

# When run as root, drop privileges to this user and its primary 
# group.
# Leave blank to stay as root, but this is not recommended.
# If run as a non-root user, this setting has no effect.
# Note that on Windows this has no effect and so mosquitto should 
# be started by the user you wish it to run as.
user mosquitto

# =================================================================
# Default listener
# =================================================================

listener 1883
protocol mqtt

# =================================================================
# Extra listeners
# =================================================================

listener 8083
protocol websockets 

# =================================================================
# Logging
# =================================================================

log_dest file /mosquitto/log/mosquitto.log
log_type information
connection_messages true
log_timestamp_format %Y-%m-%dT%H:%M:%S
log_timestamp true

# =================================================================
# Data persistance
# =================================================================

persistence true
persistence_location /mosquitto/data/

# =================================================================
# Security
# =================================================================

allow_anonymous false

# -----------------------------------------------------------------
# Default authentication and topic access control
# -----------------------------------------------------------------

# Control access to the broker using a password file. This file can be
# generated using the mosquitto_passwd utility. 
password_file /mosquitto/config/passwd
```

## Generating User ID/Password

Mosquitto provides a utility (mosquitto_passwd) for adding users to a password file with encrypted passwords.  Assuming the passwd file is in the standard location as shown in the mosquitto.conf file above, you can add a user/password combination (e.g. firstuser firstPwd) to the file once the docker container is up and running, using the following command:

```
docker exec -it mqtt mosquitto_passwd -b /mosquitto/config/passwd firstuser firstPwd
```

This command doesn't provide any feedback if successful, but does show errors if there are problems.  You can verify success simply looking in the passwd file.  You should see an entry similar to: firstuser:\$6\$+NKkI0p3oZmSukn9$mOUEEHUizK2zqc8Hk2l0JlHHXTW8GPzSonP9Ujrjhs1tVNQqN3lGCAFcFKnpJefOjUPwjqE5mZqSjBl6BCKnPA==

After adding users you should reload the mosquitto server with the following command:
```
docker restart mqtt
```

## Testing Your Server

To test your server locally (i.e. within the container), you can pop into the container and use mosquitto_pub and mosquitto_sub.  Note that you'll need to do this from two separate terminal sessions so see the effect. If you receive error messages, look in the mosquitto error log (/mosquitto/log) for diagnostic information.  You should also make sure the container came up properly using a command like:

```
docker logs mqtt
```

For the MQTT subscriber:

```
docker exec -it mqtt /bin/sh
mosquitto_sub -h <yourserveraddr> -u "firstuser" -P "firstPwd" -t "testQueue"
```

The mosquitto_sub command will block waiting for messages from testQueue.

To publish a message to testQueue, open another terminal and use the following:

```
docker exec -it mqtt /bin/sh
mosquitto_pub -h <yourserveraddr> -u "firstuser" -P "firstPwd" -t "testQueue" -m "Hello subscribers to testQueue!"
```

In the first (subscriber) terminal window, you should immediately see the message "Hello subscribers to testQueue!"

To test remotely, [mqtt-explorer](http://mqtt-explorer.com/) by Thomas Nordquist is an excellent resource.  Note that you need to make sure any filewalls/routers between your container and the internet are properly configured to route requests on the ports specified before attempting to use mqtt-explorer.





