# Alternative ISO Serving with Nginx

[Back to Install](index.md#boot-the-machines)

If the Podman-based HTTP serving solution does not work for your environment, you can install nginx directly on the bastion host.

```shell
sudo dnf install -y nginx

cat <<EOF | sudo tee /etc/nginx/conf.d/iso.conf
server {
    listen       8080;
    server_name  _;

    location / {
        root   /home/user1/ocp/install;
        autoindex on;
    }
}
EOF

sudo chcon -Rt httpd_sys_content_t /home/user1/ocp/install

sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

sudo systemctl enable --now nginx
```

Test it:

```shell
curl http://localhost:8080/agent.x86_64.iso --head
```
