# Setting Up a Monitoring Stack with Prometheus, Grafana, and Node Exporter on Amazon EC2
In the world of DevOps, monitoring is key. It’s not enough to just build and deploy applications; we also need to ensure they’re running smoothly. That’s where tools like Prometheus and Grafana come in. In this blog post, we’ll walk through setting up a monitoring stack using Prometheus, Grafana, and Amazon EC2.
Before we dive into the setup, let’s briefly discuss the technologies we’ll be using:
#### Amazon EC2: 
Amazon Elastic Compute Cloud (EC2) I think this doesn’t need much introduction. It is a web service that provides resizable compute capacity in the cloud. You should be fairly comfortable using AWS and more specifically ec2 and security groups side of things.
#### Prometheus: 
Prometheus is an open-source systems monitoring and alerting toolkit. It collects metrics from configured targets [ec2 in our case] at given intervals, evaluates rule expressions, displays the results, and can trigger alerts if some condition is observed to be true.
#### Grafana: 
Grafana is a multi-platform open-source analytics and interactive visualization web application. It provides charts, graphs, and alerts for the web when connected to supported data sources, in our case, Prometheus.
#### Node Exporter: 
Node Exporter is a Prometheus exporter for hardware and OS metrics with pluggable metric collectors. It allows you to measure various machine resources such as memory, disk I/O, CPU, network, etc. We install Node Exporter on the nodes (in this case, the third EC2 instance) we want to monitor. Prometheus then scrapes and stores these metrics and Grafana visualizes them. 

## Setting Up the Monitoring Stack
Our setup involves three Amazon EC2 instances. One runs Prometheus, another runs Grafana, and the third is a general Amazon Linux instance that sends logs to Prometheus. Grafana then visualizes these logs.

To start, we'll launch three t2.micro EC2 instances using the Amazon Linux 2 operating system. We'll configure the security group to allow inbound traffic on a few specific ports:  22 for SSH access (you can also use the 'Instance Connect' feature instead, which I'd recommend),  Port 9100 for the Node Exporter, Port 9090 for Prometheus, Port 3000 for Grafana

Normally, it's best to create individual security groups for each EC2 instance. But to keep this lab simpler and shorter, we'll use the same security group for all three instances.

Just keep in mind that when building lab environments in the cloud, it's important to be cautious about security. Even though we're opening the EC2 instances to all IPv4 traffic, except for the SSH port, the other open ports aren't a major security risk since we'll only be using this setup for a short time.

Once we're done with the lab, we can quickly dismantle everything. I just wanted to mention security as a general consideration, but the focus here is on getting the lab environment set up efficiently.
Here is a screenshot of the security group I have configured. Your security group should look the same as this one.

<img src="/prometheus-grafana/pics/prom-1.png" width="500" />


## Creating the EC2 Instances
Use the following guide to create 3 instances you can create all three at once but increasing the number of instances in section "Number of instances".

1. Log in to the AWS Management Console: Go to the AWS website and sign in to your account.

2. Navigate to the EC2 service: In the AWS Management Console, locate and click on the "EC2" service.

3. Launch new instances: In the EC2 dashboard, click on the "Launch instances" button to start the instance creation process.

4. Choose the Amazon Machine Image (AMI): On the "Choose AMI" page, select the "Amazon Linux 2 AMI" as the operating system for the instances.

5. Choose the instance type: For the instance type, select the "t2.micro" option.

6. Configure the instance details: On the "Configure instance details" page, keep the default settings, or adjust them as needed for your use case.

7. Add storage: On the "Add storage" page, you can keep the default storage settings or modify them if required.

8. Configure the security group: Use the existing security group that you have just created.

9. Review and launch: Enter 3 in "Number of instances". Review the instance configuration details, and if everything looks good, click "Launch" to start the instance creation process.

10. Select a key pair: Choose an existing key pair or create a new one to be able to connect to the instances via SSH. Optional step.

11. Wait for the instances to launch: It may take a few minutes for the instances to be provisioned and become available.

Once the instances are running, you can connect to them using the selected key pair and the security group rules you configured. Remember, you can also use the "Instance Connect" feature instead of SSH if you prefer.
To make it easier to identify, you can name the security group the same way as shown in the picture below.


<img src="/prometheus-grafana/pics/prom-15.png" width="500" />



## Configuring the Frist EC2 Instance
On the first EC2 instance, we install Node Exporter, a Prometheus exporter for machine metrics. We start the Node Exporter service to begin collecting and exposing metrics. We then update the prometheus.yml configuration file on the second  EC2 instance to scrape metrics from the first EC2 instance. If you found the link in wget section to be not valid, a quick internet search will find the correct version and link at that time.

The provided commands set up the environment for running the Node Exporter. They create a dedicated "node_exporter" user account, download and extract the Node Exporter binary, and move the binary to a system-wide location (/usr/local/bin/). This preparation is typically done before configuring the systemd service file to run the Node Exporter.

```
$ sudo useradd --no-create-home --shell /bin/false node_exporter
$ mkdir lab
$ cd lab
$ wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
$ tar -xvf node_exporter-1.7.0.linux-amd64.tar.gz 
$ sudo mv node_exporter-1.7.0.linux-amd64/node_exporter  /usr/local/bin/

```
The command `$ sudo vi /etc/systemd/system/node_exporter.service` opens a systemd service file for the Node Exporter. This file sets up the Node Exporter as a system service that runs automatically when the system enters the multi-user state. The service runs as the "node_exporter" user and group, and starts the Node Exporter process located at `/usr/local/bin/node_exporter`.
```
$ sudo vi /etc/systemd/system/node_exporter.service
```
We intentionally avoid running this service as the root user. Instead, we specify the 'node_exporter' user to run the service, which is a more secure practice.

```
[Unit]
Description=Node Exporter Service File
[Service]
User=node_exporter
Group=node_exporter
ExecStart=/usr/local/bin/node_exporter
[Install]
WantedBy=multi-user.target
```
The `sudo systemctl daemon-reload` command reloads the systemd daemon to pick up changes to the service file. The `sudo systemctl enable --now node_exporter` command enables and starts the Node Exporter service. The `sudo systemctl status node_exporter` command checks the current status of the Node Exporter service.

```
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now node_exporter 
$ sudo systemctl status node_exporter
```

To access your metrics for your Node Exporter follow the following link [ http://<YOUR-EC2-IP>:9100 ];

http://<YOUR-EC2-IP>:9100     In my case: http://52.30.162.253:9100

<img src="/prometheus-grafana/pics/prom-2.png" width="500" />

Once there click on metrics or just type the following;

http://<YOUR-EC2-IP>:9100/metrics   In my case:http://52.30.162.253:9100/metrics

<img src="/prometheus-grafana/pics/prom-3.png" width="500" />

## Installing and Configuring Prometheus
On the second EC2 instance, we download and extract the latest Prometheus release.
The provided commands set up Prometheus monitoring on the second EC2 instance. A new "prometheus" system user is created, and a "lab" directory is set up to download and extract the latest Prometheus release. Directories for Prometheus configuration and data storage are created, and the necessary binaries and console files are copied and their ownership set to the "prometheus" user. The Prometheus configuration file is then created, defining the scraping job for the Node Exporter running on the same machine. A systemd service file is also set up to manage the Prometheus server, and the service is enabled and started. Finally, the Prometheus web UI can be accessed at the specified IP address and port, allowing you to view the monitored targets.

```
$ sudo useradd --no-create-home --shell /bin/false prometheus
$ mkdir lab
$ cd lab
$ wget https://github.com/prometheus/prometheus/releases/download/v2.45.2/prometheus-2.45.2.linux-amd64.tar.gz

$ tar -xvf prometheus-2.45.2.linux-amd64.tar.gz
$ cd prometheus-2.45.2.linux-amd64/
```


```
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```
Copy prometheus and promtool binary from prometheus-files folder to /usr/local/bin and change the ownership to prometheus user;
```
sudo cp prometheus /usr/local/bin/
sudo cp promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
```
Move the consoles and console_libraries directories from prometheus-files to /etc/prometheus folder and change the ownership to prometheus user
```
sudo cp -r consoles /etc/prometheus
sudo cp -r console_libraries /etc/prometheus
sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
```
Create the prometheus.yml file.
```
sudo vi /etc/prometheus/prometheus.yml
```
Here’s an example of a scraping job that scrapes metrics from a Node Exporter running on the same machine: [Dont forget to replace it to Your IP]
```
scrape_configs:
  - job_name: 'generalec2'
    static_configs:
      - targets: ['54.216.168.237:9100']
```

```
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

```
sudo vi /etc/systemd/system/prometheus.service
```

And Copy the following, save and exit.

```
[Unit]
Description=Prometheus Service File
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

```
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now prometheus
$ sudo systemctl status prometheus
```

To access Prometheus use the following link

Change the IP address to your own.


http://54.77.35.30:9090

if we visit the following link we should see our Node Exporter on 'generalec2' instance.

http://54.77.35.30:9090/targets


<img src="/prometheus-grafana/pics/prom-4.png" width="500" />


## Installing and Configuring Grafana

In the third EC2 instance, the provided commands set up Grafana. First, a Grafana repository is configured in the YUM package manager by creating a new repository file. This allows the system to download and install the latest version of Grafana from the official Grafana repository.

After setting up the repository, the Grafana package is installed using the YUM package manager. The systemd daemon is then reloaded, and the Grafana server is enabled and started.

Once the Grafana server is running, the Grafana web UI can be accessed at the specified IP address and port (in this case, `http://3.250.36.97:3000`). The user can then log in using the default "admin" username and the password "admin".

Finally, the user can add Prometheus as a data source in Grafana, allowing them to visualize and analyze the metrics collected by the Prometheus monitoring system set up on the second EC2 instance.

Let's Create a repository.

```
$ sudo vi /etc/yum.repos.d/grafana.repo
```

Enter the following, save and exit.

```
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
```

The following will install enable and start the Grafan service.

```
$ sudo yum install grafana -y
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now grafana-server
```
Access the Grafana web interface by visiting the URL with the public IP address or DNS name of the EC2 instance where Grafana is installed. Replace the IP address or DNS name in the URL with the one that corresponds to your specific environment.

http://3.250.36.97:3000

After logging in to the Grafana web interface using the default admin credentials, you will be taken to the Grafana homepage. At this point, you can optionally change the username and password to your preferred settings.

<img src="/prometheus-grafana/pics/prom-5.png" width="300" />

<img src="/prometheus-grafana/pics/prom-6.png" width="300" />

Once on homepage you can click on "Data Sources" to add prometheus to it.

<img src="/prometheus-grafana/pics/prom-7.png" width="300" />

click on Prometheus.

<img src="/prometheus-grafana/pics/prom-8.png" width="500" />

The only change you are going to make here is enter the server URL of Prometheus, leave everythig default.

<img src="/prometheus-grafana/pics/prom-11.png" width="500" />

After clicking on "Save & Test", you should see the following message.

<img src="/prometheus-grafana/pics/prom-12.png" width="500" />


From the Grafana homepage, navigate to the Dashboards section. Here, you will be presented with several options. For this example, we will import a dashboard called "Node Explorer Full" with the ID "1860". You can access this dashboard and many others by visiting the provided link.

https://grafana.com/grafana/dashboards/

After entering the dashboard ID "1860", you will be taken to a page where you can select the data source for the dashboard. As shown in the picture, choose "Prometheus" as the data source for this dashboard.

<img src="/prometheus-grafana/pics/prom-13.png" width="500" />

Once the dashboard is loaded, you will be presented with a beautiful overview of the "generalec2" stats, providing valuable insights into the performance and health of your EC2 instance.

 <img src="/prometheus-grafana/pics/prom-14.png" width="500" />

This setup provides a robust monitoring solution that can help us understand the performance and health of our applications and infrastructure. Prometheus collects and stores time series data such as metrics from our third EC2 instance. Grafana allows us to visualize this data in a user-friendly way, making it easier to interpret and understand.
Furthermore, both Prometheus and Grafana are open-source and have active communities, which means they’re continually being improved. They’re also highly configurable and can be tailored to suit the specific needs of your project.
In conclusion, setting up a monitoring stack with Prometheus, Grafana, and Amazon EC2 is a great way to keep track of your applications and infrastructure. It might seem daunting at first, especially if you’re new to these technologies, but with a bit of practice, you’ll find it’s a worthwhile investment.


 


