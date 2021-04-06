
# Automating OCP 4.7 UPI Installation on Vsphere with Static IPs




## Introduction
In this post we will show how to automate an customize an OCP 4.7 UPI installation on Vsphere.
In the first part we will use govc,an open source command-line utility for performing administrative actions on a VMware vCenter or vSphere and in the second part we will deploy the cluster with Terraform.






## Prequisites:
### DNS, Loadbalancer and Webserver

Use the provided templates in files folder to configure the required services
```bash
dnf install -y named httpd haproxy
mkdir -p  /var/www/html/ignition
```

### Download necessary binaries

We need to install govc client, oc client and ocp installer and Terraform. This is how we proceed
```bash
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest-4.7/openshift-client-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest-4.7/openshift-install-linux.tar.gz
wget https://github.com/vmware/govmomi/releases/download/v0.24.0/govc_linux_amd64.gz
tar zxvf openshift-client-linux.tar.gz
tar zxvf openshift-install-linux.tar.gz
gunzip govc_linux_amd64.gz
rm -f *gz README.md
mv oc kubectl openshift-install /usr/local/
mv govc_linux_amd64 /usr/local/govc
dnf install -y dnf-plugins-core
dnf config-manager --add-repo https://rpm.releases.hashicorp.com/$release/hashicorp.repo
dnf config-manager --add-repo https://rpm.releases.hashicorp.com/$release/hashicorp.repo -y
```

### QuickStart
Modify install-config.yaml to fit your env and then:

If using govc do
```bash
git clone https://github.com/latouchek/OCP4-UPI-Vsphere-StaticIP.git  OCP4
mkdir deploy-OCP4
cd deploy-OCP4
cp ../OCP4/scripts/deploy-with-govc.sh .
cp ../OCP4/files/install-config.yaml .
sh deploy-with-govc.sh
```
If using Terraform do
```bash
git clone https://github.com/latouchek/OCP4-UPI-Vsphere-StaticIP.git OCP4
mkdir deploy-OCP4
cp -r OCP4/terraform/ deploy-OCP4/
cd deploy-OCP4
cp ../OCP4/scripts/deploy-with-terraform.sh .
cp ../OCP4/files/install-config.yaml .
sh deploy-with-terraform.sh
```


## Automating with govc:

### Export variables for govc (modify according to your env)

```bash
export OCP_RELEASE="4.7.4"
export CLUSTER_DOMAIN="vmware.lab.local"
export GOVC_URL='192.168.124.3'
export GOVC_USERNAME='administrator@vsphere.local'
export GOVC_PASSWORD='vSphere Pass'
export GOVC_INSECURE=1
export GOVC_NETWORK='VM Network'
export VMWARE_SWITCH='DSwitch'
export GOVC_DATASTORE='datastore1'
export GOVC_DATACENTER='Datacenter'
export GOVC_RESOURCE_POOL=yourcluster_name/Resources  ####default pool
export MYPATH=$(pwd)
export HTTP_SERVER="192.168.124.1"
export bootstrap_name="bootstrap"
export bootstrap_ip="192.168.124.20"

export HTTP_SERVER="192.168.124.1"
export master_name="master"
export master1_ip="192.168.124.7"
export master2_ip="192.168.124.8"
export master3_ip="192.168.124.9"

export worker_name="worker"
export worker1_ip="192.168.124.10"
export worker2_ip="192.168.124.11"
export worker3_ip="192.168.124.12"

export MASTER_CPU="4"
export MASTER_MEMORY="16384"   
export WORKER_CPU="4"
export WORKER_MEMORY="16384"

export ocp_net_gw="192.168.124.1"
export ocp_net_mask="255.255.255.0"
export ocp_net_dns="192.168.124.235"

```
### Create ignition files

Because bootstrap ignition is too big, it needs to be placed on a webserver and downloaded during the first boot. To acheve that, we create bootstrap-append.ign that points to the right file
 ```bash
 rm -f /var/www/html/ignition/*.ign
 rm -rf ${MYPATH}/openshift-install
 rm -rf ~/.kube
 mkdir ${MYPATH}/openshift-install
 mkdir ~/.kube
 cp install-config.yaml ${MYPATH}/openshift-install/install-config.yaml
 cat > ${MYPATH}/openshift-install/bootstrap-append.ign <<EOF
 {
   "ignition": {
     "config": {
       "merge": [
       {
         "source": "http://${HTTP_SERVER}:8080/ignition/bootstrap.ign"
       }
       ]
     },
     "version": "3.1.0"
   }
 }
 EOF
 openshift-install create ignition-configs --dir  openshift-install --log-level debug
 cp ${MYPATH}/openshift-install/*.ign /var/www/html/ignition/
 chmod o+r /var/www/html/ignition/*.ign
 restorecon -vR /var/www/html/
 cp ${MYPATH}/openshift-install/auth/kubeconfig ~/.kube/config

 ```

 ### Prepare CoreOS template

 Before downloading the ova we create coreos.json to modify Network mapping (Make sure GOVC_NETWORK is properly defined )

 ```bash
cat > coreos.json <<EOF
{
"DiskProvisioning": "flat",
"IPAllocationPolicy": "dhcpPolicy",
"IPProtocol": "IPv4",
"PropertyMapping": [
{
  "Key": "guestinfo.ignition.config.data",
  "Value": ""
},
{
  "Key": "guestinfo.ignition.config.data.encoding",
  "Value": ""
}
],
"NetworkMapping": [
{
  "Name": "VM Network",
  "Network": "${GOVC_NETWORK}"
}
],
"MarkAsTemplate": false,
"PowerOn": false,
"InjectOvfEnv": false,
"WaitForIP": false,
"Name": null
}
EOF
```
We can now download the image, apply the changes, import, tag the resulting VM as template and finaly create the bootsrap VM out of this template

```bash
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.7/latest/rhcos-vmware.x86_64.ova
govc import.ova -options=coreos.json -name coreostemplate rhcos-vmware.x86_64.ova
govc vm.markastemplate coreostemplate
govc vm.clone -vm coreos47  -on=false  bootstrap
```
IGN files need to be provided to Vsphere instance through guestinfo.ignition.config.data. We need to encode it in base64 before anything and change the previously created bootstrap VM:
```bash
bootstrap=$(cat openshift-install/append-bootstrap.ign | base64 -w0)
govc vm.change -e="guestinfo.ignition.config.data=${bootstrap}" -vm=${bootstrap_name}
govc vm.change -e="guestinfo.ignition.config.data.encoding=base64" -vm=${bootstrap_name}
govc vm.change -e="disk.EnableUUID=TRUE" -vm=${bootstrap_name}
```
To set Static IP to bootstrap we issue the following command:
```bash
govc vm.change -e="guestinfo.afterburn.initrd.network-kargs=ip=${bootstrap_ip}::${ocp_net_gw}:${ocp_net_mask}:${bootstrap_name}:ens192:off nameserver=${ocp_net_dns}" -vm=${bootstrap_name}
```
We are going to repeat those steps for Masters and Workers :
```bash
govc vm.clone -vm coreostemplate  -on=false  ${master_name}00.${CLUSTER_DOMAIN}
govc vm.clone -vm coreostemplate  -on=false  ${master_name}01.${CLUSTER_DOMAIN}
govc vm.clone -vm coreostemplate  -on=false  ${master_name}02.${CLUSTER_DOMAIN}

govc vm.change -c=${MASTER_CPU} -m=${MASTER_MEMORY} -vm=${master_name}00.${CLUSTER_DOMAIN}
govc vm.change -c=${MASTER_CPU} -m=${MASTER_MEMORY} -vm=${master_name}01.${CLUSTER_DOMAIN}
govc vm.change -c=${MASTER_CPU} -m=${MASTER_MEMORY} -vm=${master_name}02.${CLUSTER_DOMAIN}

govc vm.disk.change -vm ${master_name}00.${CLUSTER_DOMAIN} -size 120GB
govc vm.disk.change -vm ${master_name}01.${CLUSTER_DOMAIN} -size 120GB
govc vm.disk.change -vm ${master_name}02.${CLUSTER_DOMAIN} -size 120GB

master=$(cat openshift-install/master.ign | base64 -w0)

govc vm.change -e="guestinfo.ignition.config.data=${master}" -vm=${master_name}00.${CLUSTER_DOMAIN}
govc vm.change -e="guestinfo.ignition.config.data=${master}" -vm=${master_name}01.${CLUSTER_DOMAIN}
govc vm.change -e="guestinfo.ignition.config.data=${master}" -vm=${master_name}02.${CLUSTER_DOMAIN}

govc vm.change -e="guestinfo.ignition.config.data.encoding=base64" -vm=${master_name}00.${CLUSTER_DOMAIN}
govc vm.change -e="guestinfo.ignition.config.data.encoding=base64" -vm=${master_name}01.${CLUSTER_DOMAIN}
govc vm.change -e="guestinfo.ignition.config.data.encoding=base64" -vm=${master_name}02.${CLUSTER_DOMAIN}

govc vm.change -e="disk.EnableUUID=TRUE" -vm=${master_name}00.${CLUSTER_DOMAIN}
govc vm.change -e="disk.EnableUUID=TRUE" -vm=${master_name}01.${CLUSTER_DOMAIN}
govc vm.change -e="disk.EnableUUID=TRUE" -vm=${master_name}02.${CLUSTER_DOMAIN}

govc vm.change -e="guestinfo.afterburn.initrd.network-kargs=ip=${master1_ip}::${ocp_net_gw}:${ocp_net_mask}:${master_name}00.${CLUSTER_DOMAIN}:ens192:off nameserver=${ocp_net_dns}" -vm=${master_name}00.${CLUSTER_DOMAIN}
govc vm.change -e="guestinfo.afterburn.initrd.network-kargs=ip=${master2_ip}::${ocp_net_gw}:${ocp_net_mask}:${master_name}01.${CLUSTER_DOMAIN}:ens192:off nameserver=${ocp_net_dns}" -vm=${master_name}01.${CLUSTER_DOMAIN}
govc vm.change -e="guestinfo.afterburn.initrd.network-kargs=ip=${master3_ip}::${ocp_net_gw}:${ocp_net_mask}:${master_name}02.${CLUSTER_DOMAIN}:ens192:off nameserver=${ocp_net_dns}" -vm=${master_name}02.${CLUSTER_DOMAIN}

worker=$(cat /var/opsh/ocpddc-test/worker.ign | base64 -w0)
govc vm.clone -vm coreostemplate  -on=false  ${worker_name}00.${CLUSTER_DOMAIN}
govc vm.clone -vm coreostemplate  -on=false  ${worker_name}01.${CLUSTER_DOMAIN}
# govc vm.clone -vm coreostemplate  -on=false  ${worker_name}02.${CLUSTER_DOMAIN}

govc vm.change -e="guestinfo.ignition.config.data=${worker}" -vm=${worker_name}00.${CLUSTER_DOMAIN}
govc vm.change -e="guestinfo.ignition.config.data=${worker}" -vm=${worker_name}01.${CLUSTER_DOMAIN}
# govc vm.change -e="guestinfo.ignition.config.data=${worker}" -vm=${worker_name}02.${CLUSTER_DOMAIN}

govc vm.change -e="guestinfo.ignition.config.data.encoding=base64" -vm=${worker_name}00.${CLUSTER_DOMAIN}
govc vm.change -e="guestinfo.ignition.config.data.encoding=base64" -vm=${worker_name}01.${CLUSTER_DOMAIN}
# govc vm.change -e="guestinfo.ignition.config.data.encoding=base64" -vm=${worker_name}02.${CLUSTER_DOMAIN}

govc vm.change -e="disk.EnableUUID=TRUE" -vm=${worker_name}00.${CLUSTER_DOMAIN}
govc vm.change -e="disk.EnableUUID=TRUE" -vm=${worker_name}01.${CLUSTER_DOMAIN}
govc vm.change -e="disk.EnableUUID=TRUE" -vm=${worker_name}02.${CLUSTER_DOMAIN}

govc vm.change -e="guestinfo.afterburn.initrd.network-kargs=ip=${worker1_ip}::${ocp_net_gw}:${ocp_net_mask}:${worker_name}00.${CLUSTER_DOMAIN}:ens192:off nameserver=${ocp_net_dns}" -vm=${worker_name}00.${CLUSTER_DOMAIN}
govc vm.change -e="guestinfo.afterburn.initrd.network-kargs=ip=${worker2_ip}::${ocp_net_gw}:${ocp_net_mask}:${worker_name}01.${CLUSTER_DOMAIN}:ens192:off nameserver=${ocp_net_dns}" -vm=${worker_name}01.${CLUSTER_DOMAIN}
govc vm.change -e="guestinfo.afterburn.initrd.network-kargs=ip=${worker3_ip}::${ocp_net_gw}:${ocp_net_mask}:${worker_name}02.${CLUSTER_DOMAIN}:ens192:off nameserver=${ocp_net_dns}" -vm=${worker_name}02.${CLUSTER_DOMAIN}

govc vm.change -c=${WORKER_CPU} -m=${WORKER_MEMORY} -vm=${worker_name}00.${CLUSTER_DOMAIN}
govc vm.change -c=${WORKER_CPU} -m=${WORKER_MEMORY} -vm=${worker_name}01.${CLUSTER_DOMAIN}
govc vm.change -c=${WORKER_CPU} -m=${WORKER_MEMORY} -vm=${worker_name}02.${CLUSTER_DOMAIN}

govc vm.disk.change -vm ${worker_name}00.${CLUSTER_DOMAIN} -size 120GB
govc vm.disk.change -vm ${worker_name}01.${CLUSTER_DOMAIN} -size 120GB
govc vm.disk.change -vm ${worker_name}02.${CLUSTER_DOMAIN} -size 120GB
```
### Time to start the nodes:
```bash
govc vm.power -on=true bootstrap
govc vm.power -on=true ${master_name}00.${CLUSTER_DOMAIN}
govc vm.power -on=true ${master_name}01.${CLUSTER_DOMAIN}
govc vm.power -on=true ${master_name}02.${CLUSTER_DOMAIN}
govc vm.power -on=true ${worker_name}00.${CLUSTER_DOMAIN}
govc vm.power -on=true ${worker_name}01.${CLUSTER_DOMAIN}
govc vm.power -on=true ${worker_name}02.${CLUSTER_DOMAIN}
```
### Wait for the installation to complete

```bash
openshift-install --dir=openshift-install wait-for bootstrap-complete
openshift-install --dir=openshift-install wait-for bootstrap-complete > /tmp/bootstrap-test 2>&1
grep safe /tmp/bootstrap-test > /dev/null 2>&1
if [ "$?" -ne 0 ]
then
	echo -e "\n\n\nERROR: Bootstrap did not complete in time!"
	echo "Your environment (CPU or network bandwidth) might be"
	echo "too slow. Continue by hand or execute cleanup.sh and"
	echo "start all over again."
	exit 1
fi
echo -e "\n\n[INFO] Completing the installation and approving workers...\n"
for csr in $(oc -n openshift-machine-api get csr | awk '/Pending/ {print $1}'); do oc adm certificate approve $csr;done
sleep 180

for csr in $(oc -n openshift-machine-api get csr | awk '/Pending/ {print $1}'); do oc adm certificate approve $csr;done

openshift-install --dir=openshift-install wait-for install-complete --log-level debug       

```

## Automating with Terraform
 * To be continued
