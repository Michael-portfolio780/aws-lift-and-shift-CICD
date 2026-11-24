
Vagrant.configure("2") do |config|

  config.vm.box = "ubuntu/jammy64"

  config.vm.hostname = "jenkins-sonarqube"

  # Private network
  config.vm.network "private_network",
    ip: "192.168.56.20"

  # HTTP access
  config.vm.network "forwarded_port",
    guest: 80,
    host: 8080,
    auto_correct: true

  # VM resources
  config.vm.provider "virtualbox" do |vb|
    vb.name = "Jenkins-SonarQube-Lab"
    vb.memory = 8192
    vb.cpus = 4
  end

  config.vm.provision "shell", inline: <<-SHELL

    set -e

    # Update Ubuntu
    apt-get update
    apt-get install -y \
      ca-certificates \
      curl \
      gnupg \
      ufw

    # Install Docker
    install -m 0755 -d /etc/apt/keyrings

    curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
      -o /etc/apt/keyrings/docker.asc

    chmod a+r /etc/apt/keyrings/docker.asc

    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
      > /etc/apt/sources.list.d/docker.list

    apt-get update

    apt-get install -y \
      docker-ce \
      docker-ce-cli \
      containerd.io \
      docker-buildx-plugin \
      docker-compose-plugin

    systemctl enable docker
    systemctl start docker

    # Configure SonarQube kernel requirement
    sysctl -w vm.max_map_count=524288
    sysctl -w fs.file-max=131072

    echo "vm.max_map_count=524288" >> /etc/sysctl.conf
    echo "fs.file-max=131072" >> /etc/sysctl.conf

    # Configure firewall
    ufw allow 22/tcp
    ufw allow 80/tcp
    ufw --force enable

    # Create project directory
    mkdir -p /opt/jenkins-sonarqube

    cd /opt/jenkins-sonarqube

    # Create Docker Compose configuration
    cat > compose.yaml <<'EOF'
services:

  jenkins:
    image: jenkins/jenkins:lts-jdk21
    container_name: jenkins
    restart: unless-stopped
    environment:
      JAVA_OPTS: "-Djenkins.install.runSetupWizard=true --prefix=/jenkins"
    volumes:
      - jenkins_home:/var/jenkins_home
    networks:
      - devops

  postgres:
    image: postgres:16
    container_name: sonar-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar_lab_password
      POSTGRES_DB: sonarqube
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - devops

  sonarqube:
    image: sonarqube:lts-community
    container_name: sonarqube
    restart: unless-stopped
    depends_on:
      - postgres
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://postgres:5432/sonarqube
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar_lab_password
      SONAR_WEB_CONTEXT: /sonarqube
    volumes:
      - sonar_data:/opt/sonarqube/data
      - sonar_extensions:/opt/sonarqube/extensions
      - sonar_logs:/opt/sonarqube/logs
    networks:
      - devops

  nginx:
    image: nginx:stable
    container_name: devops-nginx
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - jenkins
      - sonarqube
    networks:
      - devops

volumes:
  jenkins_home:
  postgres_data:
  sonar_data:
  sonar_extensions:
  sonar_logs:

networks:
  devops:
    driver: bridge
EOF

    # Create Nginx reverse proxy configuration
    cat > nginx.conf <<'EOF'
server {
    listen 80;
    server_name _;

    location = / {
        return 302 /jenkins/;
    }

    location /jenkins/ {
        proxy_pass http://jenkins:8080/jenkins/;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_read_timeout 90s;
        client_max_body_size 50m;
    }

    location /sonarqube/ {
        proxy_pass http://sonarqube:9000/sonarqube/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 300s;
        client_max_body_size 50m;
    }
}
EOF

    # Start services
    docker compose up -d

    echo "Jenkins and SonarQube provisioning initiated."

  SHELL

end
