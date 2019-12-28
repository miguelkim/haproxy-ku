### Centos7 Haproxy 컴파일 및 설정하기

###### 2019/12/28. by Miguel.

- Centos7

- Haproxy 2.0.12 ( HAProxy 2.0 is a long-term supported stable version )
  
  

1. #####  haproxy 다운받기

   * http://www.haproxy.org/

     * 2019년 12월 28일 기준으로 2.0.12 가 LTS stable version

       ```
       export haproxy_version=haproxy-2.0.12
       wget http://www.haproxy.org/download/2.0/src/${haproxy_version}.tar.gz
       ```

      

2. ##### haproxy 컴파일하기

   1. 디펜던시 설치하기

      ```
      yum install \ 
      gcc openssl-devel readline-devel systemd-devel make pcre-devel \ 
      openssl socat nmap
      ```

      

   2. 압축풀기
   
      ```
      tar -zxf ${haproxy_version}.tar.gz
      ```

      

   3. 폴더이동
   
      ```
      cd ~/${haproxy_version}
      ```

      

   4. 컴파일하기
   
      ```
      make -j $(nproc) \
      TARGET=linux-glibc \ 
      USE_OPENSSL=1 \ 
      USE_ZLIB=1 \ 
      USE_PCRE=1 \ 
      USE_SYSTEMD=1 \ 
   EXTRA_OBJS="contrib/prometheus-exporter/service-prometheus.o";
      ```

      

      - 옵션설명

        - -j $(nproc)

          : 컴파일 시 Thread는 서버 프로세스"nproc" 만큼 사용

        - TARGET=linux-glibc

          : Linux kernel 2.6.28 이상의 리눅스 ( for Linux kernel 2.6.28 and above )

        - USE_OPENSSL

          : https 사용. ( add native support for SSL )

        - USE_ZLIB

          : 데이터 압축 사용. ( In order to use zlib, simply pass "USE_ZLIB=1" )

        - USE_PCRE

          : pcre 정규식 사용. ( enable PCRE version 1, dynamic linking )

        - USE_SYSTEMD
   
          : systemctl 로 컨트롤 ( enables support for the sdnotify features of systemd )
        
        - EXTRA_OBJS="contrib/prometheus-exporter/service-prometheus.o"
        
          : status 내용을 프로메테우스 형식으로 출력
          
   
3. #####  설치하기

   1. 관리 폴더 생성

      ```
      mkdir /haproxy
      ```

      

   2. 서브 폴더 생성

      ```
      mkdir -p \ 
      /haproxy/config \ 
      /haproxy/errorfiles \ 
      /haproxy/pid \ 
      /haproxy/run \ 
      /haproxy/sbin
      ```

      

   3. 컴파일 완성한 haproxy 파일 복사

      ```
      cp ~/${haproxy_version}/haproxy /haproxy/sbin/.
      ```

      

   4. limit ( nofile, npoc ) 수정

      - 타 계정 값에 영향을 줄 수 있으므로 주의.

      ```
      cat <<EOF > /etc/security/limits.d/haproxy.conf
      *        soft    nofile    65536
      *        hard    nofile    65536
      *        soft    nproc     60000
      *        hard    nproc     100000
      root     soft    nproc     unlimited
      EOF
      ```
   
      
   
   5. log 폴더 생성
   
      ```
      mkdir /LOG_FOR_HAPROXY
      ```
   
      
   
   6. rsyslog 등록
   
      ```
      cat <<EOF > /etc/rsyslog.d/haproxy.conf
      # Collect log with UDP
      $ModLoad imudp
      $UDPServerAddress 127.0.0.1
      $UDPServerRun 514
      
      $template Haproxy, "%msg%\n"
      
      # Creating separate log files based on the severity
      local0.notice -/LOG_FOR_HAPROXY/error.log;Haproxy
      local0.* -/LOG_FOR_HAPROXY/access.log;Haproxy
      EOF
      ```
   
      
   
   7. logrotate 등록
   
      - logrotate.d 안에 파일은 반드시 644 권한이어야 함.
      - root 소유권으로 log 파일이 생성됨으로 postrotate 에 권한 소유권 명령어를 넣어둠.
   
      ```
      cat <<EOF > /etc/logrotate.d/haproxy
      /LOG_FOR_HAPROXY/*.log {
          daily
          rotate 30
          missingok
          notifempty
          nocompress
          sharedscripts
          postrotate
              /bin/kill -HUP `cat /var/run/syslogd.pid 2> /dev/null` 2> /dev/null || true
      	/usr/bin/chown haproxy:haproxy /LOG_FOR_HAPROXY/*.log
          endscript
      }
      EOF
      chmod 644 /etc/logrotate.d/haproxy;
      ```
   
      
   
   8. systemd 등록
   
      ```
      cat <<EOF > /lib/systemd/system/haproxy.service
      [Unit]
      Description=HAProxy Load Balancer
      After=network.target
      
      [Service]
      Environment="CONFIG=/haproxy/config/haproxy.cfg" "PIDFILE=/haproxy/pid/haproxy.pid" "EXTRAOPTS=-x /haproxy/run/haproxy-master.sock"
      ExecStartPre=/haproxy/sbin/haproxy -f $CONFIG -c -q $EXTRAOPTS
      ExecStart=/haproxy/sbin/haproxy -Ws -f $CONFIG -p $PIDFILE $EXTRAOPTS
      ExecReload=/haproxy/sbin/haproxy -f $CONFIG -c -q $EXTRAOPTS
      ExecReload=/bin/kill -USR2 $MAINPID
      KillMode=mixed
      Restart=always
      SuccessExitStatus=143
      Type=notify
      
      [Install]
      WantedBy=multi-user.target
      }
      EOF
      
      systemctl daemon-reload;
      systemctl enable haproxy.service;
      systemctl status haproxy.service;
      ```
   
      
   
   9. haproxy 계정 생성
   
      ```
      adduser haproxy -b /
      ```
   
      
   
   10. sudo 권한 부여 ( haproxy 계정 )
   
       ```
       chmod +w /etc/sudoers;
       echo -e "\nhaproxy      ALL=(ALL)       NOPASSWD: systemctl stop haproxy,systemctl start haproxy, systemctl restart haproxy, systemctl reload haproxy">> /etc/sudoers;
       chmod -w /etc/sudoers;
       ```
   
       
   
   11. config 설정

