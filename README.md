## LRT Set up Instructions

**These instructions are a work in progress.**

The following is a set of instructions for provisioning a Kubernetes cluster in Packet,
and setting that cluster up for long run testing and monitoring of CoreDNS under load.

1. Install tools + setup
   1. Install python3
   1. Install terraform (e.g. `brew install terraform`)
   1. Clone kubespray github project (clone a stable release)
   1. Install all kubespray required python packages:
      * From kubespray dir: `sudo pip3 install -r requirements.txt`

1. Create kubespray project dir in `kubespray/inventory/`
   * e.g. `kubespray/inventory/my_lrt_instance`
   * cd in the dir. Further instructions are from this path

1. Provision Systems in Packet
   1. Define Packet API Token API env var: `export PACKET_AUTH_TOKEN="your-api-token"`
      * You can create an API token if you dont have one in https://app.packet.net/
   1. link the packet hosts to your project dir:
      * `ln -s ../../contrib/terraform/packet/hosts`
   1. copy the terraform Packet sample template files to your project dir:
      * `cp -LRp ../../contrib/terraform/packet/sample-inventory/ .`
   1. Adjust default terraform packet template settings in `cluster.tfvars` ...
      * set `cluster_name` to something other than "mycluster" e.g. "lrt-1.6.7" 
      * set project id (can be obtained from https://www.packet.com/developers/api/)
      * set public key path to blank if your keys are already deployed to the packet machines: `public_key_path = ""`
   1. Init and execute terraform
      * `terraform init ../../contrib/terraform/packet/`
      * `terraform apply -var-file=cluster.tfvars ../../contrib/terraform/packet`
   1. Disable swap and enable ip forwarding in all systems (Note: I think Kubespray might do this for us, so this step may be unnecessary):
      1. In one line: 
      ```
      for a in `cat terraform.tfstate | jq '.resources[].instances[].attributes.access_public_ipv4'`; do a=`echo $a | tr -d '"'`; ssh root@$a "sysctl -w net.ipv4.conf.all.forwarding=1 && sed -i 's/#net\.ipv4\.ip_forward=./net.ipv4.ip_forward=1/g' /etc/sysctl.conf && swapoff -a && sudo sed -i '/ swap / s/^/#/' /etc/fstab"; done;
      ```
      1. Or manually:
         1. Extract the provisioned IPs:
            1. `cat terraform.tfstate | jq '.resources[].instances[].attributes.access_public_ipv4'`
         1. Enable forwarding in each provisioned system
	        1. `sysctl -w net.ipv4.conf.all.forwarding=1` && `sed -i 's/#net\.ipv4\.ip_forward=./net.ipv4.ip_forward=1/g' /etc/sysctl.conf`
         1. Disable swap in each system
            1. `swapoff -a && sed -i '/\sswap\s/ s/^/#/' /etc/fstab`
`

1. Build K8s Cluster on Systems
   1. Run kubespray's ansible-playbook builder python script, passing space delimitd ips of provisioned packet servers e.g.
      * `CONFIG_FILE=hosts.yml python3 ../../contrib/inventory_builder/inventory.py 1.2.3.4 5.6.7.8 9.10.11.12 13.14.15.16`
   1. Adjust `hosts.yml` IPs, and master nodes.
   1. Adjust resulting ansible-playbook. Specifically in `group_vars/k8s-cluster/k8s-cluster.yml`
      * set pod and service cidrs (that do not collide with the packet 10.0.0.0/8 internal subnet)
      * remove node local dns option if desired
   1. Execute ansible-playbook e.g.
      * `sudo ansible-playbook -i hosts.yml ../../cluster.yml -b --become-user=root -v`
   1. Adjust CoreDNS deployment as needed (e.g. custom build, version, custom config)

1. Add Monitoring Infrastructure
   1. Apply prometheus yaml
      * prometheus-cluster-monitoring-config.yaml 
      * prometheus.yaml
      * node-exporter.yaml
   1. Add Grafana datasource

1. Start Test
   1. Apply dnsdrone.yaml, scale accordingly
   1. Apply kubernoisy.yaml

