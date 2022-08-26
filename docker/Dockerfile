FROM accumulo:2.1.0

COPY settings.xml /tmp/
COPY pom.xml /tmp/

RUN cd /opt/accumulo/lib && \
    yum clean all && \
    yum install -y maven && \
    mkdir /tmp/build && \
    cd /tmp/build && \
    mv /tmp/pom.xml . && \
    mvn -s /tmp/settings.xml clean dependency:copy-dependencies && \
    cp target/dependency/*.jar /opt/accumulo/lib && \
    cd /tmp && \
    rm -rf /tmp/build && \
    rm -rf /var/cache/yum
