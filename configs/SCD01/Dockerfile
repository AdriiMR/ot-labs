FROM tomcat:9.0.122-jre8-temurin

RUN rm -rf /usr/local/tomcat/webapps/*

RUN mkdir -p /var/lib/scadabr/db

COPY . /usr/local/tomcat/webapps/ScadaBR/

ENV CATALINA_OPTS="-Xms256m -Xmx512m"

EXPOSE 8080

CMD ["catalina.sh", "run"]
