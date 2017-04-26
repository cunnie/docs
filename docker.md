```
docker ps
docker run hello-world
docker pull docker/whalesay
docker run docker/whalesay cowsay boo
docker pull bosh/bosh-lite:9000.123.0 # there's no "latest" tag, so you need to
  # ...hardcode the tag to 9000.123.0
docker run --privileged=true -it bosh/bosh-lite:9000.123.0 /usr/bin/start-bosh
docker ps
docker exec -it a53a8115a52d bash
docker run hello-world
docker run -it ubuntu bash
docker run docker/whalesay cowsay "I love my dog," no really
  # after creating Dockerfile
docker build -t docker-whale .
docker run docker-whale
docker tag 5b3c4c594114 cunnie/docker-whale:latest
docker images
docker login --username=cunnie
docker push cunnie/docker-whale
docker rmi -f cunnie/docker-whale
docker run cunnie/docker-whale
$ docker ps -a | awk '{print $1}' | xargs docker rm
```

Installation for Fedora 25:
```
sudo dnf install docker
sudo systemctl start docker.service
systemctl enable docker.service
```

To not [require root](https://docs.docker.com/engine/installation/linux/linux-postinstall/#manage-docker-as-a-non-root-user)
```
sudo groupadd --system docker
sudo usermod -aG docker cunnie
sudo usermod -aG docker diarizer
sudo shutdown -r now
```
