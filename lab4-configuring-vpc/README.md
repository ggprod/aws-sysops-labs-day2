# Lab 4: Configuring VPC
In this lab you will create a VPC in a region and then proceed to create some subnets, and internet gateway, route tables,
a bastion server, and a NAT gateway

## Task 1: Create a VPC
You'll begin by creating a VPC.  You could use the wizard to make it as simple as possible but here we'll do it with
a more manual approach for practice.

1. In **Services->VPC** click **Your VPCs** on the left, and then click on **Create VPC**.

2.  For **Name tag** give it the name firstname-lastname-vpc (replace firstname with your firstname in lowercase and lastname with your lastname in lowercase).  For **IPv4 CIDR block** put 10.0.0.0/16.  Then click **Create** and **Close**

3.  Click the checkbox next to the VPC you just created in the last (make sure it is the only one selected and deselect any other if they are also selected).  Click on the **Actions** button and click **Edit DNS hostnames**.  Click the checkbox next to **enable** and click
**Save** and then **Close**

This will assign friendly dns names to your instances when they are added to the vpc.  The names will have a format that looks like
ec2-52-42-133-255.us-west-2.compute.amazonaws.com where us-west-2 is the region that your VPC is in and the first portion matches the 
public IP address of your instance.  You could change this to a more meaningful name by using Route 53 (the AWS DNS service)

## Task 2: Create subnets
Now you'll add some subnets to the VPC.  One public and one private subnet.

1. In the left-hand pane click **Subnets**, then click **Create subnet**.  Give it the **Name tag** firstname-lastname-public and select your VPC in the **VPC** dropdown selector.  Select the first **Availability Zone** in the dropdown selector list and set the **IPv4 CIDR block** to 10.0.0.0/24.  Then click **Create** and **Close**

Note: you have to select a CIDR range for the subnet that is inside the CIDR range of the VPC it is in.  and 10.0.0.0/24 is within 10.0.0.0/16.

2. Select the subnet you just created via the checkbox to the left of it in the list and click **Actions** and **Modify auto-assign IP settings**.  Click the checkbox next to **Auto-assign IPv4** and click **Save**.

This will automatically assign a public IPv4 address to all EC2 instances as they are added to the subnet.  However this subnet is not truly public yet until we add an Internet Gateway to connect it to the internet (which you'll do in the next task).

3. Repeat the steps again to make a private subnet named firstname-lastname-private with IP range 10.0.2.0/23 in the first availability zone in the list

Note this range is also within the VPC range and it is twice as large as the public range.  This is typical as most of your resources shoudl be in private subnets and so they need larger IP spaces to be able to assign IP addresses to larger numbers of resources.

## Task 3: Create an internet gateway
Now you will add an internet gateway to add internet connectivity to VPC and then you'll set up that internet gateway to provide the connectivity to the public subnet your created.  An internet gateway is a horizontally scaled redundant high capacity service.  It 
does not place any availability risk (due to failure) or bandwith limits on traffic between the internet and your VPC.

The internet gateway provides a target for your route tables to connect to the internet and also provides the NAT (network address translation) for instances with public IP addresses.

1. In the left-hand pane click **Internet gateways** then click **Create internet gateway**.  Give it the name firstname-lastname-igw then click **Create** and **Close**

2. Select the newly create internet gatway (make sure its the only one selected) and click **Action** then **Attach to VPC**. Select your VPC and click **Attach**

Your VPC is now connected to the internet but you will need to update the route table for your public subnet to use it to connect to the internet.

## Task 4: Configure route tables
A route table includes a set of routes which are rules for where to send traffic which destination IP addresses that match the routes.  Each subnet must be connected to a subnet which controls where traffic from that subnet can be routed.  A subnet can only be associated with one route table at a time, but multiple subnets can be associated with the same route table.

For a subnet to use the internet gateway it needs to have a route in its route table that points internet-bound traffic at the internet gateway.  Subnets that have such routes are known as *Public subnets*

1. In the left-hand pane click **Route Tables**.  In the list look for the one that is currently in use for your vpc.  You can tell by looking at the **Summary** tab below and you'll see the **VPC** field will have the name of your VPC in it.  If you click on the **Routes** tab you will see it has a single route and the **Target** is local (so this is a private route table that does not connect to the internet).

2. Click on the pencil icon in the **Name** column for this route table and give it the **Name** firstname-lastname-private-rt

3. Click **Create route table** and set **Name tag** to firstname-lastname-public-rt and **VPC** to your create VPC.  Click **Create** and **Close**

4. Select the newly created route table in the list via the checkbox to the left (make sure it is the only one selected).  Click on the **Routes** tab, and click the **Edit routes** button and **Add route** button.  Put 0.0.0.0/0 in **Destination** and the select **Internet gateway** in the **Target** selector and then click on your internet gateway, and then click **Save routes** and **Close**

Now we have to associate this new public route table with the public subnet

5. Click the **Subnet associations** tab and click the **Edit subnet associations** button, select your public subnet via the checkbox to the left of it, and click **Save**

Now your public subnet is truly public as it can send and receive traffic into the internet.  To summarize, to make a subnet public requires creating (or already having) an internet gateway in your VPC, creating a route table (or adding a route to an existing route table) that points 0.0.0.0/0 destinatation traffic to that internet gateway, and associating that route table with the subnet which you want to be public (which then becomes public)

## Task 5: Create a Bastion host and NAT gateway in your public subnet
Bastion hosts (also known as jump boxes) serve as secure entry points to your system (from the internet) that can then be used to securely connect to your resources in your private subnets (which could not be directly connected to from the internet).

1. In **Services->EC2** click the **Launch instance** button, choose the top AMI in the list (Amazon Linux 2), choose t2.micro, select your created VPC and the public subnet of that VPC, click Next until the **Add tags** page and add a tag with key set to Name and value set to firstname-lastname-bastion, leave the security groups set as is and click **Review and Launch**.  Create a key pair (firstname-lastname-bastion-key) and save it locally and then **Launch instance**

Resource in the private subnet can't make requests directly to the internet, but to allow them to be able to we can create a NAT gateway which will forward their requests into the internet (and then pass response back to the instances in the private subnet).  This gives your private subnet instances the ability to make requests out to the internet without anyone in the internet to be able to make requests to your instances in your private subnet

2. In **Services->VPC** in the left-hand pane click **NAT gateways**, click **Create NAT gateway**, select your public subnet, click **Create New EIP** then click **Create NAT gateway**, then click **Edit route tables**, select your private subnet, click on the **Routes** tab, and **Edit routes** and **Add route** and enter 0.0.0.0/0 and select **Target** and select the first id in the list you see (or the only one if you see only one, there may be others that someone else working in the same region as you has already created but it doesn't matter if you select the one you created or they created), click **Save routes** and **Close**

Now whenever resources in the private subnet send traffic to the internet (0.0.0.0/0 bound traffic) they will forward that traffic to the NAT gateway which will forward it into the internet (passing back responses back to the requesting instance)

## Bonus Task:  Test your Bastion host
If you have more time and would like to test your Bastion host then create an instance in your private subnet and then SSH into the Bastion Host first and then into the private instance from the Bastion host.  

To accomplish this you could either create and download a new key pair for the new private instance and then first transfer that key pair file from your local machine into the bastion host (using the scp command with the -i option similar to how it is used for ssh) and then use that key file from the bastion host to ssh into the private instance (using the private IP address of the instance).  

Another approach is to set a run a script in the User Data of the private instance (set the script when you create the private instance) that allows you to ssh into the private instance with a password instead of a key pair (you can use the script below for that purpose).  If you use this approach you can ssh into the private instance from the Bastion host just using `ssh private-ip` (replace private-ip with the IP address of the private instance)

```bash
#!/bin/bash
# Turn on password authentication for lab bonus
echo 'lab-password' | passwd ec2-user --stdin
sed -i 's|[#]*PasswordAuthentication no|PasswordAuthentication yes|g' /etc/ssh/sshd_config
systemctl restart sshd.service
```


