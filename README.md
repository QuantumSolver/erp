# ERPNext Installation on a Clean Machine

## Prerequisites

1. **Install Required Packages**
    - First, ensure that `curl` and `docker.io` are installed:
    ```bash
    sudo apt install curl docker.io
    ```

2. **Add Docker Group and User**
    - Add your user to the Docker group to allow running Docker commands without `sudo`:
    ```bash
    sudo groupadd docker
    sudo usermod -aG docker ubuntu
    newgrp docker
    ```

3. **Install Docker Compose**
    - Set up Docker Compose plugin:
    ```bash
    HOME=~
    DOCKER_CONFIG=${DOCKER_CONFIG:-$HOME/.docker}
    mkdir -p $DOCKER_CONFIG/cli-plugins
    curl -SL https://github.com/docker/compose/releases/download/v2.31.0/docker-compose-linux-aarch64 -o $DOCKER_CONFIG/cli-plugins/docker-compose
    chmod +x $DOCKER_CONFIG/cli-plugins/docker-compose
    docker-compose -v
    ```

## Clone the Frappe Docker Repository

1. **Clone the Repository**
    ```bash
    git clone https://github.com/frappe/frappe_docker
    cd frappe_docker
    ```

2. **Set Environment Variables**
    - Copy the example environment file and edit it:
    ```bash
    cp example.env .env
    vi .env
    ```

3. **Build the Docker Images**
    - Build the images using the `docker buildx bake` command:
    ```bash
    docker buildx bake
    ```

4. **Add User to Docker Group Again**
    ```bash
    sudo usermod -aG docker $USER
    newgrp docker
    ```

5. **Clean Up Old Images (Optional)**
    ```bash
    docker image rm <image_id>  # Example: docker image rm 37c7e0c35956
    docker system prune -f
    ```

6. **Build Specific Versions**
    - Example with Frappe version `v14.2.0` and ERPNext version `v14.0.0`:
    ```bash
    FRAPPE_VERSION=v14.2.0 ERPNEXT_VERSION=v14.0.0 docker buildx bake
    ```

## Install Custom Docker Repository

1. **Clone the Custom Repository**
    ```bash
    git clone https://github.com/castlecraft/custom_frappe_docker.git
    cd custom_frappe_docker/
    ```

2. **Modify Configuration Files**
    - Edit configuration files to meet your needs:
    ```bash
    vi base_versions.json
    vi ci/clone-apps.sh
    vi images/backend.Dockerfile
    vi images/frontend.Dockerfile
    vi docker-bake.hcl
    ```

3. **Run the Build**
    ```bash
    ./ci/clone-apps.sh
    docker buildx bake -f docker-bake.hcl --load
    ```

## Set Up Traefik and MariaDB

1. **Create Traefik Configuration File**
    ```bash
    mkdir ~/gitops
    echo 'TRAEFIK_DOMAIN=trfk.npesnam.com' > ~/gitops/traefik.env
    echo 'EMAIL=marcques.ethanmouton@npesnam.com' >> ~/gitops/traefik.env
    echo 'HASHED_PASSWORD='$(openssl passwd -apr1 changeit | sed 's/\$/\\\$/g') >> ~/gitops/traefik.env
    ```

2. **Start Traefik Containers**
    ```bash
    docker compose --project-name traefik --env-file ~/gitops/traefik.env -f docs/compose/compose.traefik.yaml -f docs/compose/compose.traefik-ssl.yaml up -d
    ```

3. **Set Up MariaDB**
    ```bash
    echo "DB_PASSWORD=changeit" > ~/gitops/mariadb.env
    docker compose --project-name mariadb --env-file ~/gitops/mariadb.env -f docs/compose/compose.mariadb-shared.yaml up -d
    ```

## Configure ERPNext

1. **Create and Configure ERPNext Environment**
    ```bash
    cp example.env ~/gitops/erpnext-one.env
    sed -i 's/DB_PASSWORD=123/DB_PASSWORD=changeit/g' ~/gitops/erpnext-one.env
    sed -i 's/DB_HOST=/DB_HOST=mariadb-database/g' ~/gitops/erpnext-one.env
    sed -i 's/DB_PORT=/DB_PORT=3306/g' ~/gitops/erpnext-one.env
    echo 'ROUTER=erpnext-one' >> ~/gitops/erpnext-one.env
    echo "SITES=\erp1.npesnam.com\,\erp2.npesnam.com\" >> ~/gitops/erpnext-one.env
    echo "BENCH_NETWORK=erpnext-one" >> ~/gitops/erpnext-one.env
    ```

2. **Generate Config for ERPNext**
    ```bash
    docker compose --project-name erpnext-one --env-file ~/gitops/erpnext-one.env -f compose.yaml -f overrides/compose.erpnext.yaml -f overrides/compose.redis.yaml -f docs/compose/compose.multi-bench.yaml -f docs/compose/compose.multi-bench-ssl.yaml config > ~/gitops/erpnext-one.yaml
    ```

3. **Start ERPNext Containers**
    ```bash
    docker compose --project-name erpnext-one -f ~/gitops/erpnext-one.yaml up -d
    ```

4. **Create New Sites in ERPNext**
    ```bash
    docker compose --project-name erpnext-one exec backend bench new-site erp1.npesnam.com --mariadb-root-password changeit --install-app erpnext --admin-password changeit
    docker compose --project-name erpnext-one exec backend bench new-site erp2.npesnam.com --mariadb-root-password changeit --install-app erpnext --admin-password changeit
    ```

5. **Install Additional Apps (HRMS)**
    ```bash
    docker compose --project-name erpnext-one exec backend bench --site erp1.npesnam.com install-app hrms
    docker compose --project-name erpnext-one exec backend bench --site erp2.npesnam.com install-app hrms
    ```

## Final Cleanup and Maintenance

1. **Shut Down ERPNext Containers**
    ```bash
    docker compose --project-name erpnext-one -f ~/gitops/erpnext-one.yaml down
    ```

2. **Prune Docker System**
    ```bash
    docker system prune -f
    ```

## Conclusion

- After following the above steps, you should have ERPNext running on Docker with MariaDB and Traefik. You can access the ERPNext site via the provided domain (`erp1.npesnam.com`, `erp2.npesnam.com`) and further customize your installation as needed.
