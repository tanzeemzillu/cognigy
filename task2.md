# docker swarm (file reference 3) compatible docker-compose file

In this document the docker-compose.yml is describing the deployment template for the microservices and the mongodb and nginx which are also containerized in this case. The below services will deploy via this docker-compose file 

1. api
2. user-management
3. web-frontend
4. mongo
5. nginx

The docker-compose file looks like below

```
version: '3.7'

services:
  api:
    image: cognigy/api:${TAG_API}
    environment:
      MONGO_ROOT_PASSWORD: run/secrets/mongo-pass
      MONGO_USER: ${DB_USER}
      MONGO_ENDPOINT: mongo:27017
    links:
      - user-management
    depends_on:
      - user-management
      - mongo
    deploy:
      mode: replicated
      replicas: 2
      update_config:
        delay: 20s
        failure_action: rollback
        order: start-first
      rollback_config:
        parallelism: 1
        failure_action: continue
        order: start-first
      restart_policy:
        condition: on-failure
        delay: 20s
        max_attempts: 3
        window: 240s
      resources:
        limits:
          memory: 1G
          cpus: '2'
        reservations:
          memory: 512M
          cpus: '1'
      placement:
        preferences:
          - spread: node.labels.zone
        constraints:
          - node.labels.purpose == app
    ports:
      - "5000:5000"
    secrets:
      - mongo-pass

  user-management:
    image: cognigy/user-management:${TAG_UM}
    environment:
      MONGO_ROOT_PASSWORD: run/secrets/mongo-pass
      MONGO_USER: ${DB_USER}
      MONGO_ENDPOINT: mongo:27017
    depends_on:
      - mongo
    deploy:
      mode: replicated
      replicas: 2
      update_config:
        delay: 20s
        failure_action: rollback
        order: start-first
      rollback_config:
        parallelism: 1
        failure_action: continue
        order: start-first
      restart_policy:
        condition: on-failure
        delay: 20s
        max_attempts: 3
        window: 240s
      resources:
        limits:
          memory: 1G
          cpus: '2'
        reservations:
          memory: 1G
          cpus: '1'
      placement:
        preferences:
          - spread: node.labels.zone
        constraints:
          - node.labels.purpose == app
    secrets:
      - mongo-pass

  web-frontend:
    image: cognigy/web-frontend:${TAG_WEB}
    deploy:
      mode: replicated
      replicas: 2
      update_config:
        delay: 20s
        failure_action: rollback
        order: start-first
      rollback_config:
        parallelism: 1
        failure_action: continue
        order: start-first
      restart_policy:
        condition: on-failure
        delay: 20s
        max_attempts: 3
        window: 240s
      resources:
        limits:
          memory: 1G
          cpus: '1'
        reservations:
          memory: 512M
          cpus: '0.5'
      placement:
        preferences:
          - spread: node.labels.zone
        constraints:
          - node.labels.purpose == app
    ports:
      - "4000:4000"

  mongo:
    image: mongo
    restart: always
    ports:
    - "27017:27017"
    networks:
      - db
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${DB_USER}
      MONGO_INITDB_ROOT_PASSWORD: run/secrets/mongo-pass
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.purpose == db 
    volumes:
      - /data/db:/data
    secrets:
      - mongo-pass

  nginx:
    image: nginx:1.17
    ports: 
      - "80:80"
      - "443:443"
    deploy:
      replicas: 2 
      placement:
          preferences:
            - spread: node.labels.zone
          constraints:
            - node.labels.purpose == lb
      volumes:
        - ~/default.conf:/etc/nginx/conf.d/default.conf
      secrets:
        - source: app.cert
          target: /etc/nginx/certs

secrets: 
  mongo-pass:
    external: true
  app.cert:
    file: /cert/app.cert

networks:
  db:
    external: true
``` 
> please note, here the repo which name started with `cognigy` is considered as artificial public repo.
> All the placeholder's value are defined in .env file. 

The services (api and user-management) which required connection with mongodb, there the password of mongodb user has been passed via `docker-secret` for security. The resources has been restricted according to the given requirements. 

As explained in task1, I want to deploy the `nginx` and `mongo` in the nodes which labled as `lb` and `db` respectively. The rest of the services should deploy on the nodes which are labled as `app`.To lable the node `docker lable` functionality can be used. For proper distribution placement constraint has been used. For example, in this docker-compose file for nginx the following placement has been used 

```
      placement:
          preferences:
            - spread: node.labels.zone
          constraints:
            - node.labels.purpose == lb

```      
So nginx will only deploy on `lb` labled node. This is similar in other services as well. 

To distribute the containers in different zones a `zone` lable has been used, where the value for zone will be `A` for the components of A zone and `B` for the components of B zone.Here `spread` option has been used as preference. This has not been used for mongo service because according to infrastructure drawing db will be deployed on zone `C` only. 

**api** service has a access to user-management service using `link` option and it has been exposed over `5000`port. The HTTP access for this service(which in this case `192.168.0.1:5000` or `192.168.1.1:5000`) will be configured as a backend in nginx so user can access the service via HTTP proxy. This service will come up after `mongo` and `user-management` service as it has dependency on it. 

**user-management** service has not been exposed to the outside and it has access to the `mongo` as well.

**web-frontend** service is exposed over `4000` and it can also accessed via HTTP proxy(Similar to the API service. In this case `192.168.0.1:5000` or `192.168.1.1:5000` will be configured as backed in nginx). This service has no connection with mongodb. 

**mongo** service has a docker volume mount as per requirements.  

**nginx** service also have a volume mount to get the nginx configuration and `docker secret` has been used to transfer the TLS certificate in a secure way. 
