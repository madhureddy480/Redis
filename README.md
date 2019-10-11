Redis Server Cluster {3 server nodes}.


# Redis
##### 1. install docker container on your pc. change preference for better performance.

##### 2. go download https://github.com/viragtripathi/redis-enterprise-docker.git
##### 3. edit create_redis_enterprise_3_node_cluster.sh file to your settings..like below.
```
#!/bin/bash

sudo docker kill re-node1;sudo docker rm re-node1;
sudo docker kill re-node2;sudo docker rm re-node2;
sudo docker kill re-node3;sudo docker rm re-node3;

# Start 3 docker containers. Each container is a node in the same network
echo "Starting Redis Enterprise as Docker containers..."
sudo docker run -d --cap-add sys_resource -h re-node1 --name re-node1 -p 18443:8443 -p 19443:9443 -p 14000-14005:12000-12005 -p 18070:8070 redislabs/redis:latest
       # by Madhu - Server Node 1 is created using a docker image. this Server is available at host18443.
       # by Madhu - what are all these ports :? --see here: https://docs.redislabs.com/latest/rs/administering/designing-production/networking/port-configurations/
sudo docker run -d --cap-add sys_resource -h re-node2 --name re-node2 -p 28443:8443 -p 29443:9443 -p 12010-12015:12000-12005 -p 28070:8070 redislabs/redis:latest
       # by Madhu - Server Node 2 is created using a docker image. this Server is available at host28443.
sudo docker run -d --cap-add sys_resource -h re-node3 --name re-node3 -p 38443:8443 -p 39443:9443 -p 12020-12025:12000-12005 -p 38070:8070 redislabs/redis:latest
      # by Madhu - Server Node 3 is created using a docker image. this Server is available at host38443.
# Create Redis Enterprise cluster
echo "Waiting for the servers to start..."

sleep 60

echo "Creating Redis Enterprise cluster and joining nodes..."
sudo docker exec -it --privileged re-node1 "/opt/redislabs/bin/rladmin" cluster create name cluster1.local username admin@admin.com password admin
sudo docker exec -it --privileged re-node2 "/opt/redislabs/bin/rladmin" cluster join nodes $(sudo docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' re-node1) username admin@admin.com password admin
sudo docker exec -it --privileged re-node3 "/opt/redislabs/bin/rladmin" cluster join nodes $(sudo docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' re-node1) username admin@admin.com password admin
    # see above script -- it is joining all nodes to create a cluster of nodes (a.k.a servers).
# Test the cluster 
sudo docker exec -it re-node1 bash -c "/opt/redislabs/bin/rladmin status"

# Create a demo database
echo "Creating demo-db database..."
rm create_demodb.sh
tee -a create_demodb.sh <<EOF
curl -v -k -u admin@admin.com:admin -X POST https://localhost:9443/v1/bdbs -H Content-type:application/json -d '{ "name": "demo-db", "port": 12000, "memory_size": 100000000, "type" : "redis", "replication": true}'
EOF
sudo docker cp create_demodb.sh re-node1:/opt/create_demodb.sh
    # by Madhu - see above, a db is being created in node 1. 
sudo docker exec --user root -it re-node1 bash -c "chmod 777 /opt/create_demodb.sh"
sudo docker exec --user root -it re-node1 bash -c "/opt/create_demodb.sh"
sudo docker exec -it re-node1 bash -c "redis-cli -h redis-12000.cluster1.local -p 12000 PING"
sudo docker exec -it re-node1 bash -c "rladmin status databases"       
sudo docker port re-node1 | grep 12000

echo "Now open the browser and access Redis Enterprise Admin UI at https://127.0.0.1:18443 with username=admin@admin.com and password=admin."

```
##### run the above script by executing
```
madhusudhans-MacBook-Pro:~ madhusudhankarnati$ ./create_redis_enterprise_3_node_cluster.sh 
```
wait for it download and install cluster in your local..

to access bash of node 1:
```
madhusudhans-MacBook-Pro:~ madhusudhankarnati$ docker exec -it re-node1 bash
```
to access bash of node 2:
```
madhusudhans-MacBook-Pro:~ madhusudhankarnati$ docker exec -it re-node1 bash
```
to access bash of node 3:
```
madhusudhans-MacBook-Pro:~ madhusudhankarnati$ docker exec -it re-node1 bash
```

take a note of ports.. 

 docker container ports are already mapped to host operating system ports.

 by now.. your Redis cluster is already up and running with three nodes and one database. 

 access cluster at port 8443 (with in the docker) and localhost:18443 or :28443 or :38443 at your host O.S. (see the above sh file)

access database at port 12000 (with in the docker) and localhost:14000 at your host O.S (see below 12000/tcp is listening traffic from 0.0.0.0:14000)

```
madhusudhans-MacBook-Pro:~ madhusudhankarnati$ docker port re-node1
8443/tcp -> 0.0.0.0:18443
9443/tcp -> 0.0.0.0:19443
8070/tcp -> 0.0.0.0:18070
12004/tcp -> 0.0.0.0:14004
12001/tcp -> 0.0.0.0:14001
12002/tcp -> 0.0.0.0:14002
12000/tcp -> 0.0.0.0:14000
12005/tcp -> 0.0.0.0:14005
12003/tcp -> 0.0.0.0:14003
madhusudhans-MacBook-Pro:~ madhusudhankarnati$ 
```

##### EVERYTHING IS GOOD SO FAR.

if you want Prometheus monitor cluster stats and health:

1. install Prometheus from their site. 

2. edit Prometheus.yml to add Redis cluster listener job.

3. add 
```
# scrape Redis Enterprise
  - job_name: redis-enterprise
    scrape_interval: 30s
    scrape_timeout: 30s
    metrics_path: /
    scheme: https
    tls_config:
      insecure_skip_verify: true
    static_configs:
      - targets: ['localhost:18070']
 ```
     Cluster nodes are pointed to write stats to internal docker port 8070 and configured to allow requests from host 18070. 

4. Start Prometheus by 
```
madhusudhans-MacBook-Pro:~ madhusudhankarnati$ ./Prometheus
```
Prometheus is up and running on host o.s port 9090

##### EVERYTHING IS GOOD SO FAR.    Redis Cluster :) and Prometheus. :) 

##### Now the Grafana part

1. Download from their site. 

2. Start Grafana server ```./grafana-server```

3. Access server at localhost:3000

4. Add Prometheus datasource running at localhost:9090

5. Create dashboards...


