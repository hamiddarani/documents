# Ansible

**instalation**

```bash
python3 -m venv venv
source venv/bin/activate
pip install ansible
touch inventory.ini
```

```ini
[group]
entity key=value

[all]
node15 ansible_host 192.168.12.18 ansible_user ubuntu
```

```bash
ansible all -i inventory.ini -m ping
ansible all -i inventory.ini -m raw "apt update && apt install -y python3"
```

**ansible.cfg**

```ini
[default]
host_key_cheking=False
```

```bash
ansible-playbook -i inventory.ini playbook.yml
```

**gather_fact**

```yaml
- host: all
  gather_facts: no
```

**task**

```yaml
- hosts: all
  become: true
  tasks:
    - name: update package list
      become: true 
      apt:
        update_cache: yes
```

```yaml
- hosts: all
  serial: 1
  tasks:
    - name: install mysql
      apt:
        name: mysql8
        state: present # present or absent
```

**variable**

```yaml
- hosts: all
  vars:
    docker_packeages_to_removed:
      - docker
      - docker.io
      - docker-engine
      - containerd
      - runc
  tasks:
    - name: uninstall old package
      become: true
      apt:
        name: "{{ docker_packeages_to_removed }}"
        state: absent
```

**loop**

```yaml
- hosts: all
  vars:
    docker_packeages_to_removed:
      - docker
      - docker.io
      - docker-engine
      - containerd
      - runc
  tasks:
    - name: uninstall old package
      become: true
      apt:
        name: "{{ item }}"
        state: absent
      loop: "{{ docker_packeages_to_removed }}"
      with_items: "{{ docker_packeages_to_removed }}"
```

**install docker**

```yaml
- hosts: all
  #become: yes
  vars:
    docker_package_to_remove:
      - docker
      - docker.io
      - docker-engine
      - containerd
      - runc
    
    docker_pre_required_packages_to_install:
      - gnupg
      - ca-certificates
      - curl

    docker_package_to_install:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin

  tasks:
    ################## play book end if docker exist ###################
    - name: check docker installation
      shell: "which docker"
      ignore_errors: yes
      register: result

    - name: exit if docker exists
      meta: end_host # end task on machine if docker exists
      when: result.rc == 0
    ###################################################################

    - name: update package cache
      apt:
        update_cache: yes

    - name: upgrade system
      apt:
        upgrade: full
        
    ################## show all ansible facts ###################
    - name: show facts
      debug:
        msg: "{{ ansible_facts['architecture'] }}" 

    - name: uninstall old docker package
      become: yes
      apt:
        name: "{{ docker_package_to_remove }}"
        state: absent

    - name: docker install pre-required packages
      become: yes
      loop: "{{ docker_pre_required_packages_to_install }}"
      apt: 
        name: "{{ item }}"
        state: present

    - name: install docker apt repository key
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg

    - name: add docker apt repository
      apt_repository: deb [arch="{{ "amd64" if ansible_facts['architecture'] == "x86_64" else "i386" }}" ] https://download.docker.com/linux/ubuntu "{{ ansible_facts['lsb']['codename'] }}" stable
      state: present

    - name: install docker
      become: yes
      apt: 
        name: "{{ docker_package_to_install }}"
        state: present
        update_cache: yes

      
```

**handlers**

```yaml
- name: copy nginx config
  copy:
    src: nginx.conf
    dest: /etc/nginx/nginx.conf
- name: restart systemd
  systemd:
    state: restarted
```

```yaml
tasks:
  - name: copy nginx config
  copy:
    src: nginx.conf
    dest: /etc/nginx/nginx.conf
  notify:
    - restart nginx
  tags:
    - systemd
```

```
handlers:
  - name: restart nginx
    systemd:
      state: restarted
```

