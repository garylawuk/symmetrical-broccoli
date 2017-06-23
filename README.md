# symmetrical-broccoli

# A guide for getting started with AWS CLI and Spot Instances

THE SOFTWARE (INCLUDING DOCUMENTATION) IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE. SO THERE.

## Set up AWS account

Not documented.

## Install AWS command line tools

[Try following these instructions (MacOS / Linux)](http://docs.aws.amazon.com/cli/latest/userguide/awscli-install-bundle.html)

## Configure AWS command line tools

Create a suitable user in AWS IAM with full access, and then create an access key.  Keep a copy of the ID and the secret part . Then...


```
/usr/local/bin/aws configure
AWS Access Key ID [None]: xxxxxxxxxxxxx
AWS Secret Access Key [None]: xxxxxxxxxxxxxxxxxxxxxxx
Default region name [None]:
Default output format [None]:
```

## Install and configure ssh locally

Not documented.

## Install key pair into each region

```
for i in $(aws ec2 --region=eu-west-2  describe-regions | awk '/RegionName/ {print $2}' | tr -d \")
do
    aws ec2 --region "${i}" import-key-pair --key-name "symmetrical-broccoli"  --public-key-material "$(tail -1 .ssh/authorized_keys)"
done
```

## Create VPC in each region

```
cd
mkdir -p symmetrical-broccoli
cd symmetrical-broccoli
for i in $(aws ec2 --region=eu-west-2  describe-regions | awk '/RegionName/ {print $2}' | tr -d \")
do
    aws ec2 --region "${i}" create-vpc --cidr-block "10.10.10.0/24" --no-amazon-provided-ipv6-cidr-block > "symmetrical-broccoli-vpc-${i}"
done
```

## Set up each VPC

### Tags
```
for i in $(aws ec2 --region=eu-west-2  describe-regions | awk '/RegionName/ {print $2}' | tr -d \")
do
    VPCID=$(awk -F\" '/VpcId/ {print $4}' < symmetrical-broccoli-vpc-${i})
    aws ec2 --region "${i}" create-tags --resources "${VPCID}" --tags "Key=Name,Value=symmetrical-broccoli"
 done
 ```
 
### Networking for each VPC

Note: we should really be creating more than one subnet per VPC: one per AZ in each VPC is optimal as spot prices can vary per AZ.

```
for i in $(aws ec2 --region=eu-west-2  describe-regions | awk '/RegionName/ {print $2}' | tr -d \")
do
    VPCID=$(awk -F\" '/VpcId/ {print $4}' < symmetrical-broccoli-vpc-${i})
    aws ec2 --region "${i}" create-subnet --vpc-id "${VPCID}" --cidr-block "10.10.10.0/24" > "symmetrical-broccoli-subnet-${i}"
    SUBNETID=$(awk -F\" '/SubnetId/ {print $4}' < symmetrical-broccoli-subnet-${i})
    aws ec2 --region "${i}" create-internet-gateway > "symmetrical-broccoli-internetgateway-${i}"
    IGWID=$(awk -F\" '/InternetGatewayId/ {print $4}' < symmetrical-broccoli-internetgateway-${i})
    aws ec2 --region "${i}" attach-internet-gateway --vpc-id "${VPCID}" --internet-gateway-id "${IGWID}"
    aws ec2 --region "${i}" create-route-table --vpc-id  "${VPCID}"  > "symmetrical-broccoli-routetable-${i}"
    RTID=$(awk -F\" '/RouteTableId/ {print $4}' < symmetrical-broccoli-routetable-${i})
    aws ec2 --region "${i}" create-route --route-table-id "${RTID}" --destination-cidr-block "0.0.0.0/0" --gateway-id "${IGWID}"
    aws ec2  --region "${i}" associate-route-table --subnet-id "${SUBNETID}" --route-table-id "${RTID}"
 done
 ```
 
 Plus a suitable SG for each region
 
```
for i in $(aws ec2 --region=eu-west-2  describe-regions | awk '/RegionName/ {print $2}' | tr -d \")
do
    VPCID=$(awk -F\" '/VpcId/ {print $4}' < symmetrical-broccoli-vpc-${i})
    aws ec2 --region "${i}" create-security-group --vpc-id "${VPCID}" --group-name "symmetrical-broccoli" --description "symmetrical-broccoli" > "symmetrical-broccoli-security-group-${i}"
    SGID=$(awk -F\" '/GroupId/ {print $4}' < symmetrical-broccoli-security-group-${i})
    aws ec2 --region "${i}" authorize-security-group-ingress --group-id "${SGID}"  --protocol tcp --port 22 --cidr 0.0.0.0/0
    aws ec2 --region "${i}" authorize-security-group-ingress --group-id "${SGID}"  --protocol tcp --port 30303 --cidr 0.0.0.0/0
    aws ec2 --region "${i}" authorize-security-group-ingress --group-id "${SGID}"  --protocol udp --port 30301 --cidr 0.0.0.0/0
    aws ec2 --region "${i}" authorize-security-group-ingress --group-id "${SGID}"  --protocol udp --port 30303 --cidr 0.0.0.0/0
done
```
 
 create an SG inbound 22 outbound everything Ethereum clients use a listener (TCP) port and a discovery (UDP) port, both on 30303 by default also UDP 30301

##

To filter out a specific result by tag name
```
aws ec2 --region "${i}" describe-subnets --filter "Name=cidr-block,Values=10.10.10.0/24"
aws ec2 --region "${i}" describe-vpcs --filter "Name=tag-key,Values=Name" "Name=tag-value,Values=symmetrical-broccoli"
```

## Launch instance

### Build the seed image

We are aiming for latest CUDA and Nvidia support on Ubuntu 16.04, for which there are no precompiled binaries or even accurate instructions online.  This works:


```

add-apt-repository -y ppa:ethereum/ethereum
add-apt-repository ppa:graphics-drivers/ppa

cd
mkdir -p Downloads
cd Downloads
wget http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/cuda-repo-ubuntu1604_8.0.61-1_amd64.deb
dpkg -i cuda-repo-ubuntu1604_8.0.61-1_amd64.deb
cd
apt-get update
apt-get -y install autoconf autogen autotools-dev build-essential cmake ethereum gcc-4.9 g++-4.9 git libargtable2-dev libboost-all-dev libcryptopp-dev libcuda1-367 libcurl3 libcurl4-gnutls-dev libcurl4-openssl-dev libcryptopp-dev libgmp-dev libjsoncpp-dev libjsonrpccpp-dev libleveldb-dev libmicrohttpd-dev libreadline-dev mesa-common-dev nvidia-361 nvidia-367 nvidia-common  nvidia-cuda-toolkit nvidia-prime nvidia-settings ocl-icd-libopencl1 opencl-headers software-properties-common
apt-get install -y ocl-icd-libopencl1 opencl-headers linux-headers-$(uname -r)
(su - glaw ; echo 'SECRET' > password ; geth --password ./password account new ; geth --syncmode=fast --cache=2048)
apt-get install -y cuda
# optional ?
# wget http://developer.download.nvidia.com/compute/cuda/7.5/Prod/local_installers/cuda_7.5.18_linux.run
# optional ?
# wget https://developer.nvidia.com/compute/cuda/8.0/Prod2/local_installers/cuda_8.0.61_375.26_linux-run
# sudo apt install nvidia-cuda-toolkit
cd
mkdir -p Downloads
cd Downloads
git clone https://github.com/Genoil/cpp-ethereum/
cd cpp-ethereum
mkdir build
cd build
cmake -DBUNDLE=miner ..
make -j8
cd ethminer
touch mine.sh
chmod 755 mine.sh
 
   26  vi mine.sh
   27  ./mine.sh

mkdir /download
cd /downloads
git clone https://github.com/wolf9466/cpuminer-multi

Compile and install the cpuminer-multi:

cd cpuminer-multi
./autogen.sh
CFLAGS="-march=native" ./configure
make
make install



sudo apt-get update 
sudo apt-get install build-essential
sudo apt-get install nvidia-367 

sudo apt-get install freeglut3-dev



```


create a 512Gb volume

Not documented

/dev/sdf

http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-using-volumes.html

/dev/xvdf ?

Build binary?

apt-get install cmake





```
aws ec2 --region us-east-1 run-instances --subnet-id subnet-cd6649f1 --image-id ami-39269d2f --key-name symmetrical-broccoli --associate-public-ip-address --instance-type p2.xlarge
```
 
ssh to the host as ubuntu, run

```
sudo apt-get install software-properties-common
sudo add-apt-repository -y ppa:ethereum/ethereum-qt
sudo add-apt-repository -y ppa:ethereum/ethereum
sudo add-apt-repository -y ppa:ethereum/ethereum-dev
sudo apt-get update
sudo apt-get install -y ethereum
sudo apt-get install -y cpp-ethereum
```

then run 'nohup geth --cache=1024'


while that's running

```
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:george-edison55/cmake-3.x
sudo apt-get update
https://cmake.org/files/v3.8/cmake-3.8.2-Linux-x86_64.tar.gz

mkdir build; cd build
cmake ..
cmake --build .
```





```
geth
```

come back in a hour...


also for monero

```
sudo apt-get -y install git libcurl4-openssl-dev build-essential libjansson-dev autotools-dev automake
git clone https://github.com/hyc/cpuminer-multi
cd cpuminer-multi
./autogen.sh
CFLAGS="-march=native" ./configure
make
sudo ./minerd -a cryptonight -o stratum+tcp://xmr-usa.dwarfpool.com:8005  -u 475dxRBZPtUA9Bu7LLNcaUVX9MMcRmrctZmRoAiyJrvm2CzViENnUAdE2fd9qbuc5idX844LTf34ZYJ48CkFrekK7nnY19i -p x -t 3
```

