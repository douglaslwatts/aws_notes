# Create a Custom AWS AMI: [AWS AMI VM Import Docs](https://docs.aws.amazon.com/pdfs/vm-import/latest/userguide/vm-import-ug.pdf)

## Creating a Fedora Server AMI

### Spin Up a Virtual Box

1. Download a Fedora Server ISO from [www.fedoraproject.org/server/download](https://www.fedoraproject.org/server/download)
2. Spin up a VirtualBox instance using the ISO
    1. Choose **New** to start a new instance
    2. Choose **Expert Mode** (so you can choose VMDK later)
    3. Name and Operating System
        - Enter a name for the instance
        - Choose the downloaded Fedora Server ISO from the filesystem
    4. Hardware
        - Choose memory and CPU allocations (These make no difference in the AMI as they are
          determined by the instance type when an EC2 instance is created).
    5. Hard Disk
        - Choose a disk size (This will determine the volume size attached when an EC2
          instance is created).
        - Choose **VMDK (Virtual Machine Disk)** as disk type
        - Select **Pre-allocate Full Size**.
    6. Click **Finish**
    7. With the new box selected, click **Start**

3. Install the OS
    - Choose a language
    - Click into **User Creation** and add a sudo user
        - Enter a name and username
        - Select **Add administrative privileges to this user account (wheel group membership)**
        - Select **Require a password to use this account**
        - Enter the user password
        - Click **Done**
    - Leave the root account disabled
    - Click into Installation Destination
        - Select **Custom**
        - Click **Done**
        - Manual Partitioning
            1. Select **Standard partition** partitioning scheme
            2. Click **Click here to create them automatically**
            3. Click the **+** to add a new mount point
            4. Select **swap**
            5. Enter a size for the swap (Generally twice the RAM size is recommended)
            6. Click **Add mount point**
            7. Click the **+** to add a new mount point
            8. Select **/home**
            9. Enter the size amount specified in the **Available Space** to use the remaining space
            10. Click **Add mount point**
            11. Under **File System**, select **BIOS Boot** for **BIOS Boot** (likely `sda1`)
            12. Under **File System**, select **swap** for `swap`
            13. Under **File System**, select either **EXT2**, **EXT3**, **EXT4**, or **XFS** for all
                other mount points depending on your needs
            14. Click **Done**
            15. Click **Accept Changes** after reviewing them
    - Click **Begin installation**
    - Once installation has completed, click **Reboot System**
4. With the new box selected, click **Network** to open the network settings
5. Click **Advanced**
6. Click **Port Forwarding**
7. Click the **+** and add the following entry to easily `ssh` onto the running box locally

   | Name | Protocol | Host IP | Host Port | Guest IP | Guest Port |
   |:-|:-|:-|:-|:-|:-|
   |ssh|TCP||8000||22|

8. Click **Ok** to close the Port Forwarding dialog
9. Click **Ok** to close the settings
10. In a terminal `ssh -p 8000 username@localhost`
11. Make an SSH directory in the user home directory: `mkdir .ssh`
12. In another terminal

    `cat .ssh/id_rsa.pub | xargs -I % ssh -p 8000 username@localhost 'echo % >> .ssh/authorized_keys'`

    (or whatever your public key file is named) If you do not have key files setup use `ssh-keygen`
    to create an RSA key pair
13. Back in the `ssh` terminal, check the contents of `.ssh/authorized_keys` matches your public key file
14. `exit` the SSH session
15. SSH onto the server again, now using the RSA key, so no password prompting
16. Set up any configurations you prefer such as `.bashrc`, `.bash_aliases`, `.bash_profile`, environment
    variables, `.hushlogin`, `.vimrc`, etc. You can `scp` them from your local setup and make adjustments where needed.
17. Install any packages needed in the server. To create a minimal starting point for a web server, I did

    `sudo dnf upgrade -y`

    `sudo dnf install git-all tmux httpd the_silver_searcher bat -y`
18. The upgrade will have installed a new kernel most likely. At the time of writing this, the AMI
    VM Import docs say that the latest Fedora kernel supported is 6.5.6. So, we need to set that as
    the default kernel and remove the newly installed kernel.

    1. `sudo grubby --default-kernel`
    2. `ls /boot/vmlinuz-*` (take note of the kernel version needed. 6.5.6 at the time of writing this)
    3. `sudo grubby --set-default=/boot/vmlinuz-6.5.6-300.fc39.x86_64`
    4. `rpm -q kernel-core` (take note of the newer kernel to uninstall)
    5. `sudo dnf remove kernel-core-6.7.5-200.fc39.x86_64`

    Now the first command should output the needed 6.5.6 kernel, and the fourth command should show
    only that kernel-core package.
19. The image cannot use predictable network interface names, so they must be disabled.

    Follow step 6 from the [AWS Enable ENA Docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/enhanced-networking-ena.html#predictable-network-names-ena)

    You will see that `ip addr` will show something like `enp0s3` for ethernet

    `sudo sed -i '/^GRUB\_CMDLINE\_LINUX/s/\"$/\ net\.ifnames\=0\"/' /etc/default/grub`

    `sudo grub2-mkconfig -o /boot/grub2/grub.cfg`

    `sudo reboot`

    Then, SSH onto the server again and you should see that `ip addr` shows `eth0` for ethernet
20. Add the following two entries to `/etc/ssh/sshd_config` to disallow root login and password
    authentication, so only RSA key login authentication is allowed.

    `PermitRootLogin no`

    `PasswordAuthentication no`
21. Add the following entry to `/etc/httpd/conf/httpd.conf`

    `IncludeOptional sites-enabled/*.conf`
22. `sudo mkdir /etc/httpd/sites-available /etc/httpd/sites-enabled`
23. Enable `httpd.service`: `sudo systemctl enable --now httpd`
24. Restart `sshd.service`: `sudo systemctl restart sshd`
25. Name the server: `sudo vim /etc/hostname` and add some name for the server
26. Check the output of `sestatus` or `getenforce` to be sure the SELinux current mode is `enforcing`
27. The server is ready to export and upload: `sudo poweroff`

### Export the Image

1. In VirtualBox, click **file > Export Appliance**
2. Choose the box, and click **Next**
3. Choose the latest Open Virtualization Format (2.0 at the time of writing this)
4. Choose a location and filename for the export
5. Select **Include only NAT network adapter MAC addresses**
6. Select **Additionally: Write Manifest File**
7. Click **Next**
8. Click **Finish**
9. Wait until the export has completed

### Create an S3 Bucket To Hold the Exported Image File

1. Create an S3 bucket: `aws s3 mb s3://fedora-server-image`
2. Upload the image export file to the S3 bucket: `aws s3 cp /path/to/file s3://fedora-server-image`

### Create A Service Role

1. Be sure the AWS user, group, or role has the following permissions at a minimum

    ```
     {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Action": [
             "s3:GetBucketLocation",
             "s3:GetObject",
             "s3:PutObject"
           ],
           "Resource": [
              "arn:aws:s3:::fedora-server-image",
              "arn:aws:s3:::fedora-server-image/*"
            ]
         },
         {
           "Effect": "Allow",
           "Action": [
             "ec2:CancelConversionTask",
             "ec2:CancelExportTask",
             "ec2:CreateImage",
             "ec2:CreateInstanceExportTask",
             "ec2:CreateTags",
             "ec2:DescribeConversionTasks",
             "ec2:DescribeExportTasks",
             "ec2:DescribeExportImageTasks",
             "ec2:DescribeImages",
             "ec2:DescribeInstanceStatus",
             "ec2:DescribeInstances",
             "ec2:DescribeSnapshots",
             "ec2:DescribeTags",
             "ec2:ExportImage",
             "ec2:ImportInstance",
             "ec2:ImportVolume",
             "ec2:StartInstances",
             "ec2:StopInstances",
             "ec2:TerminateInstances",
             "ec2:ImportImage",
             "ec2:ImportSnapshot",
             "ec2:DescribeImportImageTasks",
             "ec2:DescribeImportSnapshotTasks",
             "ec2:CancelImportTask"
           ],
           "Resource": "*"
         }
       ]
     }
    ```
2. Create a file `trust-policy.json` with the following contents.

    ```
    {
       "Version": "2012-10-17",
       "Statement": [
          {
             "Effect": "Allow",
             "Principal": { "Service": "vmie.amazonaws.com" },
             "Action": "sts:AssumeRole",
             "Condition": {
                "StringEquals":{
                   "sts:Externalid": "vmimport"
                }
             }
          }
       ]
    }
    ```

3. Create the role

   `aws iam create-role --role-name vmimport --assume-role-policy-document "file://trust-policy.json"`

4. Create a file `role-policy.json` with the following contents




    ```
    {
       "Version":"2012-10-17",
       "Statement":[
          {
             "Effect": "Allow",
             "Action": [
                "s3:GetBucketLocation",
                "s3:GetObject",
                "s3:ListBucket"
             ],
             "Resource": [
                "arn:aws:s3:::fedora-server-image",
                "arn:aws:s3:::fedora-server-image/*"
             ]
          },
          {
             "Effect": "Allow",
             "Action": [
                "s3:GetBucketLocation",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:PutObject",
                "s3:GetBucketAcl"
             ],
             "Resource": [
                "arn:aws:s3:::fedora-server-image",
                "arn:aws:s3:::fedora-server-image/*"
             ]
          },
          {
             "Effect": "Allow",
             "Action": [
                "ec2:ModifySnapshotAttribute",
                "ec2:CopySnapshot",
                "ec2:RegisterImage",
                "ec2:Describe*"
             ],
             "Resource": "*"
          }
       ]
    }
    ```
5. Attach the policy to the role

   `aws iam put-role-policy --role-name vmimport --policy-name vmimport --policy-document "file://role-policy.json"`

### Import the Image to Create an AMI

1. Create a file `containers.json` with the following contents, where `s3key` is the name of the image export file

    ```
    [
        {
          "Description":"vm import",
          "Format":"ova",
          "UserBucket": {
            "S3Bucket": "fedora-server-image",
            "S3Key": "fedora-server.ova"
          }
        }
    ]
    ```
2. Import the image and create the AMI

   `aws ec2 import-image --description "Fedora Server" --disk-containers "file://containers.json" --platform Linux --boot-mode uefi`

   This will return an object containing an `ImportTaskId` that can be used to check the status of the job
3. Periodically check the status of the import job

   `aws ec2 describe-import-image-tasks --import-task-ids ImportTaskIdFromImportCommand`
4. Once the job has completed successfully, you can create instances from the new AMI!
