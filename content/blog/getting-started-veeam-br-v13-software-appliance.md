---
title: "Getting Started with Veeam Backup & Replication v13: The Software Appliance"
date: 2026-05-04T18:00:00-07:00
draft: false
categories: ["Veeam", "Backup & Replication"]
tags: ["Veeam BR v13", "Software Appliance", "Installation", "Configuration"]
---

# Getting Started with Veeam Backup & Replication v13: The Software Appliance

Veeam Backup & Replication (BR) v13 introduces several enhancements, and one of the most convenient ways to deploy it is via the **Veeam Software Appliance**. This pre-configured, hardened Linux-based virtual machine includes Veeam BR v13, allowing you to get up and running quickly without worrying about underlying OS patching or complex prerequisites.

In this guide, we'll walk through downloading, deploying, and configuring the Veeam Software Appliance for your first backup job.

## Why Choose the Software Appliance?

- **Speed to Value**: No need to install Windows Server, apply patches, or configure .NET prerequisites.
- **Hardened OS**: Based on a minimal, secure Linux distribution optimized for Veeam.
- **Simplified Management**: Veeam handles the underlying OS updates via appliance updates.
- **Consistent Environment**: Eliminates variability from different Windows versions or configurations.

## Prerequisites

Before you begin, ensure you have:

1. **A hypervisor** capable of running the appliance (VMware ESXi, Microsoft Hyper-V, Nutanix AHV, or KVM).
2. **Enough resources**: The appliance typically requires:
   - 4 vCPUs
   - 8 GB RAM (minimum; 16 GB+ recommended for production workloads)
   - 50 GB disk space for the appliance OS + Veeam installation (plus additional storage for backups)
3. **Network connectivity**: The appliance needs access to:
   - Your production workloads (for backup)
   - Backup repositories (where backups will be stored)
   - The internet (for initial activation and updates)
   - DNS and NTP servers
4. **A Veeam license**: You can start with the free edition (limited to 10 instances) or apply a paid license.

## Step 1: Download the Veeam Software Appliance

1. Go to [Veeam Downloads](https://www.veeam.com/downloads.html) (requires free registration).
2. Navigate to **Backup & Replication** → **Veeam Backup & Replication v13**.
3. Look for the **"Veeam Backup & Replication v13 Software Appliance"** section.
4. Choose the format matching your hypervisor:
   - **OVA** (for VMware ESXi, Hyper-V, VirtualBox, etc.)
   - **AVHDX** (for Hyper-V)
   - **QCOW2** (for KVM, Nutanix AHV)
   - **AVHD** (older Hyper-V)
5. Download the file to your management workstation.

## Step 2: Deploy the Appliance

### VMware ESXi Example

1. In the vSphere Client, right-click your host or cluster → **Deploy OVF Template**.
2. Select the downloaded OVA file.
3. Follow the wizard:
   - **Name**: Give it a meaningful name (e.g., `VeeamBRv13-Appliance`)
   - **Location**: Choose a folder or resource pool.
   - **Storage**: Select a datastore with enough space (thin provisioning is fine).
   - **Network**: Map the appliance's network interfaces to your port groups. Typically, you'll have:
     - **Management Network**: For Veeam console access and updates.
     - **Backup Network** (optional but recommended): For backup traffic to avoid saturating your management network.
4. **Customize Template**: You may be prompted for:
   - Root password (set a strong password; you'll need this for initial console login).
   - Network settings (IP address, DNS, gateway) – you can also configure via console later.
5. Finish deployment and power on the VM.

### Hyper-V Example

1. In Hyper-V Manager, choose **Import Virtual Machine**.
2. Point to the folder containing the extracted AVHDX files (or use the import wizard for the VHD/X).
3. Choose the appropriate settings (generation, memory, etc.).
4. Complete the import and start the VM.

## Step 3: Initial Configuration via Console

Once the appliance VM is running, you can access its console to perform initial setup:

1. **Access the Console**: Use vSphere Console, Hyper-V Connect, or your hypervisor's console viewer.
2. **Login**: Use the root credentials you set during deployment (or the default if you didn't change it – check the deployment notes).
3. **Network Configuration**:
   - The appliance may come configured for DHCP by default. If you need a static IP:
     ```bash
     sudo nmtui
     ```
     (This launches a text-based NetworkManager interface to configure IP, DNS, gateway.)
   - Alternatively, edit `/etc/sysconfig/network-scripts/ifcfg-eth0` (or similar) and restart networking.
4. **Set Hostname** (optional but recommended):
   ```bash
   sudo hostnamectl set-hostname veeam-br-appliance
   ```
   Then edit `/etc/hosts` to include the new hostname.
5. **Verify Connectivity**:
   - Ensure you can ping gateways, DNS servers, and your Veeam license server (if applicable).
   - Confirm the appliance can reach the internet (for updates and activation): `curl -I https://www.veeam.com`

## Step 4: Access the Veeam Backup & Replication Console

The appliance includes a pre-configured Veeam BR v13 instance. To manage it:

1. From a management workstation on the same network (or via routed access), open a browser.
2. Navigate to: `https://<appliance-ip-address>:9398`
   - The default port for the Veeam Enterprise Manager (web interface) is 9398.
   - You may also use the Veeam Backup & Replication console if you prefer the thick client (download from the appliance or Veeam site).
3. **Accept the Certificate Warning**: The appliance uses a self-signed certificate by default. Click through or import the certificate into your trusted store.
4. **Login**:
   - Use the `root` password you set earlier (or the default if unchanged).
   - The first login will prompt you to change the password – do so immediately.

## Step 5: Initial Veeam Configuration

Once logged into the Veeam console, run through the initial setup wizard:

### 1. Licensing

- Go to **Licensing** → **License Manager**.
- Click **Add** → enter your license file or select **Free Edition** (limited to 10 instances: VMs, physical servers, workstations, or cloud instances).
- Apply the license.

### 2. Adding Backup Infrastructure

Before you can create backup jobs, you need to define:

#### A. Backup Repository

This is where your backups will be stored.

1. Go to **Backup Infrastructure** → **Backup Repositories** → **Add Repository**.
2. Choose **Microsoft Windows** or **Linux** depending on where your storage is hosted.
   - For simplicity, you can add a **Direct-attached** repository on the appliance itself (though not recommended for production due to limited space).
   - Better: Point to an existing SMB share, NFS export, or dedicated storage server.
3. Follow the wizard to specify the path, and optionally enable features like **block clustering** (for ReFS) or **storage integration**.

#### B. Configure Backup Proxies (Optional but Recommended)

The appliance includes a built-in proxy, but you can add more for scalability.

1. Go to **Backup Infrastructure** → **Backup Proxies** → **Add Proxy**.
2. The appliance itself will appear as an available host. You can use it as-is for small environments.
   - For larger deployments, consider adding separate proxy servers to offload processing from the appliance.

#### C. Add vCenter/ESXi Hosts (for VM backups)

1. Go to **Backup Infrastructure** → **Managed Servers** → **Add Server** → **VMware vSphere**.
2. Enter your vCenter Server or ESXi host credentials.
3. Veeam will automatically discover the inventory.

### 3. Configure Backup Jobs

Now you're ready to create your first backup job:

1. Go to **Backup Jobs** → **Add Job** → **Virtual machine backup**.
2. **Name**: Give it a descriptive name (e.g., `Daily-VM-Backup`).
3. **Virtual Machines**: Click **Add** and select the VMs you want to protect.
4. **Storage**: Choose the backup repository you created earlier.
5. **Schedule**: Set your desired backup frequency (e.g., daily at 2 AM).
6. **Retention**: Define how many restore points to keep (e.g., keep 14 daily backups, 4 weekly, 12 monthly).
7. **Advanced Settings**: Consider enabling:
   - **Application-aware processing** (for transactionally consistent backups of SQL, Exchange, AD, etc.)
   - **Encryption** (if your repository doesn't already provide it)
   - **Compression** and **deduplication** (optimize storage)
8. Save the job.

### 4. Run Your First Backup

- Right-click the job → **Start**.
- Monitor progress from the **Home** view or the **Jobs** ribbon.
- Once complete, verify the restore points in the **Backups** node under your repository.

## Step 6: Keeping the Appliance Updated

Veeam periodically releases updates for the Software Appliance that include both OS security patches and Veeam BR version updates.

1. In the Veeam console, go to **Help** → **Check for Updates**.
2. If an appliance update is available, you’ll be prompted to download and apply it.
   - The update process will reboot the appliance and apply the new version.
   - Plan for a brief maintenance window.

Alternatively, you can check the Veeam download portal for newer appliance versions and redeploy if needed.

## Step 7: Next Steps & Best Practices

With your appliance up and running, consider these follow-up actions:

- **Backup Monitoring**: Set up email notifications for job success/failure (**Options** → **Email**).
- **Backup Copy Jobs**: Create copies of your backups to an offsite repository for 3-2-1 rule compliance.
- **Replication**: Use Veeam Replication to create ready-to-run VM copies at a DR site.
- **SureBackup**: Automatically verify that your backups are recoverable.
- **Security**: 
  - Change the default root password if you haven’t already.
  - Consider disabling direct root SSH login and using sudo with a specific user.
  - Restrict management access to specific IPs via firewall rules (if your hypervisor supports it).
- **Documentation**: Keep a record of your appliance IP, credentials, and configuration for disaster recovery.

## Conclusion

The Veeam Software Appliance for BR v13 is an excellent choice for getting started quickly, especially in lab environments, small businesses, or as a quick proof-of-concept. It removes the complexity of OS installation and patching, letting you focus on configuring your backup infrastructure and protecting your data.

By following the steps above, you should have a fully functional Veeam BR v13 environment ready to run backup jobs in under an hour (excluding the time to download the OVA).

**Next Tutorial Idea**: Explore how to scale beyond the appliance by adding dedicated proxy servers, scale-out repositories, or integrating with cloud storage like AWS S3 or Azure Blob for long-term retention.

Happy backing up! 🚀

---

*This post is part of the InfoTechAnswers series aimed at helping IT professionals gain hands-on experience with Veeam products—perfect for building knowledge toward the Veeam MVP program.*
