# Creating secure AWS infrastructures
## Here we'll see the steps to create a secure AWS infrastructure. It will contain a linux EC2 instance on a private subnet which cannot be accessed using SSH except from a single instance from within our infrastructure, a Bastion instance AKA a jump box. Our secure EC2 instance **will** be able to access the internet freely, by passing all requests via a NAT (Network Access Translator) acting as a network request re-direct. 

##As our secure EC2 instance is in a private subnet i.e. with no internet gateway, our secure instance cannot be accessed from the the public internet, making it secure.

### Setting up the network

> Here we'll create our VPC (*virtual private cloud*) and 2 subnets - one public, which has access to the public internet, and the other one private, which can not access the public internet. The fundamental factor in making this a secure system, is by tightly controlling who can access the instance from the public internet, and any instance on the private subnet is not accessible by default and by design.


1. Create a VPC **BastionVPC** with CIDR block = **10.0.0.0/16**
    -> All inbound and outbound traffic is allowed via the default NACL. For increased control, create a custom NACL and associate with this VPC which only permits requests from the IPs, ports and protocols you define.

2. Create an internet gateway (**BastionInternetGateway**. We'll use the generic acronym IGW to refer to internet gateways from no on.

3. The created Internet will be in a 'detached' state i.e. not attached to any VPC. Select BastionInternetGateway, press the action button, then press attach to VPC. **Attach it to BastionVPC.**


4. Create a subnet called **BastionPublicSubnet**, and associate it to BastionVPC with the CIDR block = **10.0.1.0/24**

5. Create a subnet **BastionPrivateSubnet** associated to BastionVPC, with CIDR block = **10.0.2.0/24**

**ROUTE TABLES**

> Route tables define the routing of network requests directed at or coming from a subnet. Here we'll create a custom route table for the private subnet of our infrastructure and configure it to strictly control the network requests to and from our private subnet. Our public subnet on the other hand, must have access to the internet and we'll configure that too.

6. Click BastionPublicSubnet to see this subnet's associated information. Click the **route table** link and you'll see a table of information related to BastionPublicSubnet's routetable. **Click the routes tab and then the edit routes button**. Add a new route from **0.0.0.0/0 to BastionInternetGateway**. 

7. In route tables, **create a new route table  called BastionPrivateRouteTable**, associated to BastionVPC. Once created, follow the displayed link to the route table information and **check the subnets  associated to the route table**. There should be no subnets associated right now because you just created it.  **Click the edit subnet associations button and associate BastionPrivateSubnet to it**.

> You'll notice that BastionPrivateRouteTable does not have access to the public internet by default, which is exactly what we want for now. You should probably keep this browser tab open as you'll need to add a new entry to the routes table in a few minutes.


### Creating the NAT instance
1. Navigate via the Services button to the EC2 instances.
2. You will use an image dedicated to being a *Network Access Tranformation* device i.e. **a NAT instance**. Search for the images using the seach parameter **amzn-ami-vpc-nat**. Create your instance using this image on BastionVPC and BastionPublicSubnet. *Auto-assign an IP and you save yourself a job later*.
3. When prompted about security groups, go ahead and create a new one for this instance. Call it **BastionNATInstanceSG**.
	-> SSH should be allowed on port 22 from your device
	-> All ICMP - IP4 should be allowed on all ports from anywhere (you can come back and further restrict this later once its working of you choose e.g. to only http and https requests on ports 80 and 443 respectively)
4. When you launch the instance, a window will pop up "Select an existing key pair or create a new one." 

> The key pair is composed of two parts, a public and a private key. They are used to encrypt/decrypt your log in details when you SSH into your instance.

**Create a new key pair called BastionNATinstancePrivateKey**, download it and save it somewhere safe and secure on your device. 

You can now **launch the instance** with the big blue 'Launch' button.

5. In the next screen, for convenience you can add a tag to help you identify this instance in a table of instances. Add the label **BastionNATinstance** to the 'Name' column of the instances table.

6. **AWS instances are given a default configuration of Check Src/Dest = Enabled**. This means only traffic to or from the instance is allowed. As most of the traffic directed at a NAT instance is to be redirected elsewhere by the NAT, **we need to change this default value to Check Src/Dest = Disabled** by selecting BastionNATinstance and using the Action->Networking options.

7. Remember I said you'd need to add a new route to the BastionPrivateRouteTable? Well now is the time. Now that you have created your NAT instance,we can use it like an internet gateway for our private subnet.

Simply **add a new route to *BastionPrivateRouteTable* from 0.0.0.0/0 (that means to everywhere in the public internet) to BastionNATinstance.** Then BastionNATinstance will route it via the internet gateway to the public internet and return the results to our secure instance.

### Creating the JumpBox instance

> A 'jump box' or 'bastion' instance, is used to create a specific network device which can be used as a jumping point to other instances in inaccesible, private subnets. Our secure EC2 instance needs to be accessible by you in order to finish these steps, but right now it is completely isolated from the public internet, meaning you at home can't access it. The solution is, to create a jump box instance (a normal EC2 instance) in the public subnet which you can ssh into. Then to configure our secure EC2 to only accept incoming ssh on port 22 from our jump box. This closes all the possible access points to our secure instance except one, which we tightly control. 


1. Launch a new instance, a free tier Linux machine is just fine. It should be on BastionVPC and BastionPublicSubnet as before. **Assign it a public IP so you can access it from home**.

2.  Create a new security group when prompted called **BastionJumpBoxSG**. BastionJumpBoxSG should only allow SSH on port 22, from your IP. **Note that if you're working on a laptop and change locations, your ip will change and you'll need to update this setting**.

3. When prompted, create a new keypair called **BastionJumpBoxKeyPair**. Download the private key and store it safely.
4. For convenience, give it the tag **name=BastionJumpBox**.
5. **Take a note of the BastionJumpBox private IP**.


### Creating your secure, final instance.

> Here we're launching our secure EC2 instance. Not too different from any other EC2 instance we might create on AWS, except for one thing - it will only accept ssh from our jumpbox machine.

1. Launch a new instance on BastionVPC, of the same free tier linux image you used for the BastionJumpBox. **It must be on BastionPrivate and automatic assignation of a public IP should be disabled**.
2. When prompted, create a new security group called **BastionSecureFinalInstanceSG, with SSH enabled from the source of your BastionJumpBox private IP**. To be a valid CIDR block, you'll need to add **/32** at the end, to ensure that only that specific IP (and not variations of it) are given access to BastionSecureFinalInstanceSG.
3.  When promted create a new keypair called **BastionSecureFinalInstanceKeyPair**. Download the private key to a secure place on your local device and launch the instance.
4. For convenience, give this instance a name in the left most column of the instances table. Let's call it **BastionSecureFinalInstance**.

## Where are now?
We've now created 2 instances on our public subnet. One is a NAT instance, which redirects traffic through the BastionInternetGateway of its public subnet. You have a jumpbox instance, which will only accept incoming requests via port 22 using SSH protocol because its only reason for existing is to allow you to ssh into it.

The BastionSecureFinalInstance is on a private subnet, which has no direct access to the internet. However requests from BastionSecureFinalInstance can be sent to the NAT instance, which then routes the requests to the public internet. So BastionSecureFinalInstance can access the internet, but the internet cannot access BastionSecureFinalInstance. Only BastionJumpBox and BastionNATInstance can access BastionSecureFinalInstance.

### However...

You can SSH to your BastionJumpBox now but your goal is to SSH BastionJumpBox to your BastionSecureFinalInstance. In order to do this, you need the private key to authenticate your credentials. You *could store them on BastionJumpBox, but it's frowned upon as it represents a security risk to let your private key escape the control and supervision of your own device.*

What we'll do is keep both keys on our local device, and when we SSH to the jump box, we'll send two private keys - one for BastionJumpBox and one for BastionNATinstancePrivateKey.

This can be a bit tricky and you'll need a couple of small, useful and free applications related to SSH. **Putty, Puttygen and Pageant (all available here** [Putty Download Page](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) )

1. You need to convert your .pem private key files to ppk files using the import/conversion function of puttygen. Once done, you'll have ppk files of your private keys.
2. Open pageant and press the add key button, then browse your new ppk private keys and add them all. This software runs in the background, making your private keys available.
3. Now open putty. Putty is used to SSH into a remote machine. On the left side panel, enable agent port forwarding on SSH->Auth.
4. Now SSH to BastionJumpBox by adding the username and host ip you can see for BastionJumpBox.
5. You'll find yourself controlling the BastionJumpBox instance. Now from BastionJumpBox you'll now ssh into the BastionSecureFinalInstance using the user and password of BastionSecureFinalInstance you can find it on the connect buttom when BastionSecureFinalInstance is selected in the EC2 running instances. The command is **ssh -i <user and password of BastionSecureFinalInstance>** If all went well, you should now be controlling the remote instance BastionSecureFinalInstance which lives on a private subnet of your AWS infrastructure, with no internet access.
6. One last thing - you have a NAT instance to allow BastionSecureFinalInstance to access the public internet, even though it's on a private subnet with no internet gateway. You can test that that's working with the command **ping google.com**. If you receive a response, it means that BastionSecureFinalInstance which lives on a private subnet of your AWS infrastructure, with no internet access, has internet access ^_^

## This has been long and complicated process, and may have been confusing for new AWS users. However this methodology is considered best practice and is a ubiquitous method of securing instances in the cloud from malicious attacks or interventions from unknown apps or people. Being aware and experienced with this methodology makes you more prepared for using cloud computing in a professional environment.

#So congrats, you did it.


