# 초보자를 위한 앤서블
> 여러 노드에 패키지를 간편하게 배포하기 위한 도구로 앤서블을 실습합니다.
> Udemy 추천 강좌인 [처음부터 설치하며 배우는 앤서블](https://www.udemy.com/course/using_ansible_for_simple_configuration/) 강좌를 그대로 실습하고 학습흡니다

## 참고자료
* [처음부터 설치하며 배우는 앤서블](https://www.udemy.com/course/using_ansible_for_simple_configuration/)
* [Ansible: Getting Started](https://docs.ansible.com/ansible/latest/user_guide/intro_getting_started.html)
* [Ansible: Post-Install Setup](https://hvops.com/articles/ansible-post-install/)
* [psyoblade's vagrant-for-dummies](https://github.com/psyoblade/vagrant-for-dummies/)
* [ruanbekker's ansible-docker-swarm](https://github.com/psyoblade/ansible-docker-swarm/)
* [Ansible 아키텍처](https://velog.io/@hanblueblue/%EB%B2%88%EC%97%AD-Ansible)
* [Ansible Inventory, Playbooks and Roles](https://velog.io/@hanblueblue/%EB%B2%88%EC%97%AD-Ansible2-%ED%94%8C%EB%A0%88%EC%9D%B4%EB%B6%81)



## 1. 환경 구성

### 1-1. 버추얼박스 + 센트OS 구성
* [버추얼박스](https://www.virtualbox.org/wiki/Downloads) 사이트에서 다운로드 후 설치
* [CentOS](http://isoredirect.centos.org/centos/8/isos/x86_64/) 사이트에서 ISO 파일 가운데 Minimal 파일을 다운로드합니다
  - KDump 는 용량상 제외하고, Disk 설정은 기본 설정인 8GB 그대로 지정합니다
  - 네트워크 설정에서 호스트의 이름을 ansible-server 로 변경하였습니다
  - 패스워드는 간단하게 작성합니다 (CentOS 의 경우 root 패스워드를, Ubuntu 의 경우 root 계정을 별도로 생성해야 합니다)
  - 설치가 완료되면 Reboot 후, 로그인 합니다
  - Bridge 네트워크 이므로 IP 를 할당받지 못했으며, 워커 이미지를 복제해 둔 이후에 IP를 할당할 예정입니다
```bash
bash> ip addr  # ubuntu: ifconfig, centos: ip addr
poweroff
```
* 워커로 사용할 노드 3개(ansible-node01~03)를 복제하되, 완전한 복제로 생성합니다
  - 네트워크 주소는 모두 다시 생성해야 편합니다
  - 모든 서버의 메모리는 512mb 정도로 줄이고 재시작 하여 호스트명을 변경 후, 재시작하여 호스트명을 확인합니다
```bash
bash> hostnamectl set-hostname ansible-node01
reboot
```
* 각 노드에 IP 및 DNS 설정을 합니다
  - 로컬 네트워크 설정이 192.168.0.22, 255.255.255.0, 192.168.0.1 구성이므로 
  - 해당 설정을 nmtui 도구로 변경 후 시스템 네트워크 서비스를 재시작 합니다
  - 게이트웨이 및 모든 노드에 ping 확인을 합니다
```bash
bash> cat hosts
ansible-server 192.168.0.10 # subnet:255.255.255.0, gateway:192.168.0.1
ansible-node01 192.168.0.11
ansible-node02 192.168.0.12
ansible-node03 192.168.0.13

bash> nmtui # IPv4 설정에서
# Addresses : 192.168.0.10/24
# Gateway   : 192.168.0.1
# [X] Automatically connect 선택 후 [OK]

bash> systemctl restart network # 모든 노드 설정 및 재시작 후 ping 확인
ping 192.168.0.1
ping 192.168.0.10
ping 192.168.0.11
ping 192.168.0.12
ping 192.168.0.13

```


### 1-2. 앤서블 설치
* 앤서블을 yum 을 통해 설치합니다
  - Mirror 호스트를 찾을 수 없으므로 DNS 설정을 먼저 해야합니다
  - KT 공개 DNS 서버인 168.126.63.1~2 서버를 설정합니다 (/etc/resolv.conf)
  - 최신 패캐지 업데이트를 수행하고 나서 앤서블 설치가 가능합니다
```bash
bash> cat /etc/resolv.conf
nameserver 168.126.63.1

bash> systemctl resetart network
ping google.com

bash> yum repolist  # 명령어를 통해서 레포지토리를 확인하고

bash> yum install epel-release -y  # Extra Package for Enterprise Linux 를 업데이트합니다

bash> yum install ansible -y  # 앤서블 코어를 설치합니다
```
* 각종 통신사의 DNS 서버를 참고합니다
```text
[SK]
기본 DNS 서버 : 219.250.36.130
보조 DNS 서버 : 210.220.163.82

[KT]
기본 DNS 서버 : 168.126.63.1 / kns.kornet.net
보조 DNS 서버 : 168.126.63.2 / kns2.kornet.net

[LG]
기본 DNS 서버 : 164.124.101.2
보조 DNS 서버 : 203.248.252.2

[Google]
기본 DNS 서버 : 8.8.8.8
보조 DNS 서버 : 8.8.4.4
```


### 1-3. 앤서블 명령어 테스트
* 호스트 관리 파일 (/etc/ansible/hosts)
```bash
bash> cat /etc/ansible/hosts  # 아래의 nginx 는 해당 섹션의 그룹을 지정하는 키워드입니다
[nginx]
192.168.0.10
192.168.0.11
192.168.0.12
```
* 각 노드와 public key 교환 확인
```bash
bash> ansible all -m ping  # 명령 이후 노드 수 만큼 yes 타이핑 

bash> ansible all -m ping -k  # -k 옵션으로 skip exchange public key
```
* 앤서블 설정파일 (/etc/ansible/ansible.cfg)
  - 필요에 따라 기본설정을 확인하고 변경해서 사용할 수 있습니다


### 1-4. 앤서블 주요 파라메터
```bash
bash> ansible -i (--inventory-file) <file-name>  # /etc/ansible/hosts 파일 대신 다른 파일을 지정
ansible all -i ./test -m ping -k

bash> ansible -m (--module-name) <module-name>  # 사전 정의되어 있는 파이썬 모듈의 이름을 지정
ansible nginx -m ping -k

bash> ansible -k (--ask-pass)  # 패스워드를 입력을 통해 로그인하겠다
ansible nginx -m ping -k

bash> ansible -K (--ask-become-pass)  # 관리자(sudo) 권한으로 수행을 하겠다 (2번 패스워드 입력필요)
ansible nginx -m ping -k -K

bash> ansible all --list-hosts  # 명령을 수행하지 않고 대상 호스트 목록만 출력
ansible all -i ./teset -m ping -k --list-hosts
```


## 2. 다수의 시스템에 작업 수행하기

### 2-1. 쉘 모듈
```bash
bash> ansible nginx -m shell -a "uptime" -k  # 각 노드의 업타임 확인하기

bash> ansible nginx -m shell -a "df -h" -k  # 각 노드의 디스크 확인

bash> ansible nginx -m shell -a "free -h" -k  # 각 노드의 메모리 확인

bash> ansible nginx -m copy -a "src=./test dest=/tmp" -k  # 각 노드에 현재 test 파일을 복사
ansible nginx -m shell -a "ls /tmp/test" -k  # 명령으로 전송되었는지 여부를 확인 

bash> ansible nginx -m yum -a "name=httpd state=present" -k  # 각 노드에 httpd 데몬을 존재하는(present) 상태(state)로 만들어라
ansible nginx -m shell -a "yum list installed | grep httpd" -k  # 경고가 뜨긴 하지만 설치 여부를 확인할 수 있다
ansible nginx -m shell -a "systemctl status httpd" -k  # 명령어를 통해서 서비스가 떠 있지 않음을 알 수 있다
```


### 2-2. 플레이북 기초
> 대량의 서버에 항상 멱등하게 동작하는 명령을 수행할 수 있는 YAML 파일을 이용한 기능
* 로컬호스트의 호스트 파일에 lastnode 라는 섹션을 추가하는 예제를 실습합니다
```bash
bash> cat add-host.yml
---
- name: ansible_vim
  hosts: localhost

  tasks:
    - name: add ansible host
      blockinfile:
        path: /etc/ansible/hosts
        block: |
          [lastnode]
          192.168.1.13

bash> ansible lastnode -m ping -k  # 방금 추가한 노드만 선택이 가능한 지 테스트합니다
```

### 2-3. 플레이북 활용한 서비스 설치
* 플레이북 설치를 위한 YAML 파일을 생성합니다
```bash
bash> cat add-ninx.yml
---
- hosts: nginx
  remote_user: root
  tasks:
    - name: install epel-release
      yum: name=epel-release state=latest
    - name: install nginx web server
      yum: name=nginx state=present
    - name: install nginx web server
      service: name=nginx state=started
```
* 생성된 파일으로 각 노드에 nginx 를 설치합니다
  - 대상 노드에서는 정상적으로 nginx 가 떠 있지만, 외부에서는 확인이 불가능합니다.
  - 포트 단위 혹은 IP 수준에서 막아야하지만 테스트이므로 방화벽 서비스를 중지 후, 노드에 직접 접속해봅니다
```bash
bash> ansible-playbook add-nginx.yml -k

bash> ansible nginx -m shell -a "systemctl stop firewalld" -k
```
* 이번에는 다운로드 받은 index.html 파일을 업로드하는 코드를 추가합니다
  - 로컬에 nginx 메인 페이지를 다운로드 합니다
```bash
bash> curl -o index.html https://www.nginx.com

bash> cat add-nginx.yml
...
    - name: deploy index.html
      copy: src=./index.html dest=/usr/share/nginx/html mode=0644
...
```

