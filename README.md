# transit-palo-alto
AWS Transit Gateway and Palo Alto architecture

## VPN Egress Traffic Setup
This section will walk through creating the VPN attachment, attaching to the TGW, creating a route table for the attachment, and configuration updates on the Cisco CSR to support the egress traffic network pattern.
1. Create the VPN attachment 
2. Confgure Cisco CSR

# Pre-Requisites
The following steps should be performed in order listed here.  These instructions will deploy an architecture that looks like:

## Deploy Cisco VPC, CSR instance and VPN attachment
1. Deploy Cisco VPC using the transit-pa-ciscovpc.json ( See Resources section below for the parameters )
2. Deploy the Cisco CSR instance using the transit-pa-csr ( See Resources section below for parameters )
3. Deploy the VPN attachment using the VPNAttachment.json ( See Resources section below for the parameters )

## Cisco CSR necessary manual configuration

1. Configure the IPSEC VPN tunnel with the TGW
2. Configure the NAT for egress traffic

## TGW necessary manual configuration

Since the automation of TGW route-association and route-propagation is not currently supported for the VPN attachments, please perform these steps manually

1. Remove the association of the VPN-attachment from the TGW default route table
2. Associate the VPN attachment with the TGW VPN route table deployed via CFT ( The CFT will deploy the VPN route table named prefix-vpn-route-table-csr )

# Resources
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

- VPNAttachment.json: Deploys a customer gateway, VPN attachement with the Cisco CSR and TGW route table for VPN attachement
  - nameprefix: All name tags will start with this prefix
  - TGW: The ID of the transit gateway previously deployed using the transit-pa-tgw.json
  - BGPASN: BGP ASN Number for the Cisco side
  - RemoteIP: IP address of the Cisco side
 
