# raspberry-pi-Docker-swarm
- Arm系統要使用docker必須使用這個核心:
  https://github.com/kennychennetman/docker/blob/master/visualizer-arm.txt

1.	需要一個docker file
  https://docs.docker.com/get-started/part2/#dockerfile
2.	建立requirements.txt 和 app.py
3.	建置APP
    ```Bash
    docker build -t friendlyhello .
    ```
