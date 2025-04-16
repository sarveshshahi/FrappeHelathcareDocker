# 🚀 ERPNext Docker Setup Guide

Welcome to the **ERPNext Docker Setup Guide**! This guide will help you quickly set up ERPNext using Docker. 🐳

---

## 📌 Prerequisites
Ensure you have the following installed:
- Docker 🐳
- Docker Compose
- Git

## 📂 Step 1: Create a Directory for ERPNext
```sh
mkdir erpnext_docker && cd erpnext_docker
```
(For Windows, create the `erpnext_docker` folder manually and navigate to it in `cmd`.)

---

## 📝 Step 2: Create the Docker Compose File
Create a file named **docker-compose.yml** and add the following:

```yaml
version: "3"

services:
  backend:
    image: frappe/erpnext:v15.19.1
    deploy:
      restart_policy:
        condition: on-failure
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs

  configurator:
    image: frappe/erpnext:v15.19.1
    deploy:
      restart_policy:
        condition: none
    entrypoint:
      - bash
      - -c
    # add redis_socketio for backward compatibility
    command:
      - >
        ls -1 apps > sites/apps.txt;
        bench set-config -g db_host $$DB_HOST;
        bench set-config -gp db_port $$DB_PORT;
        bench set-config -g redis_cache "redis://$$REDIS_CACHE";
        bench set-config -g redis_queue "redis://$$REDIS_QUEUE";
        bench set-config -g redis_socketio "redis://$$REDIS_QUEUE";
        bench set-config -gp socketio_port $$SOCKETIO_PORT;
    environment:
      DB_HOST: db
      DB_PORT: "3306"
      REDIS_CACHE: redis-cache:6379
      REDIS_QUEUE: redis-queue:6379
      SOCKETIO_PORT: "9000"
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs

  create-site:
    image: frappe/erpnext:v15.19.1
    deploy:
      restart_policy:
        condition: none
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    entrypoint:
      - bash
      - -c
    command:
      - >
        wait-for-it -t 120 db:3306;
        wait-for-it -t 120 redis-cache:6379;
        wait-for-it -t 120 redis-queue:6379;
        export start=`date +%s`;
        until [[ -n `grep -hs ^ sites/common_site_config.json | jq -r ".db_host // empty"` ]] && \
          [[ -n `grep -hs ^ sites/common_site_config.json | jq -r ".redis_cache // empty"` ]] && \
          [[ -n `grep -hs ^ sites/common_site_config.json | jq -r ".redis_queue // empty"` ]];
        do
          echo "Waiting for sites/common_site_config.json to be created";
          sleep 5;
          if (( `date +%s`-start > 120 )); then
            echo "could not find sites/common_site_config.json with required keys";
            exit 1
          fi
        done;
        echo "sites/common_site_config.json found";
        bench new-site --no-mariadb-socket --admin-password=admin --db-root-password=admin --install-app erpnext --set-default sample.site;

  db:
    image: mariadb:10.6
    healthcheck:
      test: mysqladmin ping -h localhost --password=admin
      interval: 1s
      retries: 15
    deploy:
      restart_policy:
        condition: on-failure
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --skip-character-set-client-handshake
      - --skip-innodb-read-only-compressed # Temporary fix for MariaDB 10.6
    environment:
      MYSQL_ROOT_PASSWORD: admin
    volumes:
      - db-data:/var/lib/mysql

  frontend:
    image: frappe/erpnext:v15.19.1
    depends_on:
      - websocket
    deploy:
      restart_policy:
        condition: on-failure
    command:
      - nginx-entrypoint.sh
    environment:
      BACKEND: backend:8000
      FRAPPE_SITE_NAME_HEADER: sample.site
      SOCKETIO: websocket:9000
      UPSTREAM_REAL_IP_ADDRESS: 127.0.0.1
      UPSTREAM_REAL_IP_HEADER: X-Forwarded-For
      UPSTREAM_REAL_IP_RECURSIVE: "off"
      PROXY_READ_TIMEOUT: 120
      CLIENT_MAX_BODY_SIZE: 50m
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    ports:
      - "8080:8080"

  queue-long:
    image: frappe/erpnext:v15.19.1
    deploy:
      restart_policy:
        condition: on-failure
    command:
      - bench
      - worker
      - --queue
      - long,default,short
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs

  queue-short:
    image: frappe/erpnext:v15.19.1
    deploy:
      restart_policy:
        condition: on-failure
    command:
      - bench
      - worker
      - --queue
      - short,default
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs

  redis-queue:
    image: redis:6.2-alpine
    deploy:
      restart_policy:
        condition: on-failure
    volumes:
      - redis-queue-data:/data

  redis-cache:
    image: redis:6.2-alpine
    deploy:
      restart_policy:
        condition: on-failure
    volumes:
      - redis-cache-data:/data

  scheduler:
    image: frappe/erpnext:v15.19.1
    deploy:
      restart_policy:
        condition: on-failure
    command:
      - bench
      - schedule
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs

  websocket:
    image: frappe/erpnext:v15.19.1
    deploy:
      restart_policy:
        condition: on-failure
    command:
      - node
      - /home/frappe/frappe-bench/apps/frappe/socketio.js
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs

volumes:
  db-data:
  redis-queue-data:
  redis-cache-data:
  sites:
  logs:
```
Save and exit the file.

---

## ▶️ Step 3: Start ERPNext Containers
Run the following command:
```sh
docker compose -p erpnext_project up -d
```

---

## 🌐 Step 4: Access ERPNext
Open your browser and visit:
```
http://localhost:8080
```
Use the following login credentials:
- **Username**: Administrator
- **Password**: admin

---

## 🏥 Step 5: Install the Healthcare App
Find the backend container ID:
```sh
docker ps
```
Access the backend container:
```sh
docker exec -it <backend-container-id> bash
```
Install the Healthcare app:
```sh
bench get-app --branch version-15 healthcare
bench --site sample.site install-app healthcare
```
If you encounter errors, try:
```sh
bench get-app healthcare https://github.com/frappe/healthcare
bench --site sample.site install-app healthcare
bench restart
```

---

## 🔄 Step 6: Restart ERPNext Containers
```sh
docker compose restart
```
Reload `http://localhost:8080` in your browser to see the Healthcare module installed. 🎉

---

## 📸 Screenshots & Visuals
- ✅ Creating the ERPNext Directory
- ✅ Docker Compose File Setup
- ✅ Starting ERPNext Containers
- ✅ ERPNext Login Page

---

## 🎯 Conclusion
You have successfully set up ERPNext using Docker! 🎊 Now, explore the powerful features of ERPNext and customize it to fit your business needs. 🚀

---

## 🌟 Support & Contribution
Feel free to contribute or report issues! ✨

📌 **GitHub Repo**: [Your Repository Link]  
💬 **Community Support**: [ERPNext Forum](https://discuss.erpnext.com/)  
🐛 **Report Issues**: Open a GitHub Issue

Happy Coding! 🎉





# 🚀 ERPNext Docker Setup Guide

This guide will help you set up ERPNext using Docker, with all files stored locally.

---

## 📌 Prerequisites
Ensure you have the following installed:
- Docker 🐳
- Docker Compose
- Git

---

## 📂 Step 1: Clone the ERPNext Docker Repository
Clone the official ERPNext Docker repository:
```bash
git clone https://github.com/frappe/erpnext_docker.git
cd erpnext_docker
```

---

## 📝 Step 2: Configure Local File Storage
Update the `docker-compose.yml` file to bind local directories for `sites` and `logs`. Ensure the following configuration is present in the `volumes` section:

```yaml
volumes:
  sites:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ./sites
  logs:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ./logs
```

Create the necessary directories for local storage:
```bash
mkdir -p sites/apps
mkdir -p sites/logs
mkdir -p sites/sites
```

---

## ▶️ Step 3: Start ERPNext Containers
Run the following command to start the containers:
```bash
docker compose up -d
```

---

## 🌐 Step 4: Create a New Site
Run the following command to create a new site:
```bash
docker exec -it <backend-container-name> bench new-site --no-mariadb-socket --admin-password admin --db-root-password admin sample.site
```
Replace `<backend-container-name>` with the name of your backend container (e.g., `erpnext_docker-backend-1`).

---

## 🏢 Step 5: Install ERPNext
Install the ERPNext app on the site:
```bash
docker exec -it <backend-container-name> bench --site sample.site install-app erpnext
```

---

## 🏥 Step 6: Install the Healthcare App (Optional)
To install the Healthcare app:
1. Clone the Healthcare app repository:
   ```bash
   git clone -b version-14 https://github.com/frappe/healthcare.git sites/apps/healthcare
   ```
2. Install the Healthcare app on the site:
   ```bash
   docker exec -it <backend-container-name> bench --site sample.site install-app healthcare
   ```

---

## 🔄 Step 7: Rebuild Assets
If you make changes to the code or install new apps, rebuild the assets:
```bash
docker exec -it <backend-container-name> bench build
```

---

## 🔄 Step 8: Restart Containers
Restart the containers to apply changes:
```bash
docker compose restart
```

---

## 🌐 Step 9: Access ERPNext
Open your browser and visit:
```
http://localhost:8080
```
Use the following login credentials:
- **Username**: Administrator
- **Password**: admin

---

## 📂 Directory Structure After Setup
After completing the setup, your directory structure should look like this:
```
erpnext_docker/
├── docker-compose.yml
├── sites/
│   ├── apps/
│   │   ├── frappe/
│   │   ├── erpnext/
│   │   └── healthcare/
│   ├── logs/
│   └── sites/
│       └── sample.site/
```

---

## 📌 Common Commands
Here are some common commands you may need:

### List Installed Apps
```bash
docker exec -it <backend-container-name> bench --site sample.site list-apps
```

### Clear Cache
```bash
docker exec -it <backend-container-name> bench clear-cache
docker exec -it <backend-container-name> bench clear-website-cache
```

### Check Logs
```bash
docker logs <container-name>
```

### Stop Containers
```bash
docker compose down
```

### Start Containers
```bash
docker compose up -d
```

---

## 🎯 Conclusion
You have successfully set up ERPNext using Docker with local file storage! 🎊 Now, explore the powerful features of ERPNext and customize it to fit your business needs. 🚀

---

## 🌟 Support & Contribution
Feel free to contribute or report issues! ✨

📌 **GitHub Repo**: [Your Repository Link]  
💬 **Community Support**: [ERPNext Forum](https://discuss.erpnext.com/)  
🐛 **Report Issues**: Open a GitHub Issue

Happy Coding! 🎉
