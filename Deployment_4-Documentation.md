<h1>CI/CD pipeline deployment 4 </h1>

<h2>(1) Set up and configure your Jenkins environment within default VPC.</h2>


- install and configure jenkins on an EC2 to listen on port 22 to be able to SSH into the EC2, and 8080 to access Jenkins' web interface.

- On the EC2 install default-jre which is a Java Runtime Environment (JRE) which is required to run Java programs, python3-pip which tool for installing Python packages ,
  python3.10-venv which is a virtual environment for python so that the Python interpreter, libraries, and scripts installed on here are totally isolated from those installed 
  on the virtual machine. Then install terraform on the EC2 to enable us to provision the infrastructure necessary to successfully deploy the application by only writing a few lines of code.
  Git would also need to be installed for the purpose of versioning your infrustructure and to keep track of all changes being made.
  
  
<h2>(2) Configure credentials on jenkins.</h2>

- On jenkins, configure the credentials for your AWS user where you would set up your aws_access_key and aws_private_key to allow you to build build your infrustructure via terraform. 
  Jenkins needs credentials that it will use in its pipeline so that Terraform is authorised to control the resources in the infrastructure.
  
<h2>(3) Create a pipeline build in jenkins.</h2>

- This is where the steps on how to build, test, Init, Plan, Apply and destroy the application's infrustructure will be defined and orchestrated. The build stage will    install of the dependencies 
  needed for the application to run. The test stage test to see of a status code of <200> is received when the home page of the application. The Init stage Initializes terraform create a state file
  which records the current state of the infrustructure. The plan stage will build out and blueprint of all the resources the you intend to create with AWS. The apply stage will actually start creating
  all the resources in AWS which was planned in the plan stage. Finally the destroy stage will delete all the recources that was created in AWS. 
  
<h2>(4) Add a destroy stage.</h2>

 - After you have successfully built and ran the pipeline in jenkins, add the destroy stage to the jenkinsfile.
 - Navigate to the jenkinsfile on github and add the following code after the apply stage.
 
 ```
 stage('Destroy') {
    steps {
        withCredentials([string(credentialsId: 'AWS_ACCESS_KEY', variable: 'aws_access_key'),
            string(credentialsId: 'AWS_SECRET_KEY', variable: 'aws_secret_key')]) {
                dir('intTerraform') {
                    sh 'terraform destroy -auto-approve -var="aws_access_key=$aws_access_key" -var="aws_secret_key=$aws_secret_key"'
                }
            }
    }
}
 ```
 
<h2>(5) Deploy the application in your VPC using terraform.</h2>

- Navigate to the IntTerraform folder and open the main.tf file. Replace the existing code of that file with the code below.

```
variable "aws_access_key" {}
variable "aws_secret_key" {}

provider "aws" {
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key
  region = "us-east-1"
  
}

#EC2-1
resource "aws_instance" "web_server01" {
  ami = "ami-08c40ec9ead489470"
  instance_type = "t2.micro"
  subnet_id = aws_subnet.subnet1.id
  key_name = "jenkins-key"
  vpc_security_group_ids = [aws_security_group.web_ssh.id]
  associate_public_ip_address = true

  user_data = "${file("deploy.sh")}"

  tags = {
    "Name" : "Webserver001"
  }
}

# VPC
resource "aws_vpc" "test-vpc" {
  cidr_block           = "172.27.0.0/16"
  enable_dns_hostnames = "true"
 
  tags = {
    "Name" : "CALEB-VPC"
  }
}

# ELASTIC IP
resource "aws_eip" "nat_eip_prob" {
  vpc = true
 
}

# NAT GATEWAY
resource "aws_nat_gateway" "nat_gateway_prob" {
  allocation_id = aws_eip.nat_eip_prob.id
  subnet_id     = aws_subnet.subnet1.id
}

# SUBNET 1
resource "aws_subnet" "subnet1" {
  cidr_block              = "172.27.0.0/18"
  vpc_id                  = aws_vpc.test-vpc.id
  map_public_ip_on_launch = "true"
  availability_zone       = data.aws_availability_zones.available.names[0]
}

# INTERNET GATEWAY
resource "aws_internet_gateway" "gw_1" {
  vpc_id = aws_vpc.test-vpc.id
}

# ROUTE TABLE
resource "aws_route_table" "route_table1" {
  vpc_id = aws_vpc.test-vpc.id
 
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw_1.id
  }
}

resource "aws_route_table_association" "route-subnet1" {
  subnet_id      = aws_subnet.subnet1.id
  route_table_id = aws_route_table.route_table1.id
}

# DATA
data "aws_availability_zones" "available" {
  state = "available"
}

output "instance_ip" {
  value = aws_instance.web_server01.public_ip
  
}
```

<h2>(6) Addition.</h2>

- This is where you will check to see if the response received from the page_not_found.html page is a status code of 404
- Navigate to the 'test_app.py' file and paste in the code below.

```
#Check to see if the response received is a status code of 404
def pgNOTfound():
    response = app.test_client().get('page_not_found.html')
    assert response.status_code == 404
```

<h2>(7) Possible Errors.</h2>

- InvalidKeyPair.NotFound: The key pair 'jenkins-key' does not exist
- inside the main.tf file in the IniTerraform folder, on line 16 which asks for the key pair name, make sure this is changed to a key name which already
  exixts on your AWS account.


