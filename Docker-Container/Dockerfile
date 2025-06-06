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
