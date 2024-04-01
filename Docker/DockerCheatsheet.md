# Docker Cheatsheet

## View stopped containers
- All containers:
```bash
docker ps -a
```
- Last stopped container
```bash
docker ps -l
```

## Create an image from a container
```bash
docker ps -l 
docker commit <ID from previous output>
docker tag <ID from previous output> <new image name>
```

## Run container without keeping container
```bash
docker run --rm -ti ubuntu sleep 5
```

## Run detached (i.e. background) container
```bash
docker run -d -ti ubuntu bash
```
Where `-d` stands for *detached*

## Reattached a detached container
```bash
docker exec -ti <container name> bash
```

## Give name to a container
```bash
docker run --name
```

## Get log for a containter
```bash
docker logs <container name>
```

## Port publishing
```bash
docker run -rm -ti -p <container port>:<host port> --name <container name> ubuntu bash
```

## Networking

### View a Network
```bash
docker network ls
```

### Create a network
```bash
docker network create <network name>
```

### Add container to a network
```bash
docker run -ti --rm --net <network name> --name <container name> ubuntu bash
```

