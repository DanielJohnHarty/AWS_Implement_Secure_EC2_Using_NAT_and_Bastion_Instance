## Steps to ceate AWS Secure Infrastructure with linux instance on private subnet accessing internet through a NAT linux instance and external SSH access from the internet via a JumpBox (Bastion) linux instance.

### Setting up the network
1. Create a VPC **BastionVPC** with CIDR block = **10.0.0.0/16**
    -> All inbound and outbound traffic is allowed via the default NACL. For increased control, create a custom NACL and associate with this VPC which only allows the IPs, ports and protocols you expect

2. Create an internet gateway (**BastionInternetGateway**).

3. The created IGW will be 'detached' i.e. not attached to an exiosting VPC. Select BastionInternetGateway then press the action button, then attach to VPC. Attach to BastionVPC.


4. Create a subnet **BastionPublicSubnet** associated to BastionVPC with CIDR block = **10.0.1.0/24**

5. Create a subnet **BastionPrivateSubnet** associated to BastionVPC with CIDR block = **10.0.2.0/24**

## ROUTE TABLES

6. Click BastionPublicSubnet to see the associated information. Click the route table link, click the routes tab and then the edit routes button. Add a new route from **0.0.0.0/0 to BastionInternetGateway**. Make sure that BastionPublicSubnet's route table is associated to BastionPublicSubnet.

7. In route tables, create a new one called **BastionPrivateRouteTable** and associated to BastionVPC. Once created, follow the link to the route table and check the subnet associations. THere should be no subnets associated right now.  Click the edit subnet associations button and associate **BastionPrivateSubnet**.


### Creating the NAT instance
1. Navigate via the Services button to the EC2 instances.
2. This time, you will use a different image, becuase you are creating an instance solely dedicated to being a *Network Access Tranformation* device i.e. **a NAT instance**. Search for the images using the seach parameter **amzn-ami-vpc-nat**. Create your instance using this image on BastionVPC and BastionPublicSubnet. *Auto-assign an IP and you save yourself a job later*.
3. During the process, you'll be asked about assigning a security group. Go ahead and create a new security group for this instance. Call it **BastionNATInstanceSG**.
	-> SSH should be allowed on port 22 from anywhere (you can come back and further restrict this later once its working of you choose)
	-> All ICMP - IP4 should be allowed on all ports from anywhere (you can come back and further restrict this later once its working of you choose e.g. to only http and https requests on ports 80 and 443 respectively)
4. When you launch the instance, a window will pop up "Select an edxisting ke pair or create a new one." The key pair is composed of two parts, a public and a private key. The private key is very private and can only be downloaded now - so download it and store it securely. It is used to encrypt your log in details when you try to SSH into your instance.

The public key is stored on the instance, and is used to decrypt your login details. It will be automatically stored in the right place on the instance, so you don't need to worry about it.

So create a new key pair called BastionNATinstancePrivateKey, download it and save it somewhere safe and secure on your device. You can now launch the instance with the big blue 'Launch' button.
5. For convenience, add the label **BastionNATinstance** to the leftmost column (the 'Name' column) of the instances table.
6. AWS instances are given a default configuration of Check Src/Dest is True. This means only traffic to or from the instance is allowed. As a Nat instance is acting as a traffic post office, you need to change this instance to **Check Src/Dest = Disabled** by selecting BastionNATinstance and using the Action->Networking options.

### Creating the JumpBox instance
1. Launch a new instance, the same free tier linux image you used for the BastionNATinstance. IT should be on BastionVPC and BastionPublicSubnet as before. **Assign it a public IP as before**.
2.  Create a new security group when prompted called **BastionJumpBoxSG**. BastionJumpBoxSG should only allow SSH on port 22, from anywhere.
3. When prompted, create a new keypair called **BastionJumpBoxKeyPair**. Download the private key and store it safely.
4. For convenience, give it the name **BastionJumpBox** on the left most column of the instances, table.
5**. Take a note of the BastionJumpBox private IP**. 


### Creating your secure, final instance.
1. Launch a new instance on BastionVPC, of the same free tier linux image you used for the BastionNATinstance. *However it should be on Bastion**Private**Subnet and automatic assignation of a public IP should be **disabled***.
2. When prompted, create a new security group called **BastionSecureFinalInstanceSG**, with SSH enabled from the source of your BastionJumpBox private IP. To be a valid CIDR block, you'll need to add **/32** at the end, to ensure that only that specific IP (and not variations of it) are given access to BastionSecureFinalInstanceSG.
3.  When promted create a new keypair called **BastionSecureFinalInstanceKeyPair**. Download the private key to a secure place on your local device and launch the instance.
4. For convenience, give this instance a name in the left most column of the instances table. Let's call it **BastionSecureFinalInstance**.

## So what have we done?
We've now created 2 instances on our public subnet. One is a NAT instance, which handles traffic within through the your BastionInternetGateway. You have a jumpbox instance, which will only accept incoming requests via port 22 using SSH protocol. 

The BastionSecureFinalInstance is on a private subnet, which has no direct access to the internet. However requests from BastionSecureFinalInstance and sent to the NAT instance, which then routes the requests to the internet. So BastionSecureFinalInstance can access the internet, but the internet cannot access BastionSecureFinalInstance. Only BastionJumpBox and BastionNATInstanceSG can access BastionSecureFinalInstance.

### However...

You can SSH to your BastionJumpBox now but your goal is to SSH to your BastionSecureFinalInstance. In order to do this, you need the private key to authenticate your credentials. You could store them on BastionJumpBox, but it's considered an important security risk to let your private key escape the control and supervision of your own device.

What we'll do is keep both keys on our device, and when we SSH to the jump box, we'll send two private keys - one for BastionJumpBox and one for BastionNATinstancePrivateKey.

This can be a bit tricky and you'll need a couple of small, useful and free applications related to SSH. Putty, Puttygen and Pageant (all available here ![Putty Download Page](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) )

1. You need to convert your .pem private key files to ppk files using the import/conversion function of puttygen. You can close this app now.
2. Open pageant and press the add key button, then navigate to your keys and add them. This software runs as a service so you can close the window and it will continue to run in the background for when you need them.
3. Now open putty. This is used to SSH into a remote machine. On the left side panel, enable agent port forwarding on SSH->Auth.
4. Now SSH to BastionJumpBox using the command **ssh <BastionJumpBoxPublicIP>**
5. You'll find yourself controlling the BastionJumpBox instance. Now, remote into the BastionSecureFinalInstance using **ssh <BastionSecureFinalInstance>**. If all is well, you're controlling the remote instance BastionSecureFinalInstance which lives on a private subnet of your AWS infrastructure, with no internet access. Congrats nerd ^_^ 


