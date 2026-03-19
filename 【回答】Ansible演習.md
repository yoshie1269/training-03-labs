
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


## Ansible-8
- name: tomcat構築
  hosts: 172.31.18.124
  become: yes
  tasks:
   - name: 1. Amazon Corretto（JDK）をダウンロード
     ansible.builtin.get_url: 
       url: https://corretto.aws/downloads/latest/amazon-corretto-8-x64-linux-jdk.tar.gz
       dest: /home/ec2-user/amazon-corretto-8-x64-linux-jdk.tar.gz

   - name: 2. JDK 圧縮ファイルを展開する
     ansible.builtin.unarchive: 
       src: /home/ec2-user/amazon-corretto-8-x64-linux-jdk.tar.gz
       dest: /home/ec2-user/
       remote_src: yes

   - name: 3. JDK ディレクトリを /opt に移動する
     ansible.builtin.command: mv /home/ec2-user/amazon-corretto-8.482.08.1-linux-x64 /opt
     args:
        creates: /opt/amazon-corretto-8.482.08.1-linux-x64

   - name: 4. bash 設定ファイルをバックアップする
     ansible.builtin.copy:
       src: /root/.bash_profile
       dest: /root/.bash_profile.bak
       remote_src: yes

   - name: 5. vi /root/.bash_profile PATH に JDK を追加する
     ansible.builtin.lineinfile:
       path: /root/.bash_profile
       line: 'export PATH=$PATH:$HOME/bin:/opt/amazon-corretto-8.482.08.1-linux-x64/bin'
       state: present

   - name: 6. java確認（環境変数付き）
     ansible.builtin.command: java -version
     environment:
       PATH: "/opt/amazon-corretto-8.482.08.1-linux-x64/bin:{{ ansible_env.PATH }}"

   - name: 7. tomcat ユーザ作成
     ansible.builtin.user:
       name: tomcat
       shell: /sbin/nologin
       state: present

   - name: 8. Tomcat 本体をダウンロード
     ansible.builtin.get_url: 
       url: https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.115/bin/apache-tomcat-9.0.115.tar.gz
       dest: /home/ec2-user/apache-tomcat-9.0.115.tar.gz
       
   - name: 9. Tomcat 圧縮ファイルを展開する
     ansible.builtin.unarchive: 
       src: /home/ec2-user/apache-tomcat-9.0.115.tar.gz
       dest: /home/ec2-user/
       remote_src: yes

   - name: 10. Tomcat を /usr/local に配置する
     ansible.builtin.command: mv /home/ec2-user/apache-tomcat-9.0.115 /usr/local/
     args:
      creates: /usr/local/apache-tomcat-9.0.115

   - name: 11. Tomcatの所有者変更
     ansible.builtin.file:
      path: /usr/local/apache-tomcat-9.0.115
      owner: tomcat
      group: tomcat
      recurse: yes

   - name: 12. Tomcatのシンボリックリンク作成
     ansible.builtin.file:
      src: /usr/local/apache-tomcat-9.0.115
      dest: /usr/local/tomcat
      state: link

   - name: 13. vsetenv.sh を作成する
     ansible.builtin.copy:
       dest: /usr/local/tomcat/bin/setenv.sh
       content: |
        #!/bin/sh
        export CATALINA_HOME=/usr/local/tomcat
        export JAVA_HOME=/opt/amazon-corretto-8.482.08.1-linux-x64
        export JAVA_OPTS="-Xms128m -Xmx512m"
       owner: tomcat
       group: tomcat
       mode: '0755'

   - name: 14. server.xml をバックアップ
     ansible.builtin.copy:
       src: /usr/local/tomcat/conf/server.xml
       dest: /usr/local/tomcat/conf/server.xml.bak
       remote_src: yes

   - name: 15. 自動デプロイ設定を変更する
     ansible.builtin.lineinfile:
       path: /usr/local/tomcat/conf/server.xml
       regexp: 'unpackWARs=.*autoDeploy=.*'
       line: '      unpackWARs="false" autoDeploy="false">'

   - name: 16. Tomcat サービスを登録する
     ansible.builtin.copy:
       dest: /etc/systemd/system/tomcat.service
       content: |
        [Unit]
        Description=Apache Tomcat 9
        After=network.target

        [Service]
        Type=forking

        User=tomcat
        Group=tomcat

        Environment=JAVA_HOME=/opt/amazon-corretto-8.482.08.1-linux-x64
        Environment=CATALINA_HOME=/usr/local/tomcat

        ExecStart=/usr/local/tomcat/bin/startup.sh
        ExecStop=/usr/local/tomcat/bin/shutdown.sh

        Restart=always

        [Install]
        WantedBy=multi-user.target
       mode: '0644'

   - name: 17. systemd を再読み込み
     ansible.builtin.command: systemctl daemon-reload

   - name: 18. Tomcat起動 & 自動起動設定
     ansible.builtin.systemd:
       name: tomcat
       enabled: yes
       state: started


## Ansible-9
- name: Apache構築(tomcatと連携)
  hosts: 172.31.18.124
  become: yes
  tasks:

   - name: 1. Apache インストール
     ansible.builtin.dnf:
       name: httpd
       state: present

   - name: 2. Apache にリバースプロキシ設定を追加
     ansible.builtin.copy:
       dest: /etc/httpd/conf.d/tomcat.conf
       content: |
          ProxyRequests Off
          ProxyPass / http://127.0.0.1:8080/
          ProxyPassReverse / http://127.0.0.1:8080/
       owner: root
       group: root
       mode: '0644'

   - name: 3. Apache 設定テスト
     ansible.builtin.command: apachectl configtest
     changed_when: false

   - name: 4. Apache 起動 & 自動起動設定
     ansible.builtin.systemd:
       name: httpd
       enabled: yes
       state: restarted

