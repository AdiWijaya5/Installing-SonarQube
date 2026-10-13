# Installing-Server-SonarQube

 **Cara instal SonarQube Wit Linux/ubuntu** mengunakan manual docker images dan mengunakan asible autimation docker images

---
 
### I. instal SonarQube mengunakan manual docker images

**1**. Buat File Docker Compose Khusus SonarQube

  + **Pastikan sudah instal Docker**

```bash

mkdir -p ~/sonarqube-server && cd ~/sonarqube-server

```

**2**. Lalu buat file docker-compose.yml di dalam folder tersebut:

```bash

vim docker-compose.yml

```

- **konfigurasi lengkap** di bawah ini

```yml

version: '3.8'
services:
  sonarqube:
    image: sonarqube:lts-community
    container_name: sonarqube
    restart: unless-stopped
    ports:
      - "9000:9000"
    environment:
      - SONAR_JDBC_USERNAME=sonar
      - SONAR_JDBC_PASSWORD=sonar
      - SONAR_JDBC_URL=jdbc:postgresql://postgres:5432/sonar
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    depends_on:
      - postgres

  postgres:
    image: postgres:15-alpine
    container_name: sonarqube_db
    restart: unless-stopped
    environment:
      - POSTGRES_USER=sonar
      - POSTGRES_PASSWORD=sonar
      - POSTGRES_DB=sonar
    volumes:
      - postgresql_data:/var/lib/postgresql/data

volumes:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
  postgresql_data:

```

---

**3**. Lalu Atur Limit Sistem Linux & Nyalakan Server

- pastikan pengaturan memori virtual host Linux sudah disesuaikan agar SonarQube tidak *crash*

```bash

sudo sysctl -w vm.max_map_count=524288
sudo sysctl -w fs.file-max=131072

```

- Lalu jalankan kontainernya

```bash

docker compose up -d

```

**4**. Akses Dashboard SonarQube

1. Buka browser dan akses http://IP_SERVER:9000.
2.  Login menggunakan akun default: Username: admin, Password: admin.
3.  Anda akan diminta mengubah password lama ke password baru yang lebih aman.

---

### II. instal SonarQube mengunakan asible autimation docker images

**1**. Buat Berkas Inventory (Inventory.ini)

```ini

[sonarqube_servers]
<server_ip>


[all:vars]
ansible_user=ubuntu
ansible_port=22
ansible_ssh_private_key_file=~/.ssh/id_rsa

```

**2**. Install SonarQube Scanner mengunakan Ansible, Buat file (deploy-sonarqube.yaml)

```yaml


---
- name: Setup and Run SonarQube 
  hosts: sonarqube_servers
  become: true
  tasks:
    - name: Update apt cache
      apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Install dependencies for Docker repository
      ansible.builtin.apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
          - python3-pip
          - python3-docker
        state: present
        update_cache: true

    - name: Add Docker official GPG key
      ansible.builtin.shell: |
        install -m 0755 -d /etc/apt/keyrings
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
        chmod a+r /etc/apt/keyrings/docker.gpg
      args:
        creates: /etc/apt/keyrings/docker.gpg

    - name: Set up the stable Docker repository
      ansible.builtin.shell: |
        echo \
          "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
          $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
      args:
        creates: /etc/apt/sources.list.d/docker.list

    - name: Install Docker Engine and Containerd
      ansible.builtin.apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
        state: present
        update_cache: true

    - name: Ensure Docker service is running and enabled
      ansible.builtin.service:
        name: docker
        state: started
        enabled: true

    - name: Add remote user to docker group
      ansible.builtin.user:
        name: "{{ ansible_user }}"
        groups: docker
        append: true

    - name: Set vm.max_map_count for Elasticsearch (Minimum requirement)
      sysctl:
        name: vm.max_map_count
        value: "262144"
        state: present
        reload: true

    - name: Run Lightweight SonarQube container with Memory Limits
      ansible.builtin.shell: |
        docker run -d \
          --name sonarqube \
          --restart always \
          -p 9000:9000 \
          -m 2g \
          --memory-swap 3g \
          -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
          -e SONAR_SEARCH_JAVAADDITIONALOPTS="-Xms512m -Xmx512m -Ddiscovery.type=single-node" \
          -e SONAR_WEB_JAVAADDITIONALOPTS="-Xms256m -Xmx512m" \
          -v sonarqube_data:/opt/sonarqube/data \
          -v sonarqube_extensions:/opt/sonarqube/extensions \
          -v sonarqube_logs:/opt/sonarqube/logs \
          sonarqube:lts-community
      args:
        executable: /bin/bash

```

**3**. Jalankan Playbook Ansible

```bash

ansible-playbook -i Inventory deploy-sonarqube.yml 

```

---

### login SonarQube

---
<p align="center"><img width="1919" height="1040" alt="Screenshot 2026-10-07 024423" src="https://github.com/user-attachments/assets/1ca0d290-2d0b-4a4a-95c8-38ce4f73b334" /></p>

### Cara generate token SonarQube

* Masuk ke Pengaturan Akun (My Account)
* Buka Menu Security. pilih tab Security yang terletak di bagian atas atau samping daftar menu akun.
* Generate Token Baru
  - Pada bagian Generate Tokens, masukkan nama atau deskripsi untuk token Anda (contoh: jenkins-token).
  - Pilih tipe token (biasanya User Token).
  - Klik tombol Generate.

<p align="center"><img width="1919" height="1041" alt="image" src="https://github.com/user-attachments/assets/fe6631df-8711-4701-9425-9ed46d93ff33" /></p>

### Pengunaan CI/CD di Jenkins


```yaml

stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('SonarQubeServer') {
                        sh "${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=<project_kamu> \
                        -Dsonar.projectName=<project_kamu> \
                        -Dsonar.sources=. "
                    }
                }
            }
        }

```


### **Full Dokumentasi** https://www.sonarsource.com/products/sonarqube/









