# LetsEncypt 이용한 공인 인증서 생성 절차


- Env : Ubuntu16.04
- HTTP 서버 접속을 통한 인증 수행으로 웹서버 등록 필요
```
sudo apt install letsencrypt
```
- 도메인 등록
- 웹서버 등록
  - www.domain.com
  - admin.domain.com
- 인증서 생성: webroot-path 하위에 인증파일 생성하여 도메인 소유를 점검
```
sudo letsencrypt certonly --webroot --webroot-path=/home/cloud-user/nginx/web/ -d www.domain.com
sudo letsencrypt certonly --webroot --webroot-path=/home/cloud-user/nginx/admin/ -d admin.domain.com
```
- 인증서 확인
```
$ sudo ls /etc/letsencrypt/live/www.domain.com
total 8
drwxr-xr-x 2 root root 4096 Mar 12 13:15 .
drwx------ 3 root root 4096 Mar 12 13:15 ..
lrwxrwxrwx 1 root root   46 Mar 12 13:15 cert.pem -> ../../archive/www.x1dev.crossent.com/cert1.pem
lrwxrwxrwx 1 root root   47 Mar 12 13:15 chain.pem -> ../../archive/www.x1dev.crossent.com/chain1.pem
lrwxrwxrwx 1 root root   51 Mar 12 13:15 fullchain.pem -> ../../archive/www.x1dev.crossent.com/fullchain1.pem
lrwxrwxrwx 1 root root   49 Mar 12 13:15 privkey.pem -> ../../archive/www.x1dev.crossent.com/privkey1.pem
```
- Haproxy용 인증서 생성
```
$ cd /etc/letsencrypt/live/www.domain.com
$ cat cert.pem fullchain.pem privkey.pem > /etc/haproxy/admin.pem
```
