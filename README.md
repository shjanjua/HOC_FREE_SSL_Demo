# Free SSL Certificate with Automatic Renewal & Deployment

This repository contains the Terraform code and supporting resources for setting up free SSL certificates on an Oracle Cloud Infrastructure (OCI) Load Balancer (LB) using Let's Encrypt, Certbot, and the OCI CLI.  This setup includes automated certificate renewal.

**This repository supports the blog article on configuring free SSL certificates on OCI Load Balancer with automatic renewal:** [https://highoncloud.co.uk/oci-lb-free-ssl-cert/]

## Overview

This setup automates the process of obtaining and deploying SSL certificates from Let's Encrypt and automatically renews them on your OCI Load Balancer. The solution leverages Terraform for infrastructure provisioning, Certbot for certificate management, and the OCI CLI for certificate deployment.

## Architecture

![image](https://github.com/user-attachments/assets/59b17ee4-6753-4547-9010-6add282eb715)

The solution deploys the following components:

*   **OCI Load Balancer:**  Listens on ports 80 and 443. Port 443 handles secure TLS traffic from users, while port 80 is used for the ACME HTTP challenge during certificate creation and renewal.
*   **Certbot VM:** A compute instance running Certbot, responsible for obtaining and renewing certificates from Let's Encrypt.
*   **Web Server VM:**  A compute instance hosting a web server.  For simplicity, this example uses a basic Python HTTP server.
*   **OCI Bastion Service:** Provides secure access to the private VMs (Certbot and Web Server).
*   **OCI Service Gateway:**  Provides network access to Oracle Services required by the Compute Agent plugins.
*   **OCI Network Resources:** Virtual Cloud Network (VCN), subnets, route tables, and network security groups (NSGs) to support the infrastructure.

The high-level flow is:

![image](https://github.com/user-attachments/assets/0af798e9-2cd7-4617-a102-544b0c651db7)


1.  Certbot VM stands up a webserver on port 80 for the ACME challenge.
2.  Certbot requests a certificate from Let's Encrypt.
3.  Let's Encrypt validates domain ownership via the ACME HTTP challenge, accessing the Certbot VM through the Load Balancer.
4.  Upon successful validation, Let's Encrypt issues a new certificate.
5.  Certbot triggers a custom script (defined using `--deploy-hook`).
6.  The custom script uses the OCI CLI to upload the new certificate to the Load Balancer and updates the Load Balancer listener to use the new certificate.

## Prerequisites

*   An Oracle Cloud Infrastructure tenancy with access to Always Free services.
*   A domain name with a DNS entry pointing to the public IP address of the OCI Load Balancer.  You can obtain a free domain from services like NoIP.com.
*   Terraform installed on your local machine or in a Cloud Shell environment.
*   OCI CLI installed and configured in the Certbot VM (see Installation and Configuration section below).

## Infrastructure Setup

The infrastructure is provisioned using Terraform.  The Terraform code in this repository defines the following resources:

*   Compartment: a logical partition in the OCI tenancy
*   VCN: The Virtual Cloud Network (VCN) provides a private network for your infrastructure
*   Subnets:  Dividing the VCN into public and private subnets. The Load Balancer resides in a public subnet, while the Certbot and Web Server VMs reside in a private subnet.
*   Route Tables:  Defining network routes, including routes to the Internet Gateway and Service Gateway.
*   Security Lists/NSGs:  Controlling network traffic in and out of the subnets and VMs.
*   Compute Instances: The Certbot and web server instances
*   Load Balancer:  The OCI Load Balancer handles incoming traffic and distributes it to the backend VMs.
*   Bastion Service:  Provides secure access to the VMs in the private subnet.

## Getting Started

1.  **Clone the Repository:**

    ```bash
    git clone https://github.com/ShahvaizJanjua/HOC_FREE_SSL_Demo.git
    cd HOC_FREE_SSL_Demo
    ```

2.  **Configure Terraform Variables:**

    Create a `terraform.tfvars` file (or pass the variables via environment variables) with the following variables:

    ```terraform
    tenancy_ocid             = "ocid1.tenancy.oc1..aaaaaaa..."
    user_ocid                = "ocid1.user.oc1..aaaaaaa..."
    fingerprint              = "xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx"
    private_api_key_path     = "~/.oci/oci_api_key.pem"
    region                   = "us-ashburn-1"  # Example: us-ashburn-1
    rootCompartment           = "ocid1.compartment.oc1..aaaaaaa..."
    public_key_path          = "~/.ssh/id_rsa.pub" # Path to your SSH public key
    private_key_path         = "~/.ssh/id_rsa"  # Path to your SSH private key
    ```

    **Important:** Replace the placeholder values with your actual OCI credentials and SSH key paths.  Ensure your SSH key pair is created *before* running terraform.

3.  **Initialize Terraform:**

    ```bash
    terraform init
    ```

4.  **Plan the Infrastructure:**

    ```bash
    terraform plan
    ```

    Review the plan to ensure the changes are as expected.

5.  **Apply the Infrastructure:**

    ```bash
    terraform apply
    ```

    Type `yes` to confirm the deployment.

## Installation & Configuration

Once the Terraform deployment is complete, you need to configure Certbot and the OCI CLI on the Certbot VM.

1.  **Connect to the Certbot VM:**

    The Terraform output will provide the SSH command to connect to the Certbot VM via the Bastion service.  Copy and execute the command.  It should look something like:

    ```
    ssh -i <private_key_path> -o ProxyCommand='ssh -W %h:%p -p 22 opc@<bastion_public_ip>' opc@<certbot_private_ip>
    ```

2.  **Install Certbot:**

    Follow these steps on the Certbot VM:

    ```bash
    sudo dnf config-manager --enable ol8_developer_EPEL
    sudo dnf install snapd -y
    sudo setenforce 0
    sudo systemctl stop firewalld
    sudo ln -s /var/lib/snapd/snap /snap
    sudo systemctl enable snapd
    sudo systemctl start snapd
    sudo snap install --classic certbot
    sudo ln -s /snap/bin/certbot /usr/bin/certbot
    ```

3.  **Install and Configure OCI CLI:**

    ```bash
    sudo dnf install python36-oci-cli -y
    oci setup config
    ```

    Configure the OCI CLI using your OCI credentials. It's recommended to run `oci setup config` as both the `opc` user and `root`, or copy the `/home/opc/.oci/config` file to `/root/.oci/config`. This ensures that both the user and the root user (required for Certbot renewal hooks) can access the OCI CLI.

4.  **Create the Deployment Script:**

    Create a script (e.g., `lbcertupload.sh`) in the `/home/opc/` directory with the following content:

    ```bash
    #!/bin/bash

    oci lb certificate create \
    --load-balancer-id <your_load_balancer_ocid> \
    --wait-for-state SUCCEEDED \
    --certificate-name "$RENEWED_DOMAINS-"`date +"%Y-%m-%d"` \
    --ca-certificate-file "$RENEWED_LINEAGE/fullchain.pem" \
    --private-key-file "$RENEWED_LINEAGE/privkey.pem" \
    --public-certificate-file "$RENEWED_LINEAGE/cert.pem"

    oci lb listener update --force \
    --load-balancer-id <your_load_balancer_ocid> \
    --wait-for-state SUCCEEDED \
    --listener-name "Web_Listener" \
    --default-backend-set-name "WebBackendSet" \
    --port 443 --protocol HTTP2 --cipher-suite-name oci-default-http2-ssl-cipher-suite-v1 \
    --ssl-certificate-name "$RENEWED_DOMAINS-"`date +"%Y-%m-%d"`
    ```

    **Important:** Replace `<your_load_balancer_ocid>` with the actual OCID of your Load Balancer. Make the script executable: `chmod +x /home/opc/lbcertupload.sh`.  Also ensure the `Web_Listener` listener name is correct for your load balancer, if you changed it from the terraform default.

5. **Obtain and deploy the certificate**
```bash
sudo certbot certonly --standalone --deploy-hook /home/opc/lbcertupload.sh

Automatic Renewal

Certbot automatically configures a systemd timer to renew certificates. You can verify this by running:

sudo systemctl list-timers
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

The --deploy-hook specified during certificate creation ensures that the lbcertupload.sh script is executed every time a certificate is successfully renewed.

Verification

After successfully obtaining the certificate:

Access your domain (e.g., https://yourdomain.com) in a web browser.

Verify that the connection is secure and the certificate is valid.

Test the certificate renewal process by simulating a renewal: sudo certbot renew --dry-run. Verify the lbcertupload.sh script executed without any errors

Troubleshooting

Load Balancer Health Checks: The Load Balancer health check for the Certbot VM uses port 22 (SSH). This is because the web service on port 80 is only active during the ACME challenge. Using port 22 ensures that the Load Balancer always considers the backend healthy.

Bastion Plugin: Ensure the Bastion plugin is enabled on the Compute instance. The provided Terraform code automatically enables this plugin.

Network Security Groups: Review the NSG rules to ensure that traffic is allowed on the necessary ports (80, 443, 22, and 8080).

OCI CLI Configuration: Verify that the OCI CLI is correctly configured and can access your OCI resources.
