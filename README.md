# transit-palo-alto
AWS Transit Gateway and Palo Alto architecture

# Pre-Requisites
The following steps should be performed in order listed here.  These instructions will deploy an architecture that looks like:

## Deploy VPCs
1. Deploy the Management VPC using the vpc-singleroutetable.json (see Resources section below for parameter descriptions)
2. Deploy the Application VPC using the vpc-singleroutetable.json (see Resources section below for parameter descriptions)
3. Deploy Cisco VPC (**Bishnu**)
4. Deploy Transit Gateway (**Jayendra**)

## Deploy Additional AWS Foundational Resoures
1. Create KMS keys for Secrets Manager, S3, and EBS using the kms.json (see Resources section below for paramater descriptions)

## Deploy Bastion
1. Deploy Bastion (**Jayendra**)

# Transit Gateway Configuration
The following steps can be performed in any order, generally you'll need to create a VPN attachment to the Cisco plus VPC attachments to all VPCs

## VPC Peering Setup for Management Traffic from Bastion
This section will walk through establish VPC attachments to allow the bastion to manage devices in the Application and Cisco VPCs.
1. Create VPC attachments (**Jayendra**)

## VPN Egress Traffic Setup
This section will walk through creating the Cisco VPC, Cisco CSR, VPN attachment, attaching to the TGW, creating a route table for the attachment, and configuration updates on the Cisco CSR to support the egress traffic network pattern.

## Deploy Cisco VPC, CSR instance and VPN attachment
1. Deploy Cisco VPC using the transit-pa-ciscovpc.json ( See Resources section below for the parameters )
2. Deploy the Cisco CSR instance using the transit-pa-csr ( See Resources section below for parameters )
3. Deploy the VPN attachment using the VPN-attachment.json ( See Resources section below for the parameters )

## Cisco CSR necessary manual configuration

1. Configure the IPSEC VPN tunnel with the TGW
2. Configure the NAT for egress traffic

## TGW necessary manual configuration

Since the automation of TGW route-association and route-propagation is not currently supported for the VPN attachments, please perform these steps manually

1. Remove the association of the VPN-attachment from the TGW default route table
2. Associate the VPN attachment with the TGW VPN route table deployed via CFT ( The CFT will deploy the VPN route table named prefix-vpn-route-table-csr )

# Resources
- vpc-singleroutetable.json: Deploys a VPC with a single route table, 6 subnets (all with the same netmask) across three availability zones, 3 of the subnets are subnets for the VPC attachments to the TGW
  - nameprefix: All name tags will start with this prefix
  - cidr: CIDR for the VPC
  - subnetBits: number of subnet bits for the subnets within the VPC, used by Fn::Cidr intrinsic function to create subnet CIDRs
  - createIgw: Whether or not to create an IGW, if you need a temporary one, you can provision with this true and then perform a stack update with this value to false to remove it later
- kms.json
  - service: The AWS service this key should be used with
  - nameprefix: All name tags will start with this prefix
- transit-pa-ciscovpc.json: Deploys a VPC with two route tables, 4 subnets (all with the same netmask) on the same availability zone, 3 of the subnets are dedicated to CISCO and 1 subnet is reserved for future use.
  - nameprefix: All name tags will start with this prefix
  - cidr: CIDR for the VPC
    - createIgw: Whether or not to create an IGW, if you need a temporary one, you can provision with this true and then perform a stack update with this value to false to remove it later
	
- transit-pa-csr: Deploys a Cisco CSR with the three ENI's, one on each subnet defined by the transit-pa-ciscovpc.json CFT. Elastic IP is only attached to the Egress interface of the CSR
  - CSRInstanceType: Instance type for the CSR instance
  - CSRImageId: AMI ID for the pay-as-you go instance for the region you are deploying the instance on
  - CSRVpcId: VPC ID of the VPC created using transit-pa-ciscovpc.json CFT
  - Subnets: Respective Subnet ID's created using transit-pa-ciscovpc.json CFT
  - SSHKey: SSH Key created previously

- VPN-attachment.json: Deploys a customer gateway, VPN attachement with the Cisco CSR and TGW route table for VPN attachement
  - nameprefix: All name tags will start with this prefix
  - TGW: The ID of the transit gateway previously deployed using the transit-pa-tgw.json
  - BGPASN: BGP ASN Number for the Cisco side
  - RemoteIP: IP address of the Cisco side
 

