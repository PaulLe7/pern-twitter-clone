# Deploying a PERN app to AWS EC2
## Introduction
This is a guide to set up a PostgreSQL, Express, React, Node full stack web application to an AWS EC2 instance running an Amazon Linux AMI 2. The setup will use PM2 as a cluster manager and NGINX as a reverse proxy. We will use RDS to deply the PSQL database.

We will need to understand what the following are:
* Virtual private cloud (VPC) — A virtual network dedicated to your AWS account that you can deploy AWS resources into and have all your resources contained in one virtual place.

* Subnet — A range of IP addresses in your VPC. Subnetting in AWS is done with CIDR notation e.g. 10.11.4.0/24.

* Route table — A set of rules, called routes, that are used to determine where network traffic is directed.

* Internet gateway — A gateway that you attach to your VPC to enable communication between resources in your VPC and the internet.

* VPC endpoint — Enables you to privately connect your VPC to supported AWS services and VPC endpoint services powered by PrivateLink without requiring an internet gateway, NAT device, VPN connection, or AWS Direct Connect connection. Instances in your VPC do not require public IP addresses to communicate with resources in the service. Traffic between your VPC and the other service does not leave the Amazon network. For more information, see AWS PrivateLink and VPC endpoints.

* CIDR block —Classless Inter-Domain Routing. An internet protocol address allocation and route aggregation methodology. For more information, see Classless Inter-Domain Routing in Wikipedia.

_Source: https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html_

## 1. Setting up the VPC and Subnets

_**This section requires working from the AWS Management Console.**_

-----

**1.1. Create a new VPC**

* Choose a meaningful name for the new VPC
* Give a CIDR block e.g. 10.11.0.0/16
* Choose 'No IPv6 CIDR block'
* Set tenancy to 'Default'

In the example above, 10.11.0.0/16 is a CIDR block where the first 2 bytes [10.11] are the unchangeable network prefix and the last 2 bytes [0.0] are unsued and can be used to create private host addresses that can be assigned to different resources within the VPC. The /16 is a subnet mask which indicates that 2 bytes or 16 bits are available as ip addresses for the VPC. Since each byte is 8 bits (which is either a 0 or a 1, and which can also be called an octet), that means there are 2^8 = 256 combinations available per byte/octet. This means that there are 256 * 256 = 65,536 IP addresses are usable by the new VPC. Since computers count from 0, each octet can be a number between 0 to 255.

Note that the /16 subnet mask denotes the number of bits that are in use as the network prefix. Since there are 4 octets per IPv4 address, this means there are 32 bits in total. Hence, the /16 - which is half of the 32 total bits - implies that 16 bits are in use as the network prefix, so the remaining 16 bits are available to the subnet to be used as local host addresses.

-----

**1.2. Create new subnets**

* Create 3 subnets
* 1 public subnet which will contain our web server and be accessible over the internet
* 2 private subnets which will contain our PSQL database and will not be accessible over the internet

Create the first subnet as the public subnet. It does not need a preference for its Availability Zone.

The second and third subnets will be private and require Availability Zone preferences that differ from each other. This is because the RDS PostgreSQL database that we will use later requires two subnets in different availability zones to be deployed.

Subnets are simply sub networks within a wider network, in this case, the VPC. Subnetting is used to divide up the VPC for performance and security reasons To differentiate each subnet, they must have differing IPv4 addresses local to the VPC. We know that the last two octets of the VPC are available to use as host addresses, so the IPv4 CIDR blocks assigned to our subnets can be as such: subnet 1 can use 10.11.1.0/24, subnet 2 can use 10.11.2.0/24, and subnet 3 can use 10.11.3.0/24. Since there is one octet unsed by the subnets we just created (the third octet is used as the unique identifier for each subnet - notice the /24), this means that within each subnet, 256 other local IP addresses can be used (e.g. 10.11.X.0 to 10.11.X.255).

Our web server and database will be able to communicate with each other through a route table.

-----

**1.3 Create an Internet Gateway**

* Give it a name tag that is meaningful
* Under 'Actions', attach it to your VPC

We will connect this internet gateway to our public subnet (but first we need to set up the route table).

The internet gateway connects the VPC to the internet. It can receive requests from the internet to the VPC, or take requests from the VPC which are then sent into the internet. The internet gateway has its own public IP address which identifies the VPC network as a whole on the internet. The VPC network, (as alluded to in the above sections) can contain within itself thousands of IP addresses.

-----

**1.4 Create Route Tables**

* Create 2 route tables both connected to your VPC
* 1 route table for your public subnet (web server)
* 1 route table for your private subnets (database)
* Edit the routes and subnet associations of each route table (instructions below)

Notice that each route table already has one route with destination 10.11.0.0/16, which is the local route. This allows resources within the same network to communicate with each other.

For the public route table, add an additional route with the destination set as 0.0.0.0/0 and the target set as the internet gateway. This essentially enables any routes that are not for the local 10.11.0.0/16 route to be forwarded to the internet gateway. The 0.0.0.0/0 represents all routes that are not local.

We do not add the internet gateway route to the private route table.

For both route tables, edit the subnet associations by adding the public subnet to the public route table, and the remaining two private subnets to the private route table.

-----

**1.5 Create a Security Group for the web server**

* Give the security group a meaningful name and short description
* Connect it to your VPC
* Set up inbound and outbound rules (instructions below)

Security groups filter incoming and outgoing traffic. They are basically firewalls.

The security group we will be creating is for our public subnet (the web server). Since this subnet is accessible via the internet, we need to define inbound rules to make it secure.

| Type                | Port range          | Source              | IP version |
| ------------------- |:-------------------:|:-------------------:|:----------:|
| HTTP                | 80                  | 0.0.0.0/0           | IPv4       |
| HTTP                | 80                  | ::/0                | IPv6       |
| HTTPS               | 443                 | 0.0.0.0/0           | IPv4       |
| HTTPS               | 443                 | ::/0                | IPv6       |
| SSH                 | 22                  | Your IP address     | IPv4       |

HTTP and HTTPS are set to their standard ports of 80 and 443 respectively with 0.0.0.0/0 and ::/0 allowing all IPv4 and IPv6 traffic.

Lastly, we have SSH which will be how we connect to the EC2 instance from our local machine. We put the source IP as our own IP address so that no other machine but our own can access the instance.

## 2. Setting up the production build

**_In this section, we modify the code of our web application so that it is ready to be deployed_**

-----

**2.1 Make necessary changes to the client side code**

When running a react app on our local system we usually have have two instances running. One for the server and one for the client. However, in deployment we want combine these two to run as one instance. We do this having the srver-side code serve the client side code. Since the client side code will no longer be running on a different instance but will be running on the same instance as the server, naturally this means they will share the same port. This means we no longer need to specify the address of the server side in the client-side files when writing functions that make requests to the server.

For example, instead of writing the following:
```javascript
const response = await fetch(
                "http://localhost:5000/auth/register",
                {
                    method: "POST",
                    headers: {
                        "Content-type": "application/json"
                    },
                    body: JSON.stringify(body)
                }
            );
```
We now omit the `http://localhost:5000/` part of the fetch address so that it becomes `await fetch("/auth/register", ... )`.

-----

**2.2 Create the client-side build**

Once the necessary code changes have been made on the client side, we can now package it into a production build by running the following. This will output a build of the React app in a directory called build.
```bash
npm run build
```
Move the build folder into the node/express server.

-----

**2.3 Change relevant server-side code so that the "index.html" in our new build is served**

We now need to make changes in the server-side code so that our client side will be served up by the server.

This is achieved by setting a path to the build directory:
```javascript
const path = require("path");

// points to the build directory so the server can serve the static files from our React app
app.use(express.static(path.join(__dirname, 'build')));

// ensures that index.html is served up by the server for every route when we are in a production environment
if (process.env.NODE_ENV === 'production') {
    app.get('*/', (req, res) => {
        res.sendFile(path.resolve(__dirname, 'build', 'index.html'));
    });
}
```
**Note:** We can also use `res.sendFile(path.join(__dirname, 'build', 'index.html'));`. The different between `path.join` and `path.resolve` are in how they deal with segments starting with `/`; `join` will concatenate it with the previous argument, `resolve` will treat the previous argument as the root directory and ignore all previous paths.
```javascript
path.join('/a', '/b') // Outputs '/a/b'

path.resolve('/a', '/b') // Outputs '/b'
```
`path.resolve` will always result in an absolute URL, and will use the current working directory as a base to resolve this path. But as `__dirname` is an environment variable that tells you the absolute path of the directory containing the currently executing file, in this case it doesn't matter whether we use `join` or `resolve`.

-----

Once all the necessary changes have been made, the project is now ready to be deployed to a GitHub repository.

## 3. Launch a cloud computer using AWS EC2

_**First, go to the EC2 dashboard via the AWS Management Console.**_
-----
**3.1 Create a new EC2 instance**

* Click on 'Launch instance'
* Select the Amazon Linux AMI operating system free tier option
* Configure the instance details (instructions below)

**3.1.1 Use the following configurations for our new EC2 instance:**
* **Network:** Choose our VPC
* **Subnet:** Choose the public subnet
* **Auto-assign Public IP:** Enable this
* Leave everything else as is (or configure to your liking)

For the purposes of this exercise, we leave the storage as default and do not need to set any tags.

**3.1.2 Choose a security group for the new EC2 instance:**
* Choose 'Select an existing security group' and choose our previously defined security group for our web server
**
3.1.3 Hit 'Launch' and choose 'Create a new key pair' when prompted:**
* Give the key pair a new name
* Download the key pair
* **Important:** Make sure to keep the key pair safe, as it cannot be downloaded again. It is needed to access the EC2 instance from our local computer.

The EC2 instance will take a short while before it starts running after it has been launched.
-----

**3.2 Accessing the EC2 instance from our local computer**
* Open a new terminal window and connect to the EC2 instance using the following command:
```bash
ssh -i "your-keypair.pem" ec2-user@your-ec2-instance-public-ip-address
```
* The following commands can be used in our newly connected virtual computer:
```bash
cd /                      # goes to root directory
ls                        # lists files in current directorylea
pwd                       # shows path of current directory
sudo nano                 # opens up the ubuntu instance's code editor
sudo rm                   # removes file or directory
```

## 4. Setting up a PostgreSQL database on AWS

_**First, go to the RDS Managemment Console via the AWS Management Console.**_
-----

**4.1 Create a Database Subnet group**
* In the RDS Management Console, click on 'Subnet groups' and select 'Create DB Subnet Group'
* Give the subnet group a name
* Choose your VPC
* Add our two private subnets to the subnet group (the subnets will be listed under the availability zone that we set them up in)
* Clck 'Create'

The database subnet group spans to availability zones so that if a server fails in one availability zone, our database will still be fine.

-----
**4.2. Create a Security Group for the Database**
* Go back to the VPC console and create a new security group
* Give it a meaningful name
* Edit the inbound rules and add a 'Custom TCP Rule' with a Source set to 'Custom' like the below:

| Type                | Port range          | Source              | IP version |
| ------------------- |:-------------------:|:-------------------:|:----------:|
| Custom TCP Rule     | 5432                | Public subnet CIDR  | IPv4       |

We set Source to 'Custom' and use the CIDR block of the public subnet which contains the web server e.g. 10.11.1.0/24. This ensures that only traffic coming from this local IPv4 address can speak to our database.

-----

**4.3 Create a new PostgreSQL database**
* Choose 'Create Database' using 'PostgreSQL' (use the free tier to avoid costs)
* Name your DB instance identifier
* Pick your Master username (and set a password if you want)
* Pick a DB instance class (pick the lowest performance one if you only want to test out how things work to reduce costs)
* Choose your VPC
* Select the newly created Database Subnet Group
* Set Public Accessibility to 'No' (we don't want our database to be accessible over the internet)
* Leave the Availablity Zone as 'No preference'
* Choose your newly created Database Security Group
* Have your database name be the same as the Database instance identifier (for convenience)
* Uncheck 'Enable automated backups' (so that the new database instance is created more quickly)
* For the purposes of this exercise, we can leave everything else as default
* Create the Database

-----

**4.4 Connect to the AWS PSQL Database**
* Connect to your EC2 instance
* Install PSQL on the EC2 instance
```bash
sudo amazon-linux-extras install postgresql13     # or any other version you would like to use
```
* Access the PSQL database through our EC2 instance
```bash
psql -d database_name -h db_endpoint -p db_port -U master_username
```
**Note 1:** You will always be prompted to enter a password following the above command. However, if you did not set up a password, simply press enter to proceed.

**Note 2:** Navigate the RDS Management Console to find the database name, endpoint, and port under the 'Details' section of the newly created AWS PSQL Database.

-----
Once inside the new PSQL database, create the necessary databases and tables for your app using standard PSQL commands.

## 5. Deploying the app to AWS

**5.1 Clone your web app onto the EC2 instance**
* First, install git on your EC2 instance
```bash
sudo yum install git
```
* Then, clone the GitHub repository which contains your web app
```bash
sudo git clone your_github_repo_url
```
-----
**5.2 Install Node via the Node Version Manager**
* Install the node version manager which is used to download npm and node
```bash
sudo curl https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash 
```
* Exit and re-access the EC2 instance so that the installation takes effect
* Install Node
```bash
nvm ls-remote                           # lists the node versions available for download, choose the latest LTS
nvm install latest_node_LTS
```
-----

**5.3 Edit your Public Security Group**
* Add two new inbound rules with using 'Custom TCP Rule' with the 'Port range' set to the port that your app is running on.

For example:
| Type                | Port range          | Source              | IP version |
| ------------------- |:-------------------:|:-------------------:|:----------:|
| Custom TCP Rule     | 5000                | 0.0.0.0/0           | IPv4       |
| Custom TCP Rule     | 5000                | ::/0                | IPv6       |

**5.4 Try running your app**
* Give your app necessary permissions
```bash
sudo chmod 777 absolute_path_to_your_app
```
* Install the packages needed by your app
```bash
npm install
```
* Try running your app with `npm start` or `node your_server_files_name.js`

-----

If everything has been set up correctly, you should be able to access your app via `your_ec2_public_ipv4_address:server_port` e.g. `http://34.209.87.87:5000`.

## 6. Running the app with PM2


## 7. Nginx (reverse proxy) production setup

## 8. Terminating AWS resources (to avoid costs)

