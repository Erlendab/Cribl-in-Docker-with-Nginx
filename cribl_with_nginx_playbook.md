
# Cribl in Docker with Nginx setup Guide
# PS: This playbook is not tested.

## 1. Install Docker and Docker Desktop
Ensure that you have Docker and Docker Desktop installed. Use the following commands for Docker installation:

```bash
# Update package information
sudo apt-get update

# Install Docker
sudo apt-get install docker-ce docker-ce-cli containerd.io -y

# Install Docker Compose (if not already installed)
sudo apt-get install docker-compose -y
```

## 2. Create the project folder
```bash
mkdir -p ~/myTestEnv/criblwithnginx
cd ~/myTestEnv/criblwithnginx
```

## 3. Create docker-compose.yml
Create a `docker-compose.yml` file:

```bash
touch docker-compose.yml
```

Paste the configuration from [this GitHub link](https://github.com/bentleymi/misc/blob/main/docker/cribl/docker-compose1.yml) into the newly created file.

## 4. Start Docker Compose
Start the Cribl environment with two worker containers:

```bash
docker-compose up -d --scale workers=2
```

## 5. Access the ubuntu container
To install additional software, access the running `ubuntu` container:

```bash
docker exec -it criblwithnginx-ubuntu-1 /bin/sh
```

## 6. Install dependencies in the ubuntu container
```bash
# Update package manager and install curl
apt-get update && apt-get install curl -y

# Download Cribl and install it in /opt
cd /opt
curl -Lso - $(curl https://cdn.cribl.io/dl/latest-arm64) | tar zxv

# Install git
apt-get install git -y

# Start Cribl
/opt/cribl/bin/cribl start
```

## 7. Install and configure Nginx
Install Nginx and a text editor:
```bash
apt-get install nginx -y
apt-get install nano -y
```

Edit the Nginx configuration:
```bash
nano /etc/nginx/conf.d/stream.conf
```

Add the following to the stream.conf file:

```nginx
stream {
    upstream 5140_tcp_backend {
        least_conn;
        server criblWithNginx-workers-1:5140;
        server criblWithNginx-workers-2:5140;
    }

    server {
        listen 5140;
        proxy_pass 5140_tcp_backend;
    }
}
```

## 8. Move and include the configuration
Move the `stream.conf` file and include it in the main `nginx.conf` file:

```bash
mv /etc/nginx/conf.d/stream.conf /etc/nginx/stream.conf
echo 'include /etc/nginx/stream.conf;' >> /etc/nginx/nginx.conf
```

## 9. Test and reload Nginx
Test the configuration:

```bash
nginx -t
```

You should see:

```bash
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Reload Nginx:

```bash
nginx -s reload
```

You should see:

```bash
signal process started
```

## 10. Verify Nginx is listening on port 5140
Run:

```bash
netstat -an | grep 5140
```

## 11. Configure Cribl to receive data from Nginx
1. Go to Cribl UI —> select worker group —> data —> sources —> syslog. Configure the syslog input, and ensure it's enabled.
2. Save and commit and deploy changes.
3. In Cribl UI, open sources—>syslog—>Live data. Start a live Data session (e.g 120 sec).
4. Run a test by sending data through Nginx with the following command:

```bash
curl localhost:5140 -d "test"
```

5. Run a test by sending data through your local computer (e.g Macbook) with the following command:
curl http://localhost:5140

### Notes
If both `/etc/nginx/modules-enabled/50-mod-stream.conf` and `/etc/nginx/nginx.conf` contain the line `load_module modules/ngx_stream_module.so;`, remove it from the latter.
