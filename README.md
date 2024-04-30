# Apache Tomcat ssl 적용과 clustering

## 목적 
```
Apache, Tomcat을 연동하여 Apache는 was, Tomcat은 server로 사용하고 clustering을 적용하여 부하를 분산한다.


또한 Apache, Tomcat 모두 ssl 을 적용하여 보안 수준을 향상한다.


궁극적으로는 Apache의 화면에서 Tomcat이 뜰 수 있도록 하는데 목적을 둔다.


```


### clustering and load balancing:
```
ssl 을 설정하기 이전에는 80번 포트를 사용해 연동될 수 있도록한다.


아래는 변경해야할 환경설정이다.


```
Apache - httpd.conf:
```
LoadModule proxy_http_module modules/mod_proxy_http.so
LoadModule proxy_ajp_module modules/mod_proxy_ajp.so
LoadModule jk_module modules/mod_jk.so
LoadModule proxy_balancer_module modules/mod_proxy_balancer.so
LoadModule slotmem_shm_module modules/mod_slotmem_shm.so
LoadModule lbmethod_byrequests_module modules/mod_lbmethod_byrequests.so
LoadModule proxy_module modules/mod_proxy.so

ServerName 
<VirtualHost *:80>
	ServerName (Apache Server IP)
	ProxyRequests Off
	ProxyPass / balancer://mycluster/
	ProxyPassReverse / balancer://mycluster
	<Proxy balancer://mycluster stickysession=JSESSIONID|jsessionid scolonpathdelim=On>
		BalancerMember http://(Tomcat1번IP):8080 route=1
		BalancerMember http://(Tomcat2번IP):8080 route=2 status=+H
	</Proxy>
</VirtualHost>
최하단부
<IfModule jk_module>
	JkWorkersFile	conf/workers.properties
	JkLogFile	logs/mod_jk.log
	JkLogLevel	info
	JkMount		/* load_balancer
</IfModule>
```
Apache – workers.properties:
```
worker.list=load_balancer
worker.load_balancer.type=lb
worker.load_balancer.balanced_workers=was1,was2 이름
worker.load_balancer.sticky_session=1

worker.was1.port=8009
worker.was1.host= (Tomcat 1번 서버의 IP)
worker.was1.type=ajp13
worker.was1.lbfactor=1
worker.was1.redirect=was2

worker.was2.port=8009
worker.was2.host= (Tomcat 2번 서버의 IP)
worker.was2.type=ajp13
worker.was2.lbfactor=1
```

Tomcat – server.xml:
```
<Engine name="Catalina" defaultHost="localhost" jvmRoute="본인이 부여할 jvm route">
<Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"
                 channelSendOptions="8">
          <Manager className="org.apache.catalina.ha.session.DeltaManager"
                   expireSessionsOnShutdown="false"
                   notifyListenersOnReplication="true"/>
          <Channel className="org.apache.catalina.tribes.group.GroupChannel">
            <Membership className="org.apache.catalina.tribes.membership.McastService"
                        address="228.0.0.4"
                        port="45564"
                        frequency="500"
                        dropTime="3000"/>
            <Receiver className="org.apache.catalina.tribes.transport.nio.NioReceiver"
                      address="auto”
                      port="4000"
                      autoBind="0"
                      selectorTimeout="5000"
                      maxThreads="6"/>
            <Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter">
              <Transport className="org.apache.catalina.tribes.transport.nio.PooledParallelSender"/>
            </Sender>
            <Interceptor className="org.apache.catalina.tribes.group.interceptors.TcpFailureDetector"/>
            <Interceptor className="org.apache.catalina.tribes.group.interceptors.MessageDispatchInterceptor"/>
          </Channel>
          <Valve className="org.apache.catalina.ha.tcp.ReplicationValve"
                 filter=""/>
          <Valve className="org.apache.catalina.ha.session.JvmRouteBinderValve"/>
          <Deployer className="org.apache.catalina.ha.deploy.FarmWarDeployer"
                    tempDir="/tmp/war-temp/"
                    deployDir="/tmp/war-deploy/"
                    watchDir="/tmp/war-listen/"
                    watchEnabled="false"/>
          <ClusterListener className="org.apache.catalina.ha.session.ClusterSessionListener"/>
        </Cluster>
```
Tomcat – test.jsp(세션 클러스터링 확인용):
```
<%@page language="java" %>
<html>
<body>
<h1><font color="red">Session serviced by machine3</font></h1>
<table align="center" border="1">
  <tr>
  <td>
     Session ID
  </td>
  <td>
     <%= session.getId() %></td>
  </td>
  </tr>
  <tr>
  <td>session value
  </td>
  <td>
  <%= (String)session.getAttribute("test")%>
  </td>
  </tr>
  <tr>
  <td>
    Created on
  </td>
  <td>
    <%= session.getCreationTime() %>
  </td>
  </tr>
</table>
</body>
</html>
```

아파치에서 다음과 같이 세션 클러스터링이 완료 되었다면 이제 ssl 설정을 해야 한다.



 
(was부분은 본인이 부여한 jvm route로 뜨는 것이다.)




ssl 설정 Apache – httpd.conf:
```
LoadModule ssl_module modules/mod_ssl.so
include conf/extra/httpd-ssl.conf(주석 해제 없으면 따로 추가)

 	- SSLPassPharseDialog exec:C:\Apache24\conf\ssl_passwd.bat
- SSLCertificateFile “C:\cert/cert.pem”
- SSLCertificateKey File “C:\cert/key.pem”
- SSLCertificateChainFile “C:\cert/DigiCertCA.pem”
- httpd.conf 파일의 기존에 설정했던 부분에서 수정
<VirtualHost *:443>
	ServerName 아파치 ip
	SSLEngine on
	SSLCertificateFile C:\ssl\private.crt
	SSLCertificateKeyFile C:\ssl\private.key
	SSLProxyEngine on
	SSLProxyVerify none
	SSLProxyCheckPeerCN off
	SSLProxyCheckPeerName off
	SSLProxyCheckPeerExpire off
		ProxyRequests Off
		ProxyPass / balancer://mycluster/
		ProxyPassReverse / balancer://mycluster
	<Proxy balancer://mycluster stickysession=JSESSIONID|jsessionid scolonpathdelim=On>
		BalancerMember http://톰캣1번 ip:8080 route=1
		BalancerMember http://톰캣2번 ip:8080 route=2 status=+H

	</Proxy>
</VirtualHost> 

ssl 설정 Tomcat:
Service 섹션 8080 아래에 다음 내용 추가
	<Connector port="443" protocol="HTTP/1.1"
	    SSLEnabled="true" maxThreds="150" scheme="https"
	    secure="true" clientAuth="false"
	    SSLCertificateFile="keystore/private.crt" SSLCertificateKeyFile="keystore/private.key"
	    SSLCACertificateFile="keystore/rootCA.pem" sslProtocol="TLS"/>

```
Tomcat SSL Test:
``` 
- http://slproweb.com/products/Win32OpenSSL.html 접속 
- Win64 OpenSSL v1.1.1k 설치
- 설치 후 환경 변수 설정 - C:\Program Files\OpenSSL-Win64 폴더에 설치되었다면 환경 변수 셋팅하지 않아도 된다.
- OPENSSL_CONF=C:\OpenSSL-Win32\bin\openssl.cfg
- PATH=C:\OpenSSL-Win32\bin
- 명령 프롬프트를 열고 openssl 입력시 기동
- version 확인 방법은 openssl 접속 후 version 입력
```
 
### 1. RSA 개인키 생성
```
우선적으로 명령 프롬프트 실행
(openssl not found가 뜰 경우 C:\Program Files\OpenSSL-Win64\bin 디렉토리로 이동 후 실행)
- openssl genrsa -out private.key 2048 입력
- 결과
Generating RSA private key, 2048 bit long modulus
....................................+++++
..........+++++
e is 65537 (0x10001)
```

### 2. RSA 개인키를 이용해서 RSA 공개키를 생성
```
- openssl rsa -in private.key -pubout -out public.key 입력
- 결과
writing RSA key
```

### 3. CSR(인증요청서) 생성
```
- openssl req -new -key private.key -out private.csr 입력
필요한 부분 입력해주기
- 결과(굵게 표시되어 있는 부분 입력)
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:KR
State or Province Name (full name) [Some-State]:_
Locality Name (eg, city) []:Seoul
Organization Name (eg, company) [Internet Widgits Pty Ltd]:bluexmas
Organizational Unit Name (eg, section) []:root CA
Common Name (e.g. server FQDN or YOUR name) []:test.iptime.org
Email Address []:test@test.com
 
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:test
An optional company name []:test
```

### 4. CRT 생성

```
4.1 rootCA.key 생성
- OpenSSL 접속
- genrsa [암호화 알고리즘] -out [키이름] 2048 입력
Ex) genrsa -aes256 -out rootCA.key 2048


4.2 rootCA 사설 CSR 생성
- OpenSSL 접속
- req -x509 -new -nodes -key rootCA.key -days 3650 -out rootCA.pem
 안될 경우
- req -config ./openssl.cnf -x509 -new -nodes -key rootCA.key -days 3650 -out rootCA.pem
- 결과(굵게 표시되어 있는 부분 입력)
Enter pass phrase for rootCA.key: [패스워드 입력]
You are about to be asked to enter information that will be incorporated
Into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
----
Country Name <2 letter code> [AU]:KR
State or Province Name <full name> [Some-State]:Seoul
Locality Name <eg, city> []:Seoul
Organization Name <eg, company> [Internet Widgits Pty Ltd]
Organizational Unit Name <eg, section> []:mobile
Common Name <e.g. server FQDN or YOUR name> []:localhost
Email Address []: test@test.com


4.3 CRT 생성
- OpenSSL 접속
- x509 -req -in private.csr -CA rootCA.pem -Cakey rootCA.key -CAcreateserial -out private.crt -days 3650 입력
// 안되면
- x509 -req -config ./openssl.cnf -in private.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out private.crt -days 3650 입력

- 결과
Signature ok
Subject=/C=KR/ST=Seoul/=L=Seoul/O=test/OU=develop/CN=localhost/emailAddress=test@test.com
Getting CA Private Key
Enter pass phrase for rootCA.key: [입력]
```


이제 tomcat과 Apache에 다음 파일을 해당 위치에 놓아두면 된다.

참조
https://bluexmas.tistory.com/1122
https://inpa.tistory.com/entry/TOMCAT-%E2%9A%99%EF%B8%8F-SSL-%EC%84%A4%EC%A0%95-https
https://cheezred.tistory.com/124
****
