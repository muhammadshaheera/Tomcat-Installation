# Tomcat-Installation

1. Install Java and check compatibilty from https://tomcat.apache.org/whichversion.html 

2. Download Tomcat from https://tomcat.apache.org/download-11.cgi or `wget https://dlcdn.apache.org/tomcat/tomcat-11/v11.0.6/bin/apache-tomcat-11.0.6.tar.gz`

3. Run `mkdir -p /opt/tomcat`

4. Run `tar xzvf apache-tomcat-11.0.6.tar.gz --strip-components=1`

5. Add user `useradd tomcat` and group `groupadd tomcat`

6. Run `chown -R tomcat:tomcat /opt/tomcat`

7. Run `sh -c 'chmod +x /opt/tomcat/bin/*.sh'`

8. Check your java link by `readlink -f $(which java)`. Below is the example of output(Only bold part of path will be used):
     **/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.392.b08-2.el7_9.x86_64**/jre/bin/java

9. Create a Service file by `vim /etc/systemd/system/tomcat.service` and paste below content:
```
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target
 
[Service]
Type=oneshot
RemainAfterExit=yes
 
User=tomcat
Group=tomcat
 
Environment="JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.392.b08-2.el7_9.x86_64/"
Environment="JAVA_OPTS=-Djava.security.egd=file:///dev/urandom -Djava.awt.headless=true"
Environment="CATALINA_BASE=/opt/tomcat"
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"
ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh
 
[Install]
WantedBy=multi-user.target
```

10. Run below:
  a. `systemctl daemon-reload`
  b. `systemctl start tomcat` and `systemctl enable tomcat` and `systemctl status tomcat`

11. Verify using `localhost:8080` from browser

12. Take below steps to implement a self-signed SSL certificate:
  a. Generate a Private Key and Self-Signed Certificate
  `openssl req -x509 -newkey rsa:2048 -keyout key.pem -out certificate.crt`
  b. Combine the Private Key and Certificate into a PKCS12 File
  `openssl pkcs12 -export -out keystore.p12 -inkey key.pem -in certificate.crt`
  c. Import the PKCS12 into a Java Keystore (JKS)
  `keytool -importkeystore -srckeystore keystore.p12 -srcstoretype PKCS12 -destkeystore keystore.jks`
  d. Verify the Keystore Contents
  `keytool -list -v -keystore /opt/tomcat/conf/keystore.jks`
  e. Verification from Server
  `openssl s_client -connect localhost:8443 -CAfile /opt/tomcat/conf/certificate.crt`
  `curl -vv --cacert /opt/tomcat/conf/certificate.crt https://localhost:8443/`
  f. Edit `/opt/tomcat/conf/server.xml` and add:
```
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
               maxThreads="150" SSLEnabled="true"
               maxParameterCount="1000" keystoreFile="/opt/tomcat/conf/keystore.jks" keystorePass="123456"
               scheme="https" secure="true" clientAuth="false" sslProtocol="TLS" 
               >
<!--
        <SSLHostConfig>
            <Certificate certificateKeystoreFile="/opt/tomcat/conf/keystore.jks"
                         type="RSA" />
        </SSLHostConfig>
-->
    </Connector>
```

13. To run multiple Tomcats with Self-Signed Certificate on a single server
  a. Install tomcat in separate directories e.g. directory /opt/tomcat2 and change its user and group to “tomcat” by `chown -R tomcat: tomcat /opt/tomcat2`
  b. Create separate service files and edit below parameters with your above directory
      a.	CATALINA_BASE
      b.	CATALINA_HOME
      c.	CATALINA_PID
      d.	ExecStart
      e.	ExecStop
  c.	Goto /opt/tomcat2/conf and edit below ports in server.xml as per requirement
      a.	`Server port="8005" shutdown="SHUTDOWN"`
      b.	`Connector port="8080" protocol="HTTP/1.1” and “redirectPort="8443"`
  d.	Create certificate, private key, keystore.p12 and keystore.jks by above given method
  e.	Open server.xml and edit below parameters as per requirement
```
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
               maxThreads="150" SSLEnabled="true"
               maxParameterCount="1000" keystoreFile="/opt/tomcat/conf/keystore.jks" keystorePass="123456"
               scheme="https" secure="true" clientAuth="false" sslProtocol="TLS" 
               >
<!--
        <SSLHostConfig>
            <Certificate certificateKeystoreFile="/opt/tomcat/conf/keystore.jks"
                         type="RSA" />
        </SSLHostConfig>
-->
    </Connector>
```
