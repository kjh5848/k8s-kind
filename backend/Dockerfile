FROM eclipse-temurin:21-jdk

RUN apt-get update && apt-get install -y git

WORKDIR /var/current/app

COPY entrypoint.sh /entrypoint.sh

ENTRYPOINT ["/bin/bash", "/entrypoint.sh"]
