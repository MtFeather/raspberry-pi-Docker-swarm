# raspberry-pi-Docker-swarm
- Arm系統要使用docker必須使用這個核心:
  https://github.com/kennychennetman/docker/blob/master/visualizer-arm.txt

1.	需要一個docker file   
https://docs.docker.com/get-started/part2/#dockerfile
#### vim dockerfile
```docker
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
```Bash
Flask
Redis
```
#### vim app.py
```Bash
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
