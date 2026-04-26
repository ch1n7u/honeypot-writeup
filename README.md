## Introduction

If you want to learn how real-world attacks look without putting production systems at risk, a honeypot lab is a great way to do it. A honeypot is a deliberately exposed system or service designed to attract and record malicious activity.

T-Pot is useful because it bundles multiple honeypots into one platform and includes built-in monitoring and analytics. It runs the sensors in Docker containers and ships logs into the Elastic stack (Elasticsearch, Logstash, Kibana), so you can both capture and analyze attack data from one place. This is especially helpful for security learning, threat research, and blue-team experimentation.

## Disclaimer

This setup is for educational and research use only. Make sure your usage follows local laws, your organization policies, and AWS terms of service.

## Prerequisites

- AWS account with access to Free Tier services
- Basic Linux and SSH command-line knowledge
- A terminal on Linux with SSH available
- Realistic T-Pot system requirements:
- Recommended for meaningful data collection: at least 2 vCPU, 4 to 8 GB RAM, and 64+ GB disk
- Free Tier testing (t2.micro or t3.micro) works for lightweight learning, but performance and sensor coverage will be limited

## Architecture Overview

At a high level, T-Pot runs as a Docker-based stack:
- Multiple honeypot containers collect attacker interactions
- Log pipeline and storage are handled by the Elastic stack
- Web dashboards (including Kibana views) help explore events, source IPs, target services, and timelines

## Creating and Setting Up an EC2 Instance

1. In the AWS Console, go to **EC2 -> Instances -> Launch Instances**.
2. Set an instance name, for example: **HoneyPot**.
3. In **Application and OS Images (AMI)**, choose **Debian**.

![Debian AMI selection screen](images/ec2-debian-ami-selection.png)

4. For AWS Free Tier, select **t2.micro** or **t3.micro**.

![EC2 instance type selection](images/ec2-instance-type-selection.png)

5. Create a new **Key Pair**:
   - Name: **tpot**
   - Key pair type: **RSA**
   - Private key format: **.pem** (good choice for Linux)
   - Download and store the PEM file securely

![EC2 key pair creation settings](images/ec2-key-pair-creation.png)

6. In **Network Settings**, enable **Auto-assign public IP**.
   - Security group ports will be configured later.

![EC2 network settings with public IP enabled](images/ec2-network-public-ip-settings.png)

7. Set storage size to **28 GB**.

![EC2 storage configuration set to 28 GB](images/ec2-storage-configuration-28gb.png)

8. Launch the instance and wait until it reaches the **running** state.

![EC2 instance launch running state](images/ec2-instance-launch-status.png)

## Connecting to the Instance

Before T-Pot changes SSH settings, connect on the default SSH port (22).

1. Open a terminal and go to the folder where **tpot.pem** was downloaded.
2. Restrict key permissions:

~~~bash
chmod 600 tpot.pem
~~~

3. Debian AMI default user is **admin**.
4. Connect with SSH:

~~~bash
ssh -i tpot.pem admin@AWS_PUBLIC_IP
~~~

## Installing T-Pot

1. Update package lists:

~~~bash
sudo apt update
~~~

2. Start T-Pot installation:

~~~bash
env bash -c "$(curl -sL https://github.com/telekom-security/tpotce/raw/master/install.sh)"
~~~

Official repository: https://github.com/telekom-security/tpotce

![T-Pot installation command output](images/tpot-installation-command-screen.png)

3. When prompted for T-Pot type, select **h**.

![T-Pot profile selection prompt with option h](images/tpot-profile-selection-h.png)

4. Set the web username and password.
5. Reboot after installation:

~~~bash
sudo reboot
~~~

![T-Pot installation reboot prompt](images/tpot-installation-reboot-prompt.png)

## Setting Up Security Groups

After reboot, T-Pot uses custom management ports. Update inbound rules in:
**EC2 -> Security Groups -> Inbound rules**.

For the source IP column, do not leave SSH wide open unless you truly need to. Use your own public IP or a tight CIDR range wherever possible. Opening everything to `0.0.0.0/0` is the fastest way to invite unnecessary scans and brute-force attempts.

### Ports Reference

| Purpose                  | Port(s) | Notes                         |
| ------------------------ | ------: | ----------------------------- |
| SSH before T-Pot install |      22 |                               |
| SSH after T-Pot install  |   64295 | Restrict source to your IP    |
| T-Pot Web UI             |   64297 | Restrict source to your IP    |
| Honeypot listeners       | 1-64000 | Open these ports for everyone |

Then reconnect using the updated SSH port:

~~~bash
ssh -i tpot.pem admin@AWS_PUBLIC_IP -p 64295
~~~

## Starting T-Pot Services with Docker Compose

Run these commands in order:

~~~bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

docker-compose version

cd ~/tpotce

sudo docker-compose up -d

docker ps
~~~

![Docker containers running for T-Pot services](images/tpot-docker-containers-running.png)

## Verification Steps After Installation

Use these checks to confirm the deployment is healthy:

1. Verify Docker service state:

~~~bash
sudo systemctl status docker
~~~

2. Verify containers are running:

~~~bash
docker ps
~~~

3. Confirm T-Pot ports are listening:

~~~bash
sudo ss -tulpen | grep -E '64295|64297'
~~~

If these checks look good, the platform is ready for web access.

## Accessing the Web UI

Open this URL in your browser:
https://AWS_PUBLIC_IP:64297

Log in with the credentials created during setup. The dashboard gives you a quick operational view of your honeypots and incoming activity. In Kibana views, you can filter by source IP, destination port, time range, and event type to start basic log analysis.
	
![T-Pot web UI login and dashboard screen](images/tpot-web-ui-login-dashboard.png)

## Security Considerations

- Use strong, unique passwords for T-Pot web access and rotate them periodically.
- Keep your PEM file private and never commit it into repositories.
- Do not expose more inbound ports than necessary for your test goals.
- Wide-open honeypot ranges can increase traffic and generate unexpected AWS costs.
- Treat this as an isolated lab, not a production security control.

## Cost Awareness

- AWS Free Tier has limits on compute hours, storage, and bandwidth.
- You can still be charged for extra EBS storage, outbound data transfer, Elastic IP usage patterns, or traffic spikes.
- Monitor billing and set AWS Budgets alerts before exposing large port ranges.

## Troubleshooting:

### SSH connection fails

- Confirm instance is running and has a public IP.
- Confirm you are using the correct user (**admin**) and key permissions (**chmod 600 tpot.pem**).
- Use port **22** before install and **64295** after T-Pot install.

### Ports not reachable

- Recheck security group inbound rules and source CIDR entries.
- Verify local firewall/NACL rules are not blocking traffic.

### Docker or containers not running

- Check Docker status:

~~~bash
sudo systemctl status docker
~~~

- Restart Docker if needed:

~~~bash
sudo systemctl restart docker
~~~

- Then verify containers:

~~~bash
docker ps
~~~

### Web UI not loading

- Confirm you are using **https://AWS_PUBLIC_IP:64297**.
- Check that port **64297** is open in security groups.
- Wait a few minutes after reboot or service startup for all containers to initialize.

## Conclusion

You now have a working AWS-hosted T-Pot honeypot lab with web-based visibility into incoming attack traffic. A good next step is to spend time in Kibana building simple filters and dashboards, then track trends like top source IPs, most targeted ports, and daily event volume.
