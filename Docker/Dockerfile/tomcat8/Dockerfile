# 基础镜像centos:latest
FROM centos
# 作者签名
MAINTAINER deemoprobe<deemoprobe@gmail.com>
# 拷贝宿主机当前目录下文件
COPY tomcat.txt /usr/local/tomcat8.txt
# 添加Tomcat安装包并解压至/usr/local
ADD apache-tomcat-8.5.53.tar.gz /usr/local
# 添加jdk安装包并解压至/usr/local
ADD jdk-8u271-linux-x64.tar.gz /usr/local
# 安装vim
RUN yum install -y vim
# 设置环境变量
ENV MYPATH /usr/local
# 指定工作目录，使用ENV设定的环境变量
WORKDIR $MYPATH
# 配置JDK环境
ENV JAVA_HOME /usr/local/jdk1.8.0_271
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV CATALINA_HOME /usr/local/apache-tomcat-8.5.53
ENV CATALINA_BASE /usr/local/apache-tomcat-8.5.53
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin
# 声明端口8080
EXPOSE 8080
# 启动
# ENTRYPOINT [ "/usr/local/apache-tomcat-8.5.53/bin/startup.sh" ]
# CMD [ "/usr/local/apache-tomcat-8.5.53/bin/catalina.sh", "run" ]
CMD /usr/local/apache-tomcat-8.5.53/bin/startup.sh && tail -F /usr/local/apache-tomcat-8.5.53/bin/logs/catalina.out
