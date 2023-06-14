# otus-ansible

Проверил настройки хоста ансиблом
```
otus-ansible$ ansible nginx -m ping
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}

otus-ansible$ ansible nginx -m command -a "uname -r"
nginx | CHANGED | rc=0 >>
3.10.0-1127.el7.x86_64
```
Проверил статус файрвола
```
otus-ansible$ ansible nginx -m systemd -a name=firewalld | grep active
        "ActiveState": "inactive",  ----------------------неактивно
        "InactiveEnterTimestampMonotonic": "0",
        "InactiveExitTimestampMonotonic": "0",
```
Запустил ad-hoc, пакет установился
```
ansible nginx -m yum -a "name=epel-release state=present" -b
nginx | SUCCESS => {
"changed": true,
```
Потом запустил плейбук, он не поменял ничего. Потом снова ad-hoc на удаление пакета. А затем плейбук и пакет снова установлен.
```
otus-ansible$ ansible nginx -m yum -a "name=epel-release state=absent" -b
nginx | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": true,
    "changes": {
        "removed": [
            "epel-release"
        ]
    },
    "msg": "",
    "rc": 0,
    "results": [
        "Loaded plugins: fastestmirror\nResolving Dependencies\n--> Running transaction check\n---> Package epel-release.noarch 0:7-11 will be erased\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package                Arch             Version        Repository         Size\n================================================================================\nRemoving:\n epel-release           noarch           7-11           @extras            24 k\n\nTransaction Summary\n================================================================================\nRemove  1 Package\n\nInstalled size: 24 k\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Erasing    : epel-release-7-11.noarch                                     1/1 \n  Verifying  : epel-release-7-11.noarch                                     1/1 \n\nRemoved:\n  epel-release.noarch 0:7-11                                                    \n\nComplete!\n"
    ]
}
nakhorenko@nakhorenko-litres:~/Linux2022-12/otus-ansible$ ansible-playbook epel.yml

PLAY [Install EPEL Repo] **************************************************************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************************************************
ok: [nginx]

TASK [Install EPEL Repo package from standard repo] ***********************************************************************************************************************
changed: [nginx]

PLAY RECAP ****************************************************************************************************************************************************************
nginx                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
<details>
    <summary>Далее работаем с плейбуком и после переделываем его в роли.</summary>

```
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true

  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install NGINX package from EPEL Repo
      yum:
        name: nginx
        state: present
      tags:
        - nginx-package
        - packages
```
    
```
TASK [NGINX | Install NGINX package from EPEL Repo] ***********************************************************************************************************************
changed: [nginx]

PLAY RECAP ****************************************************************************************************************************************************************
nginx                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Потом дорабатываю плейбук хендлерами и добавляю шаблон конфигурации nginx и запускаю плейбук

```
otus-ansible$ ansible-playbook playbooks/nginx.yml 

PLAY [NGINX | Install and configure NGINX] *******************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************
ok: [nginx]

TASK [NGINX | Install EPEL Repo package from standart repo] **************************************************************************************************
ok: [nginx]

TASK [NGINX | Install NGINX package from EPEL Repo] **********************************************************************************************************
ok: [nginx]

TASK [NGINX | Create NGINX config file from template] ********************************************************************************************************
changed: [nginx]

RUNNING HANDLER [reload nginx] *******************************************************************************************************************************
changed: [nginx]

PLAY RECAP ***************************************************************************************************************************************************
nginx                      : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

По итогу разбил все по ролям и запустил
    
```
otus-ansible$ ansible nginx -m yum -a "name=nginx state=absent" -b
nginx | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": true,
    "changes": {
        "removed": [
            "nginx"
ansible nginx -m yum -a "name=epel-release state=absent" -b
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
```
    
```
otus-ansible$ ansible-playbook playbooks/nginx.yml 

PLAY [NGINX | Install and configure NGINX] *******************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************
ok: [nginx]

TASK [epel : NGINX | Install EPEL Repo package from standart repo] *******************************************************************************************
changed: [nginx]

TASK [nginx : NGINX | Install NGINX package from EPEL Repo] **************************************************************************************************
changed: [nginx]

TASK [nginx : NGINX | Create NGINX config file from template] ************************************************************************************************
changed: [nginx]

RUNNING HANDLER [nginx : restart nginx] **********************************************************************************************************************
changed: [nginx]

RUNNING HANDLER [nginx : reload nginx] ***********************************************************************************************************************
changed: [nginx]

PLAY RECAP ***************************************************************************************************************************************************
nginx                      : ok=6    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```
</details>
