## Install ansible
``` bash
sudo apt update
sudo apt install ansible -y
ansible --version
```

### Create Folder Project
``` bash
mkdir ansible-nginx
cd ansible-nginx
mkdir -p group_vars host_vars roles/nginx/{tasks,handlers,templates,files,vars}
```

### Generate Key 
``` bash
ssh-keygen -t rsa -b 4096
ssh-copy-id dev@192.168.230.145
```

## Create file inventory.ini
``` bash
nano inventory.ini
```

### paste in file
``` bash
[webserver]
192.168.230.145 ansible_user=dev ansible_ssh_private_key_file=~/.ssh/id_rsa
```

### test connection
``` bash
ansible -i inventory.ini webserver -m ping
```

### Create Vars file
``` bash
nano vars.yml
```

### content in vars file
``` bash
nginx_package: nginx
nginx_service: nginx
firewalld_service: firewalld
nginx_web_root: /var/www/html/index.html
log_dir: /var/log/nginx
log_rotation_days: 7

server_info:
  ip: "{{ ansible_default_ipv4.address }}"
  hostname: "{{ ansible_hostname }}"
  deploy_time: "{{ ansible_date_time.date }} {{ ansible_date_time.time }}"
```

### create nginx-setup.yml
``` bash
nano nginx-setup.yml
```

``` bash
- name: Install and Configure Nginx Web Server
  hosts: webserver
  become: yes
  vars_files:
    - vars.yml

  tasks:
    # Task 1: Install and configure Nginx remotely
    - name: Install Nginx package
      package:
        name: "{{ nginx_package }}"
        state: present
      notify: restart nginx

    # # Task 2: Make sure firewalld is running and port 80 is open
    - name: Allow HTTP traffic through UFW
      ufw:
        rule: allow
        port: '80'
        proto: tcp
      when: ansible_os_family == "Debian"

    #Task 3: Make sure Nginx is always started with the system
    - name: Ensure Nginx is running and enabled
      systemd:
        name: "{{ nginx_service }}"
        state: started
        enabled: yes

    # Task 4: Deploy a custom index.html file
    - name: Deploy custom index.html
      template:
        src: index.html.j2
        dest: "{{ nginx_web_root }}"
        owner: www-data
        group: www-data
        mode: '0644'
      notify: restart nginx

    # Task 5: Configure log rotation for Nginx logs
    - name: Configure log rotation for Nginx
      template:
        src: nginx_logrotate.j2
        dest: /etc/logrotate.d/nginx-custom
        owner: root
                group: root
        mode: '0644'

    # Task 6: Tạo script để ghi log quá trình chạy Playbook
    - name: Create deployment log
      lineinfile:
        path: /var/log/ansible-deployment.log
        line: "Nginx deployed on {{ ansible_date_time.date }} {{ ansible_date_time.time }} by Ansible"
        create: yes
        owner: root
        group: root
        mode: '0644'

  handlers:
    - name: restart nginx
      systemd:
        name: "{{ nginx_service }}"
        state: restarted

    - name: reload firewalld
      systemd:
        name: firewalld
        state: reloaded
```

### Create template anh setup index.html and nginx-logrotate
``` bash
mkdir templates
cd templates/
```

``` bash
nano index.html.j2
```

- content in index.html.j2

``` bash
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Nginx Server Information</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 50px auto;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 20px;
        }
        .container {
            background: rgba(255,255,255,0.1);
            padding: 30px;
            border-radius: 10px;
            backdrop-filter: blur(10px);
        }
        .info-box {
            background: rgba(255,255,255,0.2);
            padding: 15px;
            margin: 10px 0;
            border-radius: 5px;
        }
        h1 { color: #fff; text-align: center; }
        .highlight { color: #ffeb3b; font-weight: bold; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🚀 Nginx Server Successfully Deployed!</h1>
        
        <div class="info-box">
            <h3>📍 Server Information:</h3>
            <p><strong>Server IP:</strong> <span class="highlight">{{ server_info.ip }}</span></p>
            <p><strong>Hostname:</strong> <span class="highlight">{{ server_info.hostname }}</span></p>
            <p><strong>Deploy Time:</strong> <span class="highlight">{{ server_info.deploy_time }}</span></p>
        </div>

        <div class="info-box">
            <h3>⚙️ System Details:</h3>
            <p><strong>OS:</strong> {{ ansible_distribution }} {{ ansible_distribution_version }}</p>
            <p><strong>Architecture:</strong> {{ ansible_architecture }}</p>
            <p><strong>Kernel:</strong> {{ ansible_kernel }}</p>
            <p><strong>CPU Cores:</strong> {{ ansible_processor_cores }}</p>
            <p><strong>Memory:</strong> {{ (ansible_memtotal_mb / 1024) | round(2) }} GB</p>
        </div>

        <div class="info-box">
            <h3>🌐 Nginx Status:</h3>
            <p><strong>Web Server:</strong> Nginx</p>
            <p><strong>Status:</strong> <span class="highlight">Running</span></p>
            <p><strong>Port:</strong> 80 (HTTP)</p>
        </div>

        <div class="info-box">
            <h3>🔧 Deployed by:</h3>
            <p><strong>Automation Tool:</strong> Ansible</p>
            <p><strong>Playbook:</strong> nginx-setup.yml</p>
            <p><strong>Last Updated:</strong> {{ ansible_date_time.iso8601 }}</p>
        </div>
    </div>
</body>
</html>
```

``` bash
nano nginx-logrotate.j2
```

- content in nginx-logrotate.j2

``` bash
{{ log_dir }}/*.log {
    daily
    missingok
    rotate {{ log_rotation_days }}
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    prerotate
        if [ -d /etc/logrotate.d/httpd-prerotate ]; then \
            run-parts /etc/logrotate.d/httpd-prerotate; \
        fi \
    endscript
    postrotate
        invoke-rc.d nginx rotate >/dev/null 2>&1
    endscript
}
```

## Run with ansinble-playbook
``` bash
ansible-playbook --syntax-check -i inventory.ini nginx-setup.yml # checking syntax
ansible-playbook -i inventory.ini nginx-setup.yml -v # run playbook
ansible-playbook -i inventory.ini nginx-setup.yml --ask-become-pass # run playbook with sudo 
```

### Checking results
```bash
systemctl status nginx
sudo ufw status
curl http://192.168.240.145 # or run with http://192.168.240.145:80
cat /etc/logrotate.d/nginx-custom # check log rotation
cat /var/log/ansible-deployment.log # checklog inprogress Playbook 
```

### if logrotate is bug we can fix it 
``` bash 
cd ansible-nginx
cd templates/
mv nginx* temp_file.j2
mv temp_file.j2 nginx_logrotate.j2
ansible-playbook -i inventory.ini nginx-setup.yml --ask-become-pass
```