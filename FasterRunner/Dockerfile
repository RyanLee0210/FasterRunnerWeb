FROM python:3.5.2

MAINTAINER liwaiqiang

ENV LANG C.UTF-8
ENV TZ=Asia/Shanghai

WORKDIR /opt/workspace/FasterRunner/

COPY . .

RUN  pip3 install -r ./requirements.txt

EXPOSE 5000

CMD bash ./start.sh



