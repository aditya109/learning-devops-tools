FROM ubuntu
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update
RUN apt install software-properties-common -y
RUN add-apt-repository ppa:deadsnakes/ppa
RUN apt update
RUN apt install python3.8 -y
RUN apt-add-repository ppa:ansible/ansible
RUN apt-get update
RUN apt install ansible
RUN ansible --version

CMD ["sleep", "600h"]

