FROM centos
MAINTAINER deemoprobe<deemoprobe@gmail.com>
COPY tomcat.txt /usr/local/tomcat8.txt
ADD apache-tomcat-8.5.53.tar.gz /usr/local
ADD jdk-8u271-linux-x64.tar.gz /usr/local
RUN yum install -y vim
ENV MYPATH /usr/local
WORKDIR $MYPATH
ENV JAVA_HOME /usr/local/jdk1.8.0_271
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV CATALINA_HOME /usr/local/apache-tomcat-8.5.53
ENV CATALINA_BASE /usr/local/apache-tomcat-8.5.53
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin
EXPOSE 8080
# 启动
# ENTRYPOINT [ "/usr/local/apache-tomcat-8.5.53/bin/startup.sh" ]
# CMD [ "/usr/local/apache-tomcat-8.5.53/bin/catalina.sh", "run" ]
CMD /usr/local/apache-tomcat-8.5.53/bin/startup.sh && tail -F /usr/local/apache-tomcat-8.5.53/bin/logs/catalina.out
