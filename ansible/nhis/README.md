# 웹 서버 중앙 관리를 위한 간단 샘플 예제

1. [앤서블 구성 파일](README.md#1-앤서블-구성-파일)<br>
2. [앤서블 인벤토리](README.md#2-앤서블-인벤토리)<br>
3. [웹 서버 관리를 위한 샘플 플레이북](README.md#3-웹-서버-관리를-위한-샘플-플레이북)<br>
4. [테스트](README.md#4-테스트)<br>

<br>
<br>

## 1. 앤서블 구성 파일

~/[ansible.cfg](ansible.cfg) 파일
```conf
[defaults]
display_skipped_hosts = False
inventory = /redhat/nhis/inventory/hosts #1
log_path = /redhat/nhis/logs/ansible.log #2
roles_path = ./roles:/redhat/nhis/roles:/usr/share/ansible/roles:/etc/ansible/roles
collections_path = ./collections:/redhat/nhis/collections:/usr/share/ansible/collections
```
1. 앤서블 인벤토리 파일 지정
2. 로그 파일 위치 지정
<br>
<br>

## 2. 앤서블 인벤토리

~/inventory/[hosts](./inventory/hosts) 파일
```conf
[all:vars]
ansible_ssh_user=root   #1
ansible_ssh_pass=redhat #2

[web_all] #3
rhel94-web1
rhel88-web2
rhel10-web3
```
1. SSH로 접속할 사용자
2. SSH 접속시 사용할 암호
3. 앤서블 그룹
   * 다양한 호스트를 멤버로 구성 가능
   * 하나의 호스트가 여러 그룹에 멤버로 구성 가능
   * 그룹 안에 그룹 구성 가능

> [!WARNING]
> 앤서블 인벤토리 파일이 SSH 사용자 및 암호를 설정하는 보안상 위험합니다. 해당 예제에서는 간단한 샘플을 보여주기 위해서 설정한 것이고, 실제로는 다음과 형태로 구성합니다.
> * 앤서블 플레이북 실행 시에, 프롬프트를 통해 암호를 입력
> * ansible-vault를 통해 암호화된 파일에 저정된 SSH 정보를 이용
<br>
<br>

## 3. 웹 서버 관리를 위한 샘플 플레이북

~/playbooks/[manage-webs.yaml](./playbooks/manage-webs.yaml)
```yaml
---
- name: Manage Web Servers
  hosts: "{{ HOST_NAME | default('all') }}"
  gather_facts: false
  tasks:

    - name: Ping "{{ HOST_NAME }}" #1
      ansible.builtin.ping:
      when: OPS_CMD|default('None') is match("ping")

    - name: Install httpd package to "{{ HOST_NAME }}" #2
      ansible.builtin.dnf:
        name: httpd
        state: present
      when: OPS_CMD|default('None') is match("install")

    - name: Check status of httpd in "{{ HOST_NAME }}" #3
      ansible.builtin.service_facts:
      when: OPS_CMD|default('None') is match("status")

    - name: Print status of httpd in "{{ HOST_NAME }}" #4
      ansible.builtin.debug:
        var: ansible_facts.services['httpd.service']
      when: OPS_CMD|default('None') is match("status")

    - name: Start httpd service in "{{ HOST_NAME }}" #5
      ansible.builtin.service:
        name: httpd
        state: started
      when: OPS_CMD|default('None') is match("start")

    - name: Stop httpd service in "{{ HOST_NAME }}" #6
      ansible.builtin.service:
        name: httpd
        state: stopped
      when: OPS_CMD|default('None') is match("stop")
```
1. 웹 서비스를 실행 중인 호스트로 연결 가능한지 확인
2. 필요하면 해당 호스트에 웹 패키지 설치 및 구성
   * 샘플에서는 간단하게 패키지만 설치
3. 웹 서비스 정보 수집
4. 수집된 정보로부터 웹 서비스 상태 확인
5. 지정된 호스트의 웹 서비스 시작
6. 지정된 호스트의 웹 서비스 중단
<br>
<br>

## 4. 테스트

### 4.1 웹 서비스를 운영 중인 호스트 연결 테스트

실행 명령어
```bash
ansible-playbook playbooks/manage-webs.yaml -e HOST_NAME='all' -e OPS_CMD='ping'
```
* 변수 *HOST_NAME*에 모든 노드를 나타내는 `all`을 지정
* 실행 명령어를 지정하는 변수 *OPS_CMD*에 ping 테스트를 지정하는 `ping`을 지정

실행 결과
```
[root@black /redhat/nhis]# ansible-playbook playbooks/manage-webs.yaml -e HOST_NAME='all' -e OPS_CMD='ping'

PLAY [Manage Web Servers] ********************************************************************************

TASK [Ping "all"] ****************************************************************************************
ok: [rhel94-web1]
ok: [rhel88-web2]
ok: [rhel10-web3]

PLAY RECAP ***********************************************************************************************
rhel10-web3                : ok=1    changed=0    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0
rhel88-web2                : ok=1    changed=0    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0
rhel94-web1                : ok=1    changed=0    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0

[root@black /redhat/nhis]# 
```
<br>

### 4.2 웹 서비스를 운영 중인 모든 호스트에서 웹 서비스 상태 확인

실행 명령어
```bash
ansible-playbook playbooks/manage-webs.yaml -e HOST_NAME='all' -e OPS_CMD='status'
```
* 변수 *HOST_NAME*에 모든 노드를 나타내는 `all`을 지정
* 실행 명령어를 지정하는 변수 *OPS_CMD*에 웹 서비스 상태를 확인하는 `status`를 지정

실행 결과
```
[root@black /redhat/nhis]# ansible-playbook playbooks/manage-webs.yaml -e HOST_NAME='all' -e OPS_CMD='status'

PLAY [Manage Web Servers] ********************************************************************************

TASK [Check status of httpd in "all"] ********************************************************************
ok: [rhel94-web1]
ok: [rhel10-web3]
ok: [rhel88-web2]

TASK [Print status of httpd in "all"] ********************************************************************
ok: [rhel94-web1] => {
    "ansible_facts.services['httpd.service']": {
        "name": "httpd.service",
        "source": "systemd",
        "state": "inactive",
        "status": "disabled"
    }
}
ok: [rhel88-web2] => {
    "ansible_facts.services['httpd.service']": {
        "name": "httpd.service",
        "source": "systemd",
        "state": "inactive",
        "status": "disabled"
    }
}
ok: [rhel10-web3] => {
    "ansible_facts.services['httpd.service']": {
        "name": "httpd.service",
        "source": "systemd",
        "state": "inactive",
        "status": "disabled"
    }
}

PLAY RECAP ***********************************************************************************************
rhel10-web3                : ok=2    changed=0    unreachable=0    failed=0    skipped=4    rescued=0    ignored=0
rhel88-web2                : ok=2    changed=0    unreachable=0    failed=0    skipped=4    rescued=0    ignored=0
rhel94-web1                : ok=2    changed=0    unreachable=0    failed=0    skipped=4    rescued=0    ignored=0

[root@black /redhat/nhis]# 
```
<br>

### 4.3 지정된 호스트의 웹 서비스 중단 및 시작

#### 4.3.1 웹 서비스 시작

실행 명령어
```bash
ansible-playbook playbooks/manage-webs.yaml -e HOST_NAME='rhel88-web2' -e OPS_CMD='start'
```
* 변수 *HOST_NAME*에 특정 노드를 나타내는 `rhel88-web2`를 지정
* 실행 명령어를 지정하는 변수 *OPS_CMD*에 웹 서비스를 시작하는 `start`를 지정

실행 결과
```
[root@black /redhat/nhis]# ansible-playbook playbooks/manage-webs.yaml -e HOST_NAME='rhel88-web2' -e OPS_CMD='start'

PLAY [Manage Web Servers] ********************************************************************************

TASK [Start httpd service in "rhel88-web2"] **************************************************************
changed: [rhel88-web2]

PLAY RECAP ***********************************************************************************************
rhel88-web2                : ok=1    changed=1    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0

[root@black /redhat/nhis]# 
```
* 호스트 *rhel88-web2*에서 httpd 서비스가 시작됨

#### 4.3.2 웹 서비스 중지

실행 명령어
```bash
ansible-playbook playbooks/manage-webs.yaml -e HOST_NAME='rhel88-web2' -e OPS_CMD='stop'
```
* 변수 *HOST_NAME*에 특정 노드를 나타내는 `rhel88-web2`를 지정
* 실행 명령어를 지정하는 변수 *OPS_CMD*에 웹 서비스를 중지하는 `stop`를 지정

실행 결과
```
[root@black /redhat/nhis]# ansible-playbook playbooks/manage-webs.yaml -e HOST_NAME='rhel88-web2' -e OPS_CMD='stop'

PLAY [Manage Web Servers] ********************************************************************************

TASK [Stop httpd service in "rhel88-web2"] ***************************************************************
changed: [rhel88-web2]

PLAY RECAP ***********************************************************************************************
rhel88-web2                : ok=1    changed=1    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0

[root@black /redhat/nhis]#
```
* 호스트 *rhel88-web2*에서 httpd 서비스가 중지됨
<br>

### 4.4 지정된 그룹의 웹 서비스 중단 및 시작

#### 4.4.1 웹 서버 그룹에서 웹 서비스 시작

실행 명령어
```bash
ansible-playbook playbooks/manage-webs.yaml -e HOST_NAME='web_all' -e OPS_CMD='start'
```
* 변수 *HOST_NAME*에 웹 서버 그룹을 나타내는 `web_all`를 지정
* 실행 명령어를 지정하는 변수 *OPS_CMD*에 웹 서비스를 시작하는 `start`를 지정

실행 결과
```
[root@black /redhat/nhis]# ansible-playbook playbooks/manage-webs.yaml -e HOST_NAME='web_all' -e OPS_CMD='start'

PLAY [Manage Web Servers] ********************************************************************************

TASK [Start httpd service in "web_all"] ******************************************************************
changed: [rhel88-web2]
changed: [rhel94-web1]
changed: [rhel10-web3]

PLAY RECAP ***********************************************************************************************
rhel10-web3                : ok=1    changed=1    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0
rhel88-web2                : ok=1    changed=1    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0
rhel94-web1                : ok=1    changed=1    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0

[root@black /redhat/nhis]# 
```
* 웹 서버 그룹 `web_all`의 멤버인 호스트 *rhel94-web1*, *rhel88-web2*, *rhel10-web3*에서 httpd 서비스가 시작됨

#### 4.4.2 웹 서버 그룹에서 웹 서비스 중지

실행 명령어
```bash
ansible-playbook playbooks/manage-webs.yaml -e HOST_NAME='web_all' -e OPS_CMD='stop'
```
* 변수 *HOST_NAME*에 웹 서버 그룹을 나타내는 `web_all`를 지정
* 실행 명령어를 지정하는 변수 *OPS_CMD*에 웹 서비스를 중지하는 `stop`를 지정

실행 결과
```
[root@black /redhat/nhis]# ansible-playbook playbooks/manage-webs.yaml -e HOST_NAME='web_all' -e OPS_CMD='stop'

PLAY [Manage Web Servers] ********************************************************************************

TASK [Stop httpd service in "web_all"] *******************************************************************
changed: [rhel94-web1]
changed: [rhel88-web2]
changed: [rhel10-web3]

PLAY RECAP ***********************************************************************************************
rhel10-web3                : ok=1    changed=1    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0
rhel88-web2                : ok=1    changed=1    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0
rhel94-web1                : ok=1    changed=1    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0

[root@black /redhat/nhis]# 
```
* 웹 서버 그룹 `web_all`의 멤버인 호스트 *rhel94-web1*, *rhel88-web2*, *rhel10-web3*에서 httpd 서비스가 중됨
<br>
<br>

------
[차례](/README.md)