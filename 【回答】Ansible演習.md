
# Ansible-2
- name: Install the latest version of Apache
  hosts: 172.31.16.124
  become: yes
  tasks:
   - name: Install Apache
     ansible.builtin.dnf:
       name: httpd
       state: latest


## Ansible-3
- name: Install the latest version of Apache
  hosts: 172.31.16.124
  become: yes
  tasks:
   - name: stop Apache
     ansible.builtin.systemd_service:
       name: httpd
       state: stopped


## Ansible-4
- name: コピー
  hosts: 172.31.16.124
  become: yes
  tasks:
   - name: index.htmlをコピー
     ansible.builtin.copy:
       src: /var/tmp/index.html
       dest: /var/www/html


## Ansible-5
- name: ユーザ作成
  hosts: 172.31.16.124
  become: yes
  tasks:
   - name: yoshieユーザを作成
     ansible.builtin.user:
       name: yoshie
       create_home: yes


## Ansible-6
- name: 文字列置換
  hosts: 172.31.16.124
  become: yes
  tasks:
   - name: /var/www/html/index.htmlに文字列置換
     ansible.builtin.lineinfile:
       path: /var/www/html/index.html
       regexp: '^yoshie'
       line: yoshie333333


## Ansible-7
- name: シェルの実行
  hosts: 172.31.16.124
  become: yes
  tasks:
   - name: yoshie.shの実行
     ansible.builtin.script: ./yoshie.sh

       