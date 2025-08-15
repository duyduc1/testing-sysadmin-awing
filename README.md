# Hướng dẫn chạy Playbook cài đặt Nginx

## Yêu cầu
- Ansible đã được cài đặt (`sudo apt install ansible -y`).
- SSH key đã được sinh và copy (`ssh-copy-id dev@192.168.230.145`).
- Server có quyền sudo.

## Các bước thực hiện
1. Tạo thư mục dự án:
``` bash
mkdir ansible-nginx && cd ansible-nginx
mkdir -p group_vars host_vars roles/nginx/{tasks,handlers,templates,files,vars}
```

2. Cập nhật `inventory.ini` với IP server.
3. Chạy:
``` bash
ansible-playbook --syntax-check -i inventory.ini nginx-setup.yml
ansible-playbook -i inventory.ini nginx-setup.yml -v
ansible-playbook -i inventory.ini nginx-setup.yml --ask-become-pass
```

4. Kiểm tra:
``` bash
systemctl status nginx
sudo ufw status
curl http://{{ ansible_host }}
cat /etc/logrotate.d/nginx-custom
cat /var/log/ansible-deployment.log
```

## 📂 Cấu trúc thư mục dự án

```bash
project-request-2/
├── templates/            
│   ├── index.html.j2       
│   └── nginx-lograte.j2           
│
├── inventory.ini          
├── nginx-setup.yml       
├── vars.yml             
├── README.md          
```
