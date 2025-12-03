# Guide: Publishing to Azure Marketplace as a Solution Template

This guide outlines the technical steps to package your Dify-based application as a Solution Template for the Azure Marketplace. This approach allows a customer to deploy your entire solution into their Azure subscription with a few clicks.

The core idea is to use an ARM (Azure Resource Manager) template to define all the resources and configurations needed.

## Architecture Overview

1.  **Backend**: The Dify Docker stack will run on an Azure Virtual Machine (VM).
2.  **Frontend**: The static `v1.html` and image assets will be hosted on an Azure Static Web App, which is a cost-effective way to serve static content.
3.  **Deployment**: A customer clicks "Get It Now" in the Marketplace, which launches a deployment in their Azure portal. The ARM template prompts them for parameters (like VM size), then creates the VM and Static Web App, and configures them to work together.

---

## Step 1: Prepare the Backend for Automated Deployment

Your Dify backend needs to be set up automatically on the VM when it's created. The best way to do this is with a `cloud-init` script.

Create a file named `cloud-init.yaml`. This script will be fed to the VM on its first boot.

```yaml
#cloud-config
package_update: true
package_upgrade: true
packages:
  - apt-transport-https
  - ca-certificates
  - curl
  - gnupg
  - lsb-release
  - docker.io
  - docker-compose
runcmd:
  # Start Docker
  - systemctl start docker
  - systemctl enable docker

  # Create a directory for Dify and download your docker-compose.yml
  - mkdir -p /opt/dify
  - cd /opt/dify
  # NOTE: You must host your docker-compose.yml file somewhere publicly accessible, like a GitHub Gist or Azure Blob Storage.
  # Replace the URL below with the actual URL to your file.
  - curl -L "https://raw.githubusercontent.com/langgenius/dify/main/docker/docker-compose.yml" -o docker-compose.yml
  
  # Pull the images and start the containers in detached mode
  - docker-compose pull
  - docker-compose up -d

  # (Optional but Recommended) Configure a reverse proxy like Caddy or Nginx
  # This would handle HTTPS for your Dify instance. You would also need to
  # configure DNS for the VM's public IP.
```

**Action**: You will need to make your `docker-compose.yml` file available at a public URL for this script to work.

---

## Step 2: Parameterize the Frontend

The `iframe` in `v1.html` currently points to `localhost`. This must be dynamic. The frontend needs to know the public address of the backend VM that gets created.

I have already modified `v1.html` to read the backend URL from a query parameter. This allows the ARM template to construct the correct URL and pass it to the Static Web App.

**New URL Structure**: `https://<your-static-web-app-url>/?backend_url=<your-vm-public-dns-name>`

---

## Step 3: Create the ARM Template

The ARM template is a JSON file that declares all the resources to be deployed. This is the core of your Marketplace offer.

Create a file named `mainTemplate.json`.

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "vmAdminUsername": {
      "type": "string",
      "metadata": { "description": "Admin username for the Virtual Machine." }
    },
    "vmAdminPassword": {
      "type": "securestring",
      "metadata": { "description": "Admin password for the Virtual Machine." }
    }
  },
  "variables": {
    "publicIpAddressName": "[concat('dify-ip-', uniqueString(resourceGroup().id))]",
    "dnsLabelPrefix": "[concat('dify-backend-', uniqueString(resourceGroup().id))]",
    "vmName": "dify-backend-vm"
    // Other variables for network, etc.
  },
  "resources": [
    // 1. Public IP Address for the VM
    {
      "type": "Microsoft.Network/publicIPAddresses",
      "apiVersion": "2020-11-01",
      "name": "[variables('publicIpAddressName')]",
      "location": "[resourceGroup().location]",
      "properties": {
        "publicIPAllocationMethod": "Dynamic",
        "dnsSettings": {
          "domainNameLabel": "[variables('dnsLabelPrefix')]"
        }
      }
    },
    // 2. Network Security Group (to open port 80/443)
    // 3. Virtual Network & Subnet
    // 4. Network Interface for the VM
    
    // 5. The Virtual Machine itself
    {
        "type": "Microsoft.Compute/virtualMachines",
        "apiVersion": "2021-03-01",
        "name": "[variables('vmName')]",
        "location": "[resourceGroup().location]",
        "dependsOn": [
            // Dependency on Network Interface
        ],
        "properties": {
            // ... VM properties (size, image, etc.)
            "osProfile": {
                "computerName": "[variables('vmName')]",
                "adminUsername": "[parameters('vmAdminUsername')]",
                "adminPassword": "[parameters('vmAdminPassword')]",
                "customData": "[base64(file('./cloud-init.yaml'))]" // This is key!
            }
            // ... network profile linking to NIC
        }
    },
    // 6. Static Web App for the Frontend
    {
        "type": "Microsoft.Web/staticSites",
        "apiVersion": "2021-02-01",
        "name": "dify-frontend-app",
        "location": "[resourceGroup().location]",
        "sku": { "name": "Free" },
        "properties": {
            // The frontend code (v1.html, logo/) must be deployed to the
            // Static Web App. This is usually done via a deployment token
            // and a CI/CD pipeline (e.g., GitHub Actions) or by zipping
            // the content and including it in the Marketplace package.
        }
    }
  ],
  "outputs": {
      "frontendUrl": {
          "type": "string",
          "value": "[concat('https://', reference('dify-frontend-app').defaultHostname, '/?backend_url=http://', variables('dnsLabelPrefix'), '.', resourceGroup().location, '.cloudapp.azure.com/chatbot/UXOZLHzoZGfiSVsI')]"
      }
  }
}
```

**Note**: The above ARM template is a simplified skeleton. You will need to fill in the details for networking and other properties.

---

## Step 4: Package and Publish

1.  **Create a `createUiDefinition.json` file**: This file defines the user interface for the deployment in the Azure portal (e.g., the text boxes for the VM username and password).
2.  **Zip the Package**: Create a `.zip` file containing `mainTemplate.json`, `createUiDefinition.json`, and any scripts like `cloud-init.yaml`. Your frontend code can also be included.
3.  **Go to Azure Partner Center**:
    *   Navigate to [partner.microsoft.com](https://partner.microsoft.com).
    *   Go to the Commercial Marketplace and create a new offer of type "Azure Application".
    *   Choose the "Solution Template" plan type.
    *   Fill out all the required marketing, legal, and support information.
    *   Upload your `.zip` package in the "Technical Configuration" section.
    *   Submit for validation and publication.

This is a high-level overview, but it provides the complete technical path from your current code to a publishable Azure Marketplace offer.
