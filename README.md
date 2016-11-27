Metronome for Docker
===============
This project provides a tunable Metronome Dockerfile. 

Dependencies:

- Metronome itself.  The SHA used here is 2 commits beyond 0.91 because it fixes a constraint attribute bug 
- Protobuf
  Pin this to 2.61.  Protobuf 3 is out but the proto file used in this project assumes protobuf 2.  Specifically, the proto file is missing 'version' needed to use protobuf 3
- Mesos 1.0.1


Building

```
docker build --build-arg METRONOME_VERSION=4336d5b15cdd19 -f Dockerfile-ubuntu -t f4tq/metronome:0.9.1.4336d5b15cdd19-mesos-1.0.1 .

```

Running

I run docker in a vagrant based ubuntu 16.04 vm.  

- Using the included docker-compose.yml

```
vagrant $  DOCKER_IP=10.0,2.0 docker-compose up
--snip--
vagrant $ docker ps 

4675871e96c1        f4tq/metronome:0.9.1.4336d5b15cdd19-mesos-1.0.1   "/bin/sh -c '$APP_DIR"   7 seconds ago       Up 6 seconds                            mesoscompose_metronome_1
e3a3b250b04c        mesosphere/mesos-master:1.0.1-2.0.93.ubuntu1404   "mesos-master --regis"   7 seconds ago       Up 6 seconds                            mesoscompose_master_1
00821b629413        mesosphere/mesos-slave:1.0.1-2.0.93.ubuntu1404    "mesos-slave"            7 seconds ago       Up 6 seconds                            mesoscompose_slave-one_1
044e9ec1931c        mesosphere/marathon:v1.3.0                        "./bin/start"            7 seconds ago       Up 6 seconds                            mesoscompose_marathon_1
c70020fc755e        bobrik/zookeeper                                  "/run.sh"                7 seconds ago       Up 6 seconds                            mesoscompose_zk_1


```

> Note: The supplied docker-compose file starts 1 mesos master,slave, marathon and metronome.  Also, all containers are running with docker's `--net=host`. mesos-slave needs privilege to launch docker tasks.

- Directly with Docker

```
docker run -i --rm --net host -e JAVA_OPTS="-Dmetronome.mesos.master.url=${DOCKER_IP}:5050 -Dmetronome.zk.url=zk://localhost:2181/metronome -Dplay.server.https.port=9943 -Dplay.server.http.port=9001" f4tq/metronome:0.9.1.4336d5b15cdd19-mesos-1.0.1 
```

> Note: The play server takes a useful '-Duser.dir=/app/conf' which you can combine with docker volumes `-v/myconf_dir:/app/conf` that you can use to control the 100s of options configurable for metronome and the play server


Testing
I use httpie here

- Launching
  
   - Load a docker task into metronome

```

vagrant:metronone-docker $ http POST http://localhost:9000/v1/jobs < ~/metronome-dcos-internal.json
HTTP/1.1 201 Created
Content-Length: 456
Content-Type: application/json
Date: Sun, 27 Nov 2016 16:40:15 GMT

{
    "description": "Locust Test Application", 
    "id": "dcos.locust", 
    "labels": {
        "location": "olympus", 
        "owner": "zeus"
    }, 
    "run": {
        "artifacts": [], 
        "cmd": "/usr/local/bin/dcos-tests --debug --term-wait 20 --http-addr :8095", 
        "cpus": 0.5, 
        "disk": 128, 
        "docker": {
            "image": "f4tq/dcos-tests:v0.31"
        }, 
        "env": {
            "CONNECT": "direct", 
            "MON": "test"
        }, 
        "maxLaunchDelay": 3600, 
        "mem": 32, 
        "placement": {
            "constraints": []
        }, 
        "restart": {
            "activeDeadlineSeconds": 120, 
            "policy": "NEVER"
        }, 
        "user": "root", 
        "volumes": []
    }
}

```

  - Run after loading

```
vagrant:metronone-docker $ http POST http://localhost:9000/v1/jobs/dcos.locust/runs
HTTP/1.1 201 Created
Content-Length: 142
Content-Type: application/json
Date: Sun, 27 Nov 2016 16:43:58 GMT

{
    "completedAt": null, 
    "createdAt": "2016-11-27T16:43:58.408+0000", 
    "id": "20161127164358npeza", 
    "jobId": "dcos.locust", 
    "status": "INITIAL", 
    "tasks": []
}
```
   - Verify it's loaded

```
vagrant $ docker ps
CONTAINER ID        IMAGE                                             COMMAND                  CREATED             STATUS              PORTS               NAMES
573381d33c8b        f4tq/dcos-tests:v0.31                             "/bin/sh -c '/usr/loc"   3 minutes ago       Up 3 minutes                            mesos-990e0126-31eb-41d4-9450-f6ab347d655a-S0.11bf14e8-ad81-4023-bc66-ecb47da63dad

```

  - Test the app itself
```
vagrant $ http GET http://localhost:8095/sleep/1
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 110
Content-Type: text/plain
Date: Sun, 27 Nov 2016 16:51:18 GMT
Keep-Alive: timeout=60
Server: gophr

[2016-11-27 16:51:18.97817714 +0000 UTC] slept for 1 seconds starting @2016-11-27 16:51:17.977932395 +0000 UTC
```

> For more information on see [f4tq/dcos](https://github.com/f4tq/dcos-tests)




> In the above example, I set 9001 to be the metronome port (not the default 9000).  Also, the container it launches sole purpose is to listen for testing container draining on CoreOS/mesos


- Test the job
```
http GET http://localhost:9000/v1/jobs/dcos.locust/status
```

 


