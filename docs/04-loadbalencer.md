# 03 Set Up The Loadbalancer

In this lab you will set up a load balancer to distribute traffic across the Kubernetes API servers and Ingress controllers using Nginx.

## Architecture Overview

The load balancer will route traffic to:
- **Port 6443**: Kubernetes API Server (controller1, controller2, controller3)
- **Port 80**: HTTP Ingress for workload (worker1)
- **Port 443**: HTTPS Ingress for workload (worker1)

---

## Install Nginx on Ubuntu/Debian

```bash
sudo apt-get update
sudo apt-get install -y nginx
```

---

## Configure Nginx

Edit the nginx configuration file `/etc/nginx/nginx.conf` replace the entire content with:

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
}

stream {
    upstream k8s_api {
        server controller1:6443;
        server controller2:6443;
        server controller3:6443;
    }

    upstream http_ingress {
        server worker1:80;
    }

    upstream https_ingress {
        server worker1:443;
    }

    server {
        listen 6443;
        proxy_pass k8s_api;
    }

    server {
        listen 80;
        proxy_pass http_ingress;
    }

    server {
        listen 443;
        proxy_pass https_ingress;
    }
}
```

---

## Start Nginx

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Check Status

```bash
sudo systemctl status nginx
```
