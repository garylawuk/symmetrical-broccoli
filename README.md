# symmetrical-broccoli

# Instructions for GPU mining of ETH on AWS

THE SOFTWARE (INCLUDING DOCUMENTATION) IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

## Set up AWS account

Not documented.

## Install AWS command line tools

Not documented.

## Configure AWS command line tools

Not documented.

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
 
 create an SG inbound 22 outbound everything Ethereum clients use a listener (TCP) port and a discovery (UDP) port, both on 30303 by default also UDP 30301

## Launch instance

### Build the seed image

based mine on pytorch-cuda75

create a 512Gb volume

Not documented

/dev/sdf

http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-using-volumes.html

/dev/xvdf ?





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




then run 'nohup geth'

```
geth
```

come back in a hour...




