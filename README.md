### Centos7 Haproxy 컴파일 및 설정하기

###### 2019/12/28. by Miguel.

- Centos7
- Haproxy 2.0.12 ( HAProxy 2.0 is a long-term supported stable version )
  

1. #####  haproxy 다운받기

   * http://www.haproxy.org/

     * 2019년 12월 28일 기준으로 2.0.12 가 LTS stable version

     * ```
       wget http://www.haproxy.org/download/2.0/src/haproxy-2.0.12.tar.gz
       ```

       

2. ##### haproxy 컴파일하기

   1. 디펜던시 설치하기

      ```
      yum install gcc openssl-devel readline-devel systemd-devel make pcre-devel
      ```

      

   2. 압축풀기

      ```
      tar -zxf haproxy-2.0.12.tar.gz
      ```

      

   3. 폴더이동

      ```
      cd haproxy-2.0.12
      ```

      

   4. 컴파일하기

      - 옵션설명

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

      ```
      make -j $(nproc) \
      TARGET=linux-glibc \ 
      USE_OPENSSL=1 \ 
      USE_ZLIB=1 \ 
      USE_PCRE=1 \ 
      USE_SYSTEMD=1 \ 
      EXTRA_OBJS="contrib/prometheus-exporter/service-prometheus.o";
      ```

      

