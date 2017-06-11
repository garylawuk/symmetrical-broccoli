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

