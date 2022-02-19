# ansible_basics
# ansible tutorial

1. erstmal ein neues Github repo erstellen: *ansible_basics*
2. dann auf dem Ansible Host das repo clonen: **git clone git@github.com:janoking/ansible_basics.git**
3.  inventory erstellen
4.  nun muss Ansible noch den Key mitbekommen und ein Ping: **ansible all --key-file ~/.ssh/id_rsa -i inventory -m ping**
m= module ping

nun kann eine *ansible.cfg* file erstellt werden, die liest ansible jedes mal ein solange die cfg in unserem Ordner liegt. Die cfg Datei in /etc/ansible wird damit ignoriert.

Alle möglcihen Infos wie CPU Version, etc..
**ansible all -m gather_facts**
für nur einen hosts:
**ansible all -m gather_facts --limit 136.172.10.81**
nützlich zum debugging.

![Screenshot_2022-02-18-08-16-15_3840x2160.png](:/50540d1860f24b5abc7e2ab1290139e1)

# adhoc commands

adhoc apt cache update:
**ansible all -m apt -a update_cache=true --become --ask-become-pass**
-m = modul : apt
-a = argument für das modul
update_cache=true = cache für apt updaten
--become --ask-become-pass = sudo rechte abfragen

mehr Infos gibt es hier: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html
![Screenshot_2022-02-18-08-23-09_3840x2160.png](:/c4065881a272427a8aac7463c31cae21)

install a package to all servers:
**ansible all -m apt -a name=vim-nox --become --ask-become-pass**
package upgrade:
**ansible all -m apt -a "upgrade=dist" 

# Playbooks

es wird ein Playbook erstellt:
*install_apache.yml
```
---

- hosts: all
  #  become: true
  tasks:

  - name: install apache2 package
    apt:
      name: apache2
```

gestartet wird dies dann mit **ansible-playbook install_apache.yml**

![Screenshot_2022-02-18-09-33-47_3840x2160.png](:/ea7fbb219316408eb283679d27ecd1ee)


mit state **absent** können die Pakete wieder gelöscht werden
```
---

- hosts: all
  #  become: true
  tasks:

  - name: install apache2 package
    apt:
      name: apache2
			 state: absent
```

# 'when' Condition

mit when können wir auf unterschiedliche Distrobutionen eingehen: 	  **when: ansible_distribution == 'Debian'**
```
---

- hosts: all
  #  become: true
  tasks:

  - name: update repository index
    apt:
      update_cache: yes
	  when: ansible_distribution == 'Debian'

  - name: install apache2 package
    apt:
      name: apache2
      state: latest
	  when: ansible_distribution == 'Debian'
	  
  - name: add php support for apache
    apt:
      name: libapache2-mod-php
      state: latest
	  when: ansible_distribution == 'Debian'	  
```

# improve playbooks

man kann viel mit listen Arbeiten, z.B.: brauch man nicht 2 Sektionen mit apache und dem php mod! Einfach als Liste Schreiben, apt update kann auch noch mit rein
```
 ---
 
 - hosts: all
   become: true
   tasks:
 
   - name: install apache2 package
     apt:
       name:
         - apache2
         - libapache2-mod-php
       state: latest
       update_cache: yes
     when: ansible_distribution == "Ubuntu"
 
   - name: install httpd package
     dnf:
       name:
         - httpd
         - php
       state: latest
       update_cache: yes
     when: ansible_distribution == "CentOS"
```

statt apt oder dnf, kann auch einfach **package:** benutzt werden, wenn dann im Inventory noch die Variablen stehen, kann ansible automatisch herausfinden welchen Packagemanager benutzt wird:
```
136.172.10.81 apache_package=apache2 php_package=libapache2-mod-php
136.172.10.82 apache_package=apache2 php_package=libapache2-mod-php
136.172.10.83 apache_package=apache2 php_package=libapache2-mod-php
136.172.10.84 apache_package=apache2 php_package=libapache2-mod-php
172.16.250.248 apache_package=httpd php_package=php
```

# Targeting Specific Nodes

Das inventory wird nun in Verschiedenen Servergruppen geteilt:

```
[web_servers]
136.172.10.81 			
136.172.10.82 			

[db_servers]
136.172.10.83 			

[file_servers]
136.172.10.84 			
```
und es wird ein neues Playbook mit den Namen `site.yml`, site = weil das ganze inventory einmal bearbeitet wird, der name kann aber so heißen wie man will.
hier werden diesmal die server unterschieden zwischen, centos ubuntu, db server, webserver und file server. Fileserver benötigen z.B. den Samba Server, und die Webserver natürlich apache.
bei `- hosts: all` werden jetzt zwar immer noch alle Server angesprochen, aber nur für ein *UPDATE*, danach wird mit *- hosts: web_servers* andere Gruppen angesprochen.

```
 ---
 
 - hosts: all
   become: true
   pre_tasks:
 
   - name: install updates (CentOS)
     dnf:
       update_only: yes
       update_cache: yes
     when: ansible_distribution == "CentOS"
 
   - name: install updates (Ubuntu)
     apt:
       upgrade: dist
       update_cache: yes
     when: ansible_distribution == "Ubuntu"
 
 
 - hosts: web_servers
   become: true
   tasks:
 
   - name: install httpd package (CentOS)
     dnf:
       name:
         - httpd
         - php
       state: latest
     when: ansible_distribution == "CentOS"
 
   - name: install apache2 package (Ubuntu)
     apt:
       name:
         - apache2
         - libapache2-mod-php
       state: latest
     when: ansible_distribution == "Ubuntu"
 
 - hosts: db_servers
   become: true
   tasks:
 
   - name: install mariadb server package (CentOS)
     dnf:
       name: mariadb
       state: latest
     when: ansible_distribution == "CentOS"
 
   - name: install mariadb server
     apt:
       name: mariadb-server
       state: latest
     when: ansible_distribution == "Ubuntu"
 
 - hosts: file_servers
   become: true
   tasks:
 
   - name: install samba package
     package:
       name: samba
       state: latest
```

# tags

tags sind für eine bessere Übersicjt gedacht, auch können die tags eingesehen werden mit **ansible-playbook --list-tags site_tags.yml**

![Screenshot_2022-02-19-10-09-36_3840x2160.png](:/7fbdcbf0b70c40dda3f7a1f6f98db95c)

mit **ansible-playbook --tags centos** wird dann nur die TASKS ausgeführt, die den tag centos haben.
für mehrere tags werden "" benötigt.
**ansible-playbook --tags "apache,db"**

# managing files

copy a file to a server.
**mkdir files && cd files**

dann wird eine default site Datei erstellt
**vi default_site.html**
```
 <html>
     <title>Web-site test</title>
     <body>
        Ansible is awesome!
    </body>
 </html>
```

dann kann in der site.yml angegeben werden, welche Datei, wo die hinkopiert werden muss und die Berechtigung wird auch benötigt.

```
  - name: install apache2 package (Ubuntu)
    apt:
      name:
        - apache2
        - libapache2-mod-php
      state: latest
    when: ansible_distribution == "Ubuntu"

  - name: copy default html file for site
    tags: apache,apache2,httpd
    copy:
      src: default_site.html
      dest: /var/www/html/index.html
      owner: root
      group: root
      mode: 0644
```
name, copy ,src, dest, owner, group, mode
> owner: root - ist hier OK, weil es nur eine statische testfile ist. sonst muss das der  www-data User sein.

# install unzip, download .zip and extract

mit dem Modul unarchive, kann eine ZIP Datei heruntergeladen werden und entpackt und installiert werden, hier z.B.: terraform. 

```
  - name: install terraform
      unarchive:
        src: https://releases.hashicorp.com/terraform/0.12.28/terraform_0.12.28_linux_amd64.zip
        dest: /usr/local/bin
        remote_src: yes
        mode: 0755
        owner: root
        group: root
```
**remote_src:** Ansible sagen das das es sich hier um eine Remotequelöle handelt, die erst heruntergeladen werden soll, das Ziel (dest: ) wird auch angegeben.

## systemctl start & enabled
bei RHEL müssen neu installierte services erst mit systemctl start/enabled werden, sonst laufen die nicht:
```
  - name: start httpd (CentOS)
    tags: apache,centos,httpd
    service:
      name: httpd
      state: started
      enabled: yes
    when: ansible_distribution == "CentOS"
```
## eine config file ändern mit regex und dann apache2 restart restarten:
```
  - name: change hostnamelookup
    lineinfile:
      path: /etc/apache2/apache2.conf
      regexp: '^HostnameLookups'
      line: HostnameLookups on
    when: ansible_distribution == "Debian"
    register: apache2  #register ist hier wichtig da es bei when: apache2.changed wieder aufgerufen wird

  - name: restart apache2 (Debian)
    service:
      name: apache2
      state: restarted
    when: apache2.changed    # siehe register
```

https://www.youtube.com/watch?v=soeBHGAMkoQ

![Screenshot_2022-02-19-11-28-36_3840x2160.png](:/581de117b93941ca895fcf380eef50f7)

> wenn in lineinfile ein typo ist, dann schreibt nsible immer wieder den neuen String, Daher sollte man das Playbook als test gleich ein weiteres mal starten um zu sehen ob etwas gechanged wurde, wenn alles skipped und grün ist, dann ist auch alles OK

wenn mehrere Strings geändert werden muss, dann muss auch ein neuer Register angegeben werden, sonst wird der service nicht neugestartet (kein trigger).

# create a user

auf alle hosts den user molotov erstellen

```
- hosts: all
  become: true
  tasks:

  - name: create molotov user
    tags: always
    user:
      name: molotov
      group: root
```

SSH Public Key kopieren, damit ist eine passwortfreier Login möglich, auch wird der user molotov in den sudoers file eingetragen, dafür muss die Datei  /files/sudoer_molotov im ansible master erstellt werden (src: ) diese wird dann auf den remote server (dest: ) nach /etc/sudoers.d/molotov kopiert:

```
  - name: add ssh key molotov
    tags: always
    authorized_key:
      user: molotov
      key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC6SfokAoJ3pGZPOF26IKQ6uDR5p9sQzJTlFMrgd8yB8//wUJXXxLKRCcjNjtJWoROYCZH4ZfSItGclTWbZhZYI4bzVPJRWKODocGXGdWntsPL178rC1Sfp6qiJUaoYfkKfCc192RpF1fBZ4hFKtDvXiZy3zwRYhF334OB87pS5hII1IrPAG0+X6b53jdkOTk6a/EemmHJilqX9irHYQm/ND1T53ntpE6ciDWZOImxxo+9dzn1tZGBq6b3n2CA0KBTlc94Zf1d6YN81ZXCIzhZ0JAdHJbdjniZMzUAwf62rtNVSSJHoc8UoMkQWmhLLyZpHZVmN2pAqT+jeovnXEm5PwAPU7VOZCU6W2z2ZCLlN2dYcuEqGjbcdMjj+ii16Rzd+7wo2Bq1jYcqb+ilFrQ0+SOV8C/hiRoJDj4QaNqFZET2EJxs+85GsWUuYUatz7Y9ZJFsDktAq2BozwKARYMbU32tc5Va2R+OMFhm0sM8bq4EfvJTs8grFUInW+047cuKF237ws7j+Ppcyon97AgGMssLo4k39rFZuBq1BiR0r3p2B4THQMuPVzEKOFUBL41hv3CDt+OcD/ROnOV2uPj8cxAyao+jcBDCRcqVbXgs+Vfxxv4VlRoHGYhN5nBDLEK47NYLAFmsPNlU2Iu+sGvjdnwOtS4mmteK8WAUWWQaGpw== root@ansible"

  - name: add sudoers file for molotov
    tags: always
    copy:
      src: sudoer_molotov
      dest: /etc/sudoers.d/molotov
      owner: root
      group: root
      mode: 0440
```
die Datei in `~/.ansible/role/ansible_basics/files/sudoer_molotov` sieht so aus:

`molotov ALL=(ALL) NOPASSWD: ALL`

ein cp wird noch benötigt, da der private key noch im root home liegt:
`cp ~/.ssh/ansible /home/molotov/.ssh`
dann noch ein chown, damit der user molotov damit auch arbeiten kann:
`chown molotov:molotov /home/molotov/.ssh/ansible`

dann als `su molotov` mit dem private key auf den Server Verbinden ohne password:
`ssh -i ~/.ssh/ansible molotov@136.172.10.81`

## become user bestimmen
damit der become user nicht nötig ist, kann in der `ansible.cfg` der Key und der remote user eingetragen werden:
```
[defaults]
inventory = inventory
private_key_file = ~/.ssh/ansible
remote_user = molotov
```

Damit das henne und Ei Problem nicht existiertm wird ein Bootstrap Playbook angelegt
`cp site.yml bootstrap.yml`
in bootstrap wird alles gelöscht ausser:

```
---

- hosts: all
  become: true
  pre_tasks:

  - name: install updates (CentOS)
    dnf:
      update_only: yes
      update_cache: yes
    when: ansible_distribution == "CentOS"

  - name: install updates (Ubuntu)
    apt:
      upgrade: dist
      update_cache: yes
    when: ansible_distribution == "Ubuntu"


- hosts: all
  become: true
  tasks:


  - name: add ssh key molotov to authorized key file
    tags: always
    authorized_key:
      user: molotov
      key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC6SfokAoJ3pGZPOF26IKQ6uDR5p9sQzJTlFMrgd8yB8//wUJXXxLKRCcjNjtJWoROYCZH4ZfSItGclTWbZhZYI4bzVPJRWKODocGXGdWntsPL178rC1Sfp6qiJUaoYfkKfCc192RpF1fBZ4hFKtDvXiZy3zwRYhF334OB87pS5hII1IrPAG0+X6b53jdkOTk6a/EemmHJilqX9irHYQm/ND1T53ntpE6ciDWZOImxxo+9dzn1tZGBq6b3n2CA0KBTlc94Zf1d6YN81ZXCIzhZ0JAdHJbdjniZMzUAwf62rtNVSSJHoc8UoMkQWmhLLyZpHZVmN2pAqT+jeovnXEm5PwAPU7VOZCU6W2z2ZCLlN2dYcuEqGjbcdMjj+ii16Rzd+7wo2Bq1jYcqb+ilFrQ0+SOV8C/hiRoJDj4QaNqFZET2EJxs+85GsWUuYUatz7Y9ZJFsDktAq2BozwKARYMbU32tc5Va2R+OMFhm0sM8bq4EfvJTs8grFUInW+047cuKF237ws7j+Ppcyon97AgGMssLo4k39rFZuBq1BiR0r3p2B4THQMuPVzEKOFUBL41hv3CDt+OcD/ROnOV2uPj8cxAyao+jcBDCRcqVbXgs+Vfxxv4VlRoHGYhN5nBDLEK47NYLAFmsPNlU2Iu+sGvjdnwOtS4mmteK8WAUWWQaGpw== root@ansible"

  - name: add sudoers file for molotov
    tags: always
    copy:
      src: sudoer_molotov
      dest: /etc/sudoers.d/molotov
      owner: root
      group: root
      mode: 0440
```
> damit der chache update vom Paketmanager nicht zu nervigen changed: 1 in der Info führt, kann dieses modul verwendet werden:
>       update_cache: yes
         changed_when: false


