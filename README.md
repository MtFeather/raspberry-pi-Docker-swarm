# raspberry-pi-Docker-swarm
## Containers
- Arm系統要使用docker必須使用這個核心:
  https://github.com/kennychennetman/docker/blob/master/visualizer-arm.txt

1.	需要一個docker file   
https://docs.docker.com/get-started/part2/#dockerfile
#### vim dockerfile
```vim
# Use an official Python runtime as a parent image
FROM python:2.7-slim

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
ADD . /app

# Install any needed packages specified in requirements.txt
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]
```
2.	建立requirements.txt 和 app.py
#### vim requirements.txt
```vim
Flask
Redis
```
#### vim app.py
```python
from flask import Flask
from redis import Redis, RedisError
import os
import socket

# Connect to Redis
redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)

app = Flask(__name__)

@app.route("/")
def hello():
    try:
        visits = redis.incr("counter")
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"

    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>" \
           "<b>Visits:</b> {visits}"
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```
3.	建置APP
```Bash
docker build -t friendlyhello .
```
4. 執行APP
```Bash
docker run -p 4000:80 friendlyhello
```

5. 上傳image到docker
```Bash
docker push username/repository:tag
```
6. 可以到雲端去抓image來用
```Bash
$ docker run -p 4000:80 username/repository:tag
Unable to find image 'john/get-started:part2' locally
part2: Pulling from john/get-started
10a267c67f42: Already exists
f68a39a6a5e4: Already exists
9beaffc0cf19: Already exists
3c1fe835fb6b: Already exists
4c9f1fa8fcb8: Already exists
ee7d8f576a14: Already exists
fbccdcced46e: Already exists
Digest: sha256:0601c866aab2adcc6498200efd0f754037e909e5fd42069adeff72d1e2439068
Status: Downloaded newer image for john/get-started:part2
 * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
```

## Services
1. create docker-compose.yml file
```yml
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: username/repo:tag
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
      - "80:80"
    networks:
      - webnet
networks:       # 必須要有同樣名稱才能互相連線
  webnet:
```
2. 建立管理者
#### manager 
- 可以建置多個管理者，以防image掛掉，但是不須為奇數管理者，才能去決定
```Bash
$ docker swarm init
```
- 取得要join的碼
```Bash
$ docker swarm join-token worker
```
- 給予app名子
```Bash
$ docker stack deploy -c docker-compose.yml getstartedlab
```
- 查看服務名子
```Bash
$ docker service ls
```
```Bash
$ docker service ps getstartedlab_web
```
```Bash
docker container ls -q
```
- 如果yml檔案有更動要重新執行
```Bash
docker stack deploy -c docker-compose.yml getstartedlab
```
- 停止app
```Bash
docker stack rm getstartedlab
```
- 停止swarm
```Bash
docker swarm leave --force
```
## Swarm
1. 加入node
- 可以先查詢加入代碼
```Bash
$ docker swarm join-token worker
```
- 加入
```Bash
$ docker swarm join --token <token> <myvm ip>:<port>
```
- 當結點遺失並回復後，服務並不會發派，所以要用此指令重新發派服務
```Bash
$ docker service scale getstartedlab_web=5
```

## Stack
### Persist the data 加入新服務
#### Redis:
- 建立新的yml檔
```yml
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: username/repo:tag
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
      - "80:80"
    networks:
      - webnet
  redis:
    image: redis
    ports:
      - "6379:6379"
    volumes:
      - "/home/docker/data:/data"
    deploy:
      placement:
        constraints: [node.role == manager]
    command: redis-server --appendonly yes
    networks:
      - webnet
networks: 
  webnet:
```
- 可以用 `--no-trunc` 查看錯誤狀態
```Bash
$ docker service ps getstartedlab_redis --no-trunc
```
***
#### visualizer: 圖形化系統狀態
- 新增visualizer服務
```yml
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: username/repo:tag
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
      - "80:80"
    networks:
      - webnet
  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]
    networks:
      - webnet
  redis:
    image: redis
    ports:
      - "6379:6379"
    volumes:
      - "/home/docker/data:/data"
    deploy:
      placement:
        constraints: [node.role == manager]
    command: redis-server --appendonly yes
    networks:
      - webnet
networks: 
  webnet:
```
![visualizer-with-redis.png](https://docs.docker.com/get-started/images/visualizer-with-redis.png)
