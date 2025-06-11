# Dockerized solution for running NGINX and the AWS CloudWatch Agent on Amazon Linux 2 using Supervisord.
# Why are we using this approach 
Using **Supervisord** to manage NGINX and the AWS CloudWatch agent in a single Docker container was chosen for its simplicity and reliability in handling multiple processes. Below, I’ll explain why Supervisord was used, why it works well for this scenario, and why other approaches were considered more complicated. I’ll also provide context for the provided Docker setup and address the specific needs of your trial test (running NGINX and CloudWatch agent in a single `amazonlinux:2`-based container to push NGINX access logs and disk metrics to CloudWatch).

---

### Why Supervisord Was Used
Docker containers are designed to run a single foreground process by default. When you need to run multiple processes (e.g., NGINX and the CloudWatch agent) in a single container, you need a process manager to start, monitor, and keep both processes running. Supervisord was chosen for the following reasons:

<details>
   <summary>Click to view Detailed Explaination</summary>

1. **Simple Process Management**:
   - Supervisord is a lightweight process control system that can start, stop, and monitor multiple processes, ensuring they run concurrently and restart if they fail.
   - It allows you to define NGINX and the CloudWatch agent as separate programs in a single configuration file (`supervisord.conf`), making it easy to manage both in one container.

2. **Foreground Process Requirement**:
   - Docker requires the container’s entrypoint to run in the foreground to keep the container alive. Supervisord runs as the foreground process (`nodaemon=true`) and manages NGINX and the CloudWatch agent in the background, satisfying this requirement.
   - Without a process manager, you’d need complex scripting to keep both processes running and ensure the container doesn’t exit.

3. **Ease of Configuration**:
   - Supervisord uses a simple INI-style configuration file (`supervisord.conf`) to define how to run NGINX (`nginx -g 'daemon off;'`) and the CloudWatch agent (`start-amazon-cloudwatch-agent`).
   - It redirects logs to `/dev/stdout` and `/dev/stderr`, making them accessible via `docker logs`, which simplifies debugging.

4. **Compatibility with `amazonlinux:2`**:
   - Supervisord is easily installed via `pip3 install supervisor` on `amazonlinux:2`, requiring minimal setup compared to other process managers.
   - It integrates seamlessly with the Docker environment and the CloudWatch agent.

5. **Reliability**:
   - Supervisord automatically restarts processes if they crash (`autorestart=true`), ensuring NGINX and the CloudWatch agent remain operational.
   - It handles process dependencies and logging, reducing the risk of one process failing and causing the container to exit.

</details>

---

### Why Supervisord Works Well
Supervisord is particularly effective for your scenario because:
- **Single Container Requirement**: Your trial test requires running NGINX and the CloudWatch agent in one container. Supervisord elegantly manages both processes without requiring separate containers or complex orchestration.
- **NGINX and CloudWatch Agent Compatibility**: NGINX needs to run with `daemon off;` to stay in the foreground, and the CloudWatch agent runs as a long-lived process. Supervisord handles both seamlessly.
- **Log and Metric Collection**: Supervisord ensures both processes run continuously, allowing the CloudWatch agent to collect NGINX access logs (`/var/log/nginx/access.log`) and disk metrics (`disk_used`, `disk_free`) for EBS volumes, pushing them to CloudWatch without interruption.
- **Scalability Across Instances**: The Supervisord-based setup is portable across multiple EC2 instances, as it doesn’t rely on instance-specific configurations beyond the `config.json` file, which uses a universal wildcard (`"resources": ["*"]`) for disk monitoring.

---

### Why Other Approaches Were Complicated
Several alternative approaches to running multiple processes in a single Docker container were considered but deemed more complicated or less suitable for your trial test:

<details>
   <summary>Click to view Detailed Explaination</summary>

1. **Custom Shell Script**:
   - **Approach**: Use a shell script as the container’s entrypoint to start NGINX and the CloudWatch agent (e.g., `#!/bin/bash\nnginx -g "daemon off;" &\n/opt/aws/amazon-cloudwatch-agent/bin/start-amazon-cloudwatch-agent`).
   - **Complications**:
     - **Process Management**: The script doesn’t handle process monitoring or restarts. If NGINX or the CloudWatch agent crashes, the container may continue running without one of the services, breaking log or metric collection.
     - **Signal Handling**: Docker sends signals (e.g., SIGTERM) to the entrypoint process. A shell script may not properly forward signals to NGINX and the CloudWatch agent, leading to ungraceful shutdowns.
     - **Log Handling**: Managing logs from both processes is harder, as you’d need to redirect them manually to `/dev/stdout` and `/dev/stderr`.
     - **Complexity**: Writing and debugging a robust script to handle process failures, logging, and signal propagation is more error-prone than using Supervisord’s built-in features.

2. **Docker Init Systems (e.g., tini, dumb-init)**:
   - **Approach**: Use a lightweight init system like `tini` or `dumb-init` to handle process cleanup and signal forwarding, combined with a script to start both processes.
   - **Complications**:
     - **Limited Process Management**: These tools are designed for PID 1 cleanup (e.g., reaping zombie processes) but lack Supervisord’s ability to monitor and restart processes.
     - **Script Dependency**: You’d still need a custom script to start NGINX and the CloudWatch agent, inheriting the same issues as the shell script approach.
     - **Setup Overhead**: Installing and configuring `tini` or `dumb-init` adds complexity without providing the full process management capabilities needed.

3. **Multiple Containers**:
   - **Approach**: Run NGINX and the CloudWatch agent in separate containers, using Docker Compose or networking to share logs and metrics.
   - **Complications**:
     - **Violates Single Container Requirement**: Your trial test explicitly requires a single container, making this approach unsuitable.
     - **Increased Complexity**: Managing multiple containers requires orchestration (e.g., Docker Compose, ECS), shared volumes for logs, and network configuration, which is overkill for a trial test.
     - **Resource Overhead**: Multiple containers consume more resources than a single container with Supervisord.
     - **Log Access**: The CloudWatch agent container would need access to NGINX’s log directory via a shared volume, adding setup complexity.

4. **Running CloudWatch Agent as a Sidecar Process**:
   - **Approach**: Run the CloudWatch agent as a background process within the container, with NGINX as the primary process.
   - **Complications**:
     - **Process Management**: Without a process manager, the CloudWatch agent may exit unexpectedly, and Docker won’t restart it.
     - **Container Lifecycle**: If NGINX is the entrypoint and exits, the container stops, even if the CloudWatch agent is still running.
     - **Debugging**: Background processes are harder to monitor without a tool like Supervisord to handle logs and restarts.

5. **Using `systemd` or Full Init System**:
   - **Approach**: Use a full init system like `systemd` inside the container to manage NGINX and the CloudWatch agent.
   - **Complications**:
     - **Not Docker-Friendly**: `systemd` is designed for full OS environments and requires special container configurations (e.g., `--privileged`, custom volumes), which deviate from Docker best practices.
     - **Resource Heavy**: `systemd` is overkill for a simple trial test, consuming more resources than Supervisord.
     - **Setup Complexity**: Configuring `systemd` in a container is complex and error-prone compared to Supervisord’s lightweight INI configuration.

</details>

---

### How Supervisord Simplifies the Setup
Supervisord addresses the challenges of other approaches by:
- **Unified Process Control**: It starts both NGINX and the CloudWatch agent, monitors their status, and restarts them if they fail.
- **Simple Configuration**: The `supervisord.conf` file is straightforward, requiring only a few lines to define each process.
- **Log Management**: It redirects process logs to `/dev/stdout` and `/dev/stderr`, making them accessible via `docker logs`.
- **Docker Compatibility**: It runs as the foreground process, keeping the container alive and handling Docker signals correctly.
- **Lightweight**: Supervisord is installed via `pip3` and has minimal overhead, suitable for the `amazonlinux:2` base image.

---

### Context of the Setup
Your trial test requires a single Docker container (using `amazonlinux:2`) to run NGINX and the CloudWatch agent, pushing NGINX access logs and disk metrics (`disk_used`, `disk_free`) for EBS volumes to CloudWatch. The provided setup uses:
- **Dockerfile**: Installs NGINX (via EPEL), Supervisord (via `pip3`), and the CloudWatch agent, ensuring compatibility with `amazonlinux:2`.
- **Supervisord Configuration**: Manages NGINX and the CloudWatch agent, ensuring both run reliably.
- **CloudWatch Agent Configuration**: Uses `"resources": ["*"]` to automatically detect all mounted filesystems (including EBS volumes) and `${aws:InstanceId}` to uniquely identify metrics and logs across multiple instances.

Supervisord was critical to making this work in a single container, as it eliminates the need for complex scripting or multiple containers, which would complicate deployment across multiple EC2 instances.

---

### Files (Recap for Reference)
These are the same files provided,  included here for completeness to show how Supervisord integrates with the setup. They are ready and support automatic disk detection across multiple instances.

#### 1. Dockerfile
Installs dependencies and sets up Supervisord.

<details>
   <summary>Click to view the code</summary>


```
FROM amazonlinux:2

# Enable EPEL repository and install necessary packages
RUN amazon-linux-extras install epel -y && \
    yum update -y && \
    yum install -y nginx python3-pip amazon-cloudwatch-agent && \
    yum clean all && \
    rm -rf /var/cache/yum

# Install Supervisord using pip3
RUN pip3 install supervisor

# Copy Supervisord configuration file
COPY supervisord.conf /etc/supervisord.conf

# Copy CloudWatch agent configuration file
COPY config.json /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json

# Ensure NGINX log directory permissions
RUN chmod -R 644 /var/log/nginx

# Expose NGINX port
EXPOSE 80

# Set the command to run Supervisord
CMD ["/usr/local/bin/supervisord", "-n"]
```

</details>

#### 2. Supervisord Configuration
Manages NGINX and the CloudWatch agent, ensuring both run concurrently.


<details>
   <summary>Click to view the code</summary>


```
[supervisord]
nodaemon=true

[program:nginx]
command=nginx -g 'daemon off;'
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stderr_logfile=/dev/stderr

[program:cloudwatch-agent]
command=/opt/aws/amazon-cloudwatch-agent/bin/start-amazon-cloudwatch-agent
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stderr_logfile=/dev/stderr
```

</details>

#### 3. CloudWatch Agent Configuration
Uses `"resources": ["*"]` to monitor all disks and `${aws:InstanceId}` for unique identification.


<details>
   <summary>Click to view the code</summary>


```json
{
   "metrics": {
      "metrics_collected": {
         "disk": {
            "measurement": [
               "disk_used",
               "disk_free"
            ],
            "metrics_collection_interval": 10,
            "resources": [
               "*"
            ],
            "unit": "Megabytes"
         }
      },
      "append_dimensions": {
        "ContainerName": "nginx-cloudwatch-agent-container-${aws:InstanceId}",
        "InstanceId": "${aws:InstanceId}"
      }
   },
   "logs": {
      "logs_collected": {
         "files": {
            "collect_list": [
               {
                  "file_path": "/var/log/nginx/access.log",
                  "log_group_name": "NginxLogGroup-mallick",
                  "log_stream_name": "mallick-nginxagent-${aws:InstanceId}/access.log",
                  "timestamp_format": "[%d/%b/%Y:%H:%M:%S %z]"
               }
            ]
         }
      }
   }
}
```

</details>

---

### Instructions (Recap)

<details>
   <summary>Execution Commands</summary>
   
1. **Prepare Files**:
   - Create a directory (e.g., `super-nginx-watch`) on each EC2 instance.
   - Save the `Dockerfile`, `supervisord.conf`, and `config.json` above.
2. **Ensure Disk Space**:
   - Check: `df -h /` (need ~500 MB).
   - Free space: `docker system prune`, `sudo rm -rf /tmp/*`.
   - Increase EC2 root volume if needed: `sudo growpart /dev/nvme0n1 1`, `sudo resize2fs /dev/nvme0n1p1`.
3. **Set Up IAM Role**:
   - Attach an IAM role with `logs:*`, `cloudwatch:PutMetricData`, and `ec2:DescribeInstances` permissions (or use `CloudWatchAgentServerPolicy` with `ec2:DescribeInstances` added).
4. **Build Image**:
   ```bash
   docker build -t nginx-cloudwatch-agent .
   ```
5. **Run Containers**:
   - List EBS mount points:
     ```bash
     df -h | grep /mnt | awk '{print $6}' | while read -r mount; do echo "-v $mount:$mount"; done
     ```
   - Run with IAM role:
     ```bash
     docker run -d \
       -v /mnt/ebs1:/mnt/ebs1 \
       -v /mnt/ebs2:/mnt/ebs2 \
       -p 80:80 \
       --name nginx-container nginx-cloudwatch-agent
     ```
     Adjust `-v` flags for your mount points.
6. **Verify in CloudWatch**:
   - Generate logs: `curl http://localhost`.
   - Check logs in `NginxLogGroup-mallick` under streams like `mallick-nginxagent-i-1234567890abcdef0/access.log`.
   - Check metrics in `CWAgent` namespace, filtered by `InstanceId` or `ContainerName`, for `disk_used` and `disk_free` in MB.

</details>

---

### Why Supervisord Was Necessary
Without Supervisord, managing NGINX and the CloudWatch agent in a single container would require complex scripting or multiple containers, which would:
- Increase setup complexity (e.g., custom scripts for process monitoring).
- Risk process failures without automatic restarts.
- Complicate log aggregation and debugging.
- Violate the single-container requirement for your trial test.

Supervisord’s simplicity, reliability, and compatibility made it the ideal choice for your use case, ensuring a robust and maintainable solution for monitoring NGINX logs and EBS disk metrics across multiple EC2 instances.

---

# Different Versions/Improvement of Deployment
## Folder name which is present in this repo (_CWAgent-V1_)
Follow the same execution steps and File Structure. Check below to view the images for the cloudwatch based on the files and I have mentioned the Folder name here **_CWAgent-V1_** which is present in this repo.

<details>
   <summary>Click to view the changes happened in the CloudWatch based on the scripts and Configurations</summary>

---

![image](https://github.com/user-attachments/assets/a859ef15-f094-401d-a499-1113563ce4c7)

---
   
</details>

---

# Different Versions/Improvement of Deployment
## Folder name which is present in this repo (_CWAgent-V2_)
Follow the same execution steps and File Structure. Check below to view the images for the cloudwatch based on the files and I have mentioned the Folder name here **_CWAgent-V2_** which is present in this repo.

<details>
   <summary>Click to view the changes happened in the CloudWatch based on the scripts and Configurations</summary>

---

![image](https://github.com/user-attachments/assets/569f7aca-9f4b-4cd6-9aef-eb7d19cd65e9)

---
   
</details>

---

# Different Versions/Improvement of Deployment
## Folder name which is present in this repo (_CWAgent-V3_)
Follow the same execution steps and File Structure. Check below to view the images for the cloudwatch based on the files and I have mentioned the Folder name here **_CWAgent-V3_** which is present in this repo.

<details>
   <summary>Click to view the changes happened in the CloudWatch based on the scripts and Configurations</summary>

---

![image](https://github.com/user-attachments/assets/8f95d759-1d01-4072-93d4-93934c8f2ade)

---
   
</details>

---

# Different Versions/Improvement of Deployment
## Folder name which is present in this repo (_CWAgent-V4_)
Follow the same execution steps and File Structure. Check below to view the images for the cloudwatch based on the files and I have mentioned the Folder name here **_CWAgent-V4_** which is present in this repo.

<details>
   <summary>Click to view the changes happened in the CloudWatch based on the scripts and Configurations</summary>

---

![image](https://github.com/user-attachments/assets/80ff5093-fe82-4993-a0ff-d0a21110d831)

---

</details>

---

# Different Versions/Improvement of Deployment
- We are running multiple instances and using the same configuration file and checking the metrices through console, the difference between all the versions and this version is
  - We are combining both the instances disk space used and disk space free in single metrices using **_aggregation_dimension_**
## Folder name which is present in this repo (_CWAgent-V5_)
Follow the same execution steps and File Structure. 
- Copy and paste the files in 2 or 3 instances.
- Follow the execution steps as mentioned in all the instances.
- Refresh the webpage and Check the Cloudwatch Console(Log Groups and Metrics) for the changes happened through CloudWatch Agent.

Check below to view the images for the cloudwatch based on the files and I have mentioned the Folder name here **_CWAgent-V5_** which is present in this repo.

<details>
   <summary>Click to view the changes happened in the CloudWatch based on the scripts and Configurations</summary>

---

![image](https://github.com/user-attachments/assets/2928cf9b-500f-4a53-8562-0f712cc82740)

---

</details>

---
