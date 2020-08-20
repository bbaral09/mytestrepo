# transit-palo-alto
AWS Transit Gateway and Palo Alto architecture

# Pre-Requisites
The following steps should be performed in order listed here.  These instructions will deploy an architecture that looks like:

## Deploy VPCs
1. Deploy the Management VPC using the mgmt-vpc.json (see Resources section below for parameter descriptions)
2. Deploy the Application VPC using the vpc-tworoutetables.json (see Resources section below for parameter descriptions)
3. Deploy Cisco VPC using the transit-pa-cisco-vpc.json (See Resources section below for the parameters )
4. Deploy Transit Gateway and VPC attachments using the transit-pa-tgw.json (See Resources section below for the parameters )
5. Deploy VPC attachments for Management, Application, Cisco VPCs using vpc-attachment.json (**Marion**)
6. Setup route propogation **Marion**

## Deploy Additional AWS Foundational Resoures
1. Create KMS keys for Secrets Manager, S3, and EBS using the kms.json (see Resources section below for paramater descriptions)

## Allow Management Traffic from Bastion
This section will walk through establish VPC attachments to allow the bastion to manage devices in the Application and Cisco VPCs.
1. Deploy Bastion using the transit-pa-bastion.json (See Resources section below for the parameters)
2. Setup Management Routes
    - Deploy the vpcroutes.json for the Management VPC's route to the Cisco VPC, use the Cisco VPC CIDR as the destination CIDR
    - Deploy the vpcroutes.json for the Management VPC's route to the Application VPC, use the Application VPC CIDR as the destination CIDR

# Cisco and VPN for Egress Traffic Setup 

This section will walk through creating the Cisco VPC, Cisco CSR, VPN attachment, attaching to the TGW, creating a route table for the attachment, and configuration updates on the Cisco CSR to support the egress traffic network pattern.

## Deploy CSR instance and VPN attachment

1. Deploy the Cisco CSR instance using the transit-pa-cisco-csr (See Resources section below for parameters)
2. Deploy the VPN attachment using the transit-pa-vpn-attachment.json (See Resources section below for the parameters)
3. Setup VPC route table for Cisco so that private subnets (management and internal) are accessible using transit-pa-cisco-vpc-routes.json

## Cisco CSR necessary manual configuration

1. Configure IP adddressing on any additional interfaces (External and Internal). The default (Management) interface will be autoconfigured
2. Configure a default static route pointed to the default gateway of the external interface. This will allow internet access to the external interface via the 'Elastic IP'
3. Configure the IPSEC VPN tunnel with the TGW. The configuration template can be downloaded from the AWS console 
4. Configure the access-list and NAT for egress traffic. Any egress traffic should be NAT'ed to the external interface

Detailed steps :

 1.Configure the external interface

transit-pa-csr#conf t

transit-pa-csr(config)#int gi2

transit-pa-csr(config-if)#description external-interface

transit-pa-csr(config-if)#ip address 172.1.3.45 255.255.255.0 <- We can get this IP from the EC2 instance (Eth1 interface) on AWS console

transit-pa-csr(config-if)#ip nat outside

transit-pa-csr(config-if)# no shut

transit-pa-csr(config-if)#end

2. Configure default route via the external interface and specific management route via the management interface:

transit-pa-csr#conf t

transit-pa-csr(config)#ip route 0.0.0.0 0.0.0.0 GigabitEthernet2 172.1.3.1

transit-pa-csr(config)ip route 172.0.0.0 255.255.0.0 GigabitEthernet1 172.1.2.1 <-Please note the cidr of management VPC before creating this route

transit-pa-csr(config)#end

3. Configure IPSEC tunnels and BGP sessions using the template downloaded from the AWS console

The template creates two IPSEC tunnels and two BGP sessions over those IPSEC tunnels.

The whole configuration can loaded in one go after removing the comment lines that begin with an exclamation sign (" ! " ) are removed. 

Everything can be copied/pasted as is, except for the following section
 Please replace the section local-address <interface_name/private_IP_on_outside_interface> with local-address GigabitEthernet2

However, it is better to load the configuration in small modules for better understanding of the configurations

4. Configure the required access-list and NAT for egress traffic

transit-pa-csr#conf t
transit-pa-csr(config)#ip access-list extended WEBSUBET
transit-pa-csr(config-ext-nacl)#permit ip 172.2.0.0 0.0.255.255 any
transit-pa-csr(config-ext-nacl)#end

Configure the NAT

transit-pa-csr#conf t
transit-pa-csr(config)#ip nat inside source list WEBSUBET interface GigabitEthernet2 overload <-This configures daynamic NAT/PAT
transit-pa-csr(config)#end

Configure tunnel interfaces as nat inside interfaces and external interface(gi2) as nat outside interface

transit-pa-csr#conf t

transit-pa-csr(config)#int tun1
transit-pa-csr(config-if)#ip nat inside

transit-pa-csr(config)#int tun2
transit-pa-csr(config-if)#ip nat inside

transit-pa-csr(config)#int gi2
transit-pa-csr(config-if)#ip nat outside


## TGW necessary manual configuration

Since the automation of TGW route-association and route-propagation is not currently supported for the VPN attachments, please perform these steps manually

1. Remove the association of the VPN-attachment from the TGW default route table
2. Associate the VPN attachment with the TGW VPN route table deployed via CFT ( The CFT will deploy the VPN route table named transit-gw-cisco-vpn-route-domain )
3. Any necessary routing configuration (association and propagation) on the TGW has to be done manually


# Resources
- mgmt-vpc.json (**Marion**)

- vpc-tworoutetables.json (**Marion**)

- vpc-singleroutetable.json: Deploys a VPC with a single route table, 6 subnets (all with the same netmask) across three availability zones, 3 of the subnets are subnets for the VPC attachments to the TGW
  - nameprefix: All name tags will start with this prefix
  - cidr: CIDR for the VPC
  - createIgw: Whether or not to create an IGW, if you need a temporary one, you can provision with this true and then perform a stack update with this value to false to remove it later

- transit-pa-tgw.json (**Marion**)


- vpc-singleroutetable.json: Deploys a VPC with a single route table, 6 subnets (all with the same netmask) across three availability zones, 3 of the subnets are subnets for the VPC attachments to the TGW
  - nameprefix: All name tags will start with this prefix
  - cidr: CIDR for the VPC
  - subnetBits: number of subnet bits for the subnets within the VPC, used by Fn::Cidr intrinsic function to create subnet CIDRs
  - createIgw: Whether or not to create an IGW, if you need a temporary one, you can provision with this true and then perform a stack update with this value to false to remove it later

- kms.json
  - service: The AWS service this key should be used with
  - nameprefix: All name tags will start with this prefix
  
 - vpcroutes.json
  - nameprefix: All name tags will start with this prefix, and also used to grab the subnet and route table IDs from Parameter Store with the expected naming convention from the vpcsingleroutetable.json: $nameprefix-vpc and $nameprefix-routetable
  - tgwssm: Name of the SSM parameter that holds the Transit Gateway ID
  - destinationcdir: Destination CIDR for the route
 
 - transit-pa-ciscovpc.json: Deploys a VPC with two route tables and 6 subnets. 3 of the subnets (/24 each on the same AZ) are dedicated to Cisco CSR and other 3 subnets(/28 each on different AZ) are reserved for TGW attachment
  - nameprefix: All name tags will start with this prefix
  - cidr: CIDR for the VPC
    - createIgw: Whether or not to create an IGW, if you need a temporary one, you can provision with this true and then perform a stack update with this value to false to remove it later
	
- transit-pa-cisco-csr: Deploys a Cisco CSR with the three ENI's, one on each subnet defined by the transit-pa-ciscovpc.json CFT. Elastic IP is only attached to the Egress interface of the CSR
  - CSRInstanceType: Instance type for the CSR instance
  - CSRImageId: AMI ID for the pay-as-you go instance for the region you are deploying the instance on
  - CSRVpcId: VPC ID of the VPC created using transit-pa-ciscovpc.json CFT
  - Subnets: Respective Subnet ID's created using transit-pa-ciscovpc.json CFT
  - SSHKey: SSH Key created previously

- transit-pa-vpn-attachment.json: Deploys a customer gateway, VPN attachement with the Cisco CSR and TGW route table for VPN attachement
  - nameprefix: All name tags will start with this prefix
  - TGW: The ID of the transit gateway previously deployed using the transit-pa-tgw.json
  - BGPASN: BGP ASN Number for the Cisco side
  - RemoteIP: IP address of the Cisco side

- transit-pa-vpn-attachment.json: Deploys a route to the management VPC in the routing private routing table of Cisco VPC. This will allow management of Cisco CSR from the Bastion host
  - nameprefix: All name tags will start with this prefix
  - TGW: The ID of the transit gateway previously deployed using the transit-pa-tgw.json
  - Privateroutetable: ID of private route table in the Cisco vpc
