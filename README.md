# acit2420-assignment3part2

This project will allow you setup a static index.html that contains basic system information using the generate_index script inside your home directory's bin folder. Additionally, 2 digitalocean Arch Linux droplets are also created to be attached to a load balancer, which is a helpful tool for server traffic managing.

### Short manual on Creating the Server Droplets
1.In the digital ocean website, create a project if you haven't already. 
2.From the create drop down, select droplets.
3.Choose San Francisco as the region
4.In the Datacenter box, select San Francisco Data center 3.
5.From choose an image tab, select the arch-linux image of your choice.
6.For size, go with basic
7.For CPU options, choose either Premium Intel 8$ per month or Premium AMD 7$ a month.
8.Select your SSH key
9.In finalize details section, increase your droplet to 2.
10.Type a hostname of your choosing.
11.Type web in the Tags box.
12.Select a project.
13.Create and done.

With this, we have successfully created our servers. 

### Creating the load balancer
In the project where you created the 2 web servers, we will now create a load balancer to attach the server to.

1.From the create drop down, select Load Balancers
2.Select Regional for load balance type
3.Select San Francisco 3 as the region
4.Leave VPC network as default
5.Select External(Public) for your network visibility
5.Scaling configuration will be left as default
7.Under connect droplets, type in web and select it.
8.Enter a name for your load balancer if you'd like.
9.Click create load balancer

Congratulations, you have sucessfully created a load balancer with 2 servers attached.

### Task 1

We need to create a system user called webgen and supply the user with its own home directory and a login shell for a non-login user. webgen will have ownership of the directory in its entirety as well. 

To create the user with the necessary criteria, run:
```
sudo useradd -r -m -d /var/lib/webgen -s /usr/sbin/nologin
```

Now we need to make bin and HTML folders inside webgen's home directory.
```
sudo mkdir -p /var/lib/webgen/bin
sudo mkdir -p /var/lib/webgen/HTML
```

Ext, we need to copy your current generate_index directory to webgen's bin directory
```
cp ./generate_index /var/lib/webgen/bin
```

After the directory is set up, be sure to give webgen the proper ownership
```
sudo chown -R webgen:webgen /var/lib/webgen
```

### Task 2

We will proceed to creating a .service file and .timer file that runs at 5am everyday automatically.

To create the generate-index.service file:
```
sudo nvim generate-index.service
```

Inside the file, add:
```[Unit]
Description=This file runs the generate_index script
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/var/lib/webgen/bin/generate_index
User=webgen
Group=webgen

[Install]
WantedBy=multi-user.target
```

To create the .timer file:
```
sudo nvim generate-index.timer
```

Inside the file, add:
```
[Unit]
Description=This will run the generate_index script everyday at 5 am

[Timer]
OnCalendar=*-*-* 5:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Next, we need to place these 2 files in their correct directories:
```
sudo cp generate-index.timer /etc/systemd/system/
```
and
```
sudo cp generate-index.service /etc/systemd/system/
```

Before running the scripts, there are a few prerequisite steps to do. First, reload the systemd:
```
sudo systemctl daemon-reload
```
Start the services for both files:
```
sudo systemctl start generate-index.service
sudo systemctl start generate-index.timer
```
Next, enable the services:
```
sudo systemctl enable generate-index.service
sudo systemctl enable generate-index.timer
```
Feel free to make sure .service and .timer are enabled and active. Do so with:
```
sudo systemctl status generate-index.service
sudo systemctl status generate-index.timer
```

You can check logs for execution for both the scripts with:
```
sudo journalctl -u generate-index.service
```
```
sudo journalctl -u generate-index.timer
```

### Task 3

This task involves using webgen user to run the nginx server.

Install nginx if you haven't already:
```
sudo pacman -S nginx
```

Create a server block file for nginx server. For this, we need to make 2 directories:
```
sudo mkdir /etc/nginx/sites-available
sudo mkdir /etc/nginx/sites-enabled
```

Create the configuration file for the nginx server block:
```
sudo nvim /etc/nginx/sites-available/server-block.conf
```
Add the following content to your file:
```
server {
  listen 80;
  listen [::]:80;

  server_name <your server ip>;

  root /var/lib/webgen/HTML;
  index index.html;

  	location / {
    	try_files $uri $uri/ =404;
	}
}
```

Next, we will configure the nginx.conf file. In the .config file, change:
```
user http;
``` 
to 
```
user webgen;
```
Add the following block of code for http:
```
http {
    charset utf-8;
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    server_tokens off;
    log_not_found off;
    types_hash_max_size 4096;
    client_max_body_size 16M;

    # MIME
    include mime.types;
    default_type application/octet-stream;

    # logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log warn;

    # load configs
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```
Finally, create a symbolic link for the server-block.conf file:
```
sudo ln -s /etc/nginx/sites-available/server-block.conf /etc/nginx/sites-enabled/server-block.conf
```

Make sure to reload the daemon before proceeding:
```
sudo systemctl daemon-reload
```
Now we can start nginx with:
```
sudo systemctl start nginx
```
Enable the service with:
```
sudo systemctl enable nginx
```
Check if the service is enabled and active with:
```
sudo systemctl status nginx
```

To verify the nginx configuration for any errors:
```
sudo nginx -t
```

We use a server block file for the sake of modularity; for better organization and troubleshooting without affecting nginx.conf


### Task 4

The last task concludes the project. Once the nginx server is enabled and active, we will use ufw to set up a firewall.

Install ufw if you don't have it
```
sudo pacman -S ufw
```
Set SSH and HTTP to be allowed from anywhere
```
sudo ufw allow SSH
```
```
sudo ufw allow http
```
Lastly, enable the firewall service
```
sudo ufw enable
```
Make sure the firewall rules are set
```
sudo ufw status verbose
```

Congratulations on reaching the end. You have completed all the tasks of running a nginx server with user webgen.

You can enhance the error checking in the `generate_index` script by including write permission checks with
```
if [[ ! -w "$OUTPUT_DIR" ]]; then
    echo "Error: Write permissions for $OUTPUT_DIR is disabled." >&2
    exit 1
fi
```


















