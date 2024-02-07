## Requirements
1. Sapio BLS server must be installed in a linux filesystem.
2. Sapio BLS server must have network connection to the Sapio Analytics Server over a outbound TCP port.

## Deployment of Sapio Analytics Server (Single Instance)
1. Modify the docker-compose.yaml to ensure your port outbound was desirable.
2. Modify the docker-compose.yaml to use a custom API key.
2. Double check docker-compose.yaml to make sure the image path is the version you wanted. (You should have used a tagged build)
3. Use the command "docker-compose pull" to grab latest image. (Unless this is Sapio dev building from scratch.)
```shell
sudo docker-compose pull
```
5. Use the command "docker-compose up" to create instance.
   Note you can use --detached so the lifecycle does not end when you exit console.
   You can also use -p to specify a custom project name but usually more convenient to rename the parent directory.
```shell
sudo docker-compose up -d
```
6. Grab the keystore file in /data/execserver.keystore from the image (docker container cp sapioanalyticsserver_sapio_analytics_server_1:/data/execserver.keystore ~/Downloads) and put it into your Sapio server /opt/sapiosciences folder
   Alternatively, generate your own and put that inside the data volume (it persists) and replace the keystore file.
   The generated keystore must be of JKS format and have a key with alias "server" of RSA algorithm. It can be generated from keytool.
   The file may be located any readable place in Sapio server and renamed if desired.
   The default password is "123456" (without quotes) for both the keystore password and the key password. The keystore password should not be changed unless you are rebuilding the image with custom Dockerfile.
7. In Sapio server navigate to /opt/sapiosciences and create file ClientSettings.properties like below.
   **Note the file /opt/sapiosciences/ClientSettings.properties is a hard-coded path**.
   Example content in the file:
```
server.address=sapio-analytics-server-ip-address
server.port=8686
apikey=The Key in docker-compose.yaml, value of SAPIO_EXEC_SERVER_API_KEY
keystore.location=(keystore file absolute path readable by Sapio server)
keystore.password=123456
```
The app or the Sapio Platform (BLS) server may need reboot to take effect.

## Deployment of Sapio Analytics Server (Load Balancing)
Before completing this step, you should complete setup of a single Sapio Analytics server and 
bake a docker image of all changes, including a different key and keystore, and any additional programs
you want installed.

At this time, Sapio is still testing transition of the load balanced stack from EC2 auto-scale to elastic containers. 
We will update this tutorial when the process is finalized.

For EC2 auto-scale, we have used dual-availability zone, network-level load balancer.
It is important that the load balancing is done by the expected "Network Load Balancer" in AWS or equivalent. 
The key is to navigate TCP traffic so that a single TCP session is always routed to the same analytic server instance. 
The AWS infrastructure accomplishes this by implementing the *Flow Hash Algorithm*.
More details here:
https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html

It is important that all instances of the auto-scale server has the same API key and the same keystore.

## Data Protection of Sapio Analytics Server
Besides the applications and their own static data stores installed, and the configurations such as API key and the keystore file,
there is no permanent data from Sapio that is being stored on any of these servers.
Some data will be temporarily stored in /tmp directory which will be deleted immediately when the TCP session is destroyed.

The "jailing" of download and upload directory is being enforced by sapio-analytic-server. 
When receiving a download request outside the jailed temporary directory, an error message will occur.

However, it is expected that a script is able to execute all executable programs, along with all scripts passed in.
Therefore, it is critical for information security that all modifications ot executable programs are vetted, and all 
Sapio server-side utilities and plugins are carefully examined and tested for security.

## Sapio App Setup
After deploying the Sapio Analytics Serer and configured the ClientSettings.properties, you will need to redirect the
binary locations in Sapio Analytics Settings to the correct locations installed in this container.

The default value of the baseline synergy are the correct values.
To verify, go to App Setup => Configuration Manager.
Navigate to "Analytics" menu
The values should be as follows.
### Native Analytics Form
1. python3
2. R
3. /opt/sapiosciences/rtranslator/translate.sh
### CRISPR Form
1. /data/indexes
2. GRCh38_latest_genomic

If any values are incorrect, please make adjustment as necessary and save.
The changes will take effect immediately.

To run a smoke test, Go to ELN create a table of data
Create a column "x" and a column "y"
Enter a data suitable for linear regression data such as (1, 1), (2, 3), (4, 10), (5 15)

Create an advanced curve viewer widget as the next entry, select "Polynomial" regression and pick the X and Y columns.
If the configuration is correct, the linear regression will be performed.
You will also see in the app log indicating an attempt to connect to analytics server.