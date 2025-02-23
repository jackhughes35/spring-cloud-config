FROM amazoncorretto:21.0.4
LABEL maintainer="example"

ADD target/spring-cloud-config-server*.jar app.jar
ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS $DD_AGENT_ARGS -Djava.security.egd=file:/dev/./urandom -jar /app.jar" ]