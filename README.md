# 2021TGWDemo
Learn how AWS Transit Gateway works!

We will be setting up a Transit Gateway environment with a centralized Egress VPC for internet traffic out to the Internet.  In addition we do not want to allow traffic between the Production VPC and the Development VPC.


## Prework

1. Create an EC2 Key in Primary and Secondary Regions.
2. Change security on the 2 key files...  chmod 400 KEYNAME.pem 
3. Deploy tgwDemoRegion1.yaml CloudFormation template to the primary region.
4. Deploy tgwDemoRegion2.yaml CloudFormation template to the secondary region.
5. Connect with SSH to the Egress EC2 instance via its public IP address.
6. Copy EC2 Keys to the EC2 instance so it can be used to connect to other instances in the environment.
7. Change security on the 2 key files...  chmod 400 KEYNAME.pem


## 1. Setup Transit Gateway

This will setup the Transit Gateway and put the connections in place so it will be ready to route traffic between the VPCs.  At the end of this traffic will not be flowing through the VPC, but everything will be in place to allow this.

1. Connect to Primary Region.  (Tested with ca-central-1)
2. Create Transit GW
   * Name tag: tgw-region-1
   * Amazon Side ASN: 64612
   * DNS Support: Enabled
   * VPN ECMP Support: Enabled
   * Default Route Table Associate: Enabled
   * Default Route Table Propogation: Enabled
   * Multi-cast Support: Disabled
3. Create TG Attachment - **VPC0Egress**
   * Attachment Type: VPC
   * Attachment name tag: VPC0Egress
   * DNS Support: Yes
   * IPv6 Support: No
   * Select VPC: VPC0Egress
   * Subnet: VPC0 Egress Private
4. Create TG Attachment - **VPC1Production**
   * Attachment Type: VPC
   * Attachment name tag: VPC1Production
   * DNS Support: Yes
   * IPv6 Support: No
   * Select VPC: VPC1Production
   * Subnet: VPC1 Production Private
5. Create TG Attachment - **VPC2Development**
   * Attachment Type: VPC
   * Attachment name tag: VPC2Development
   * DNS Support: Yes
   * IPv6 Support: No
   * Select VPC: VPC2Development
   * Subnet: VPC2 Development Private
6. Open TGW Route Table and review routes.  You should now see 3 Routes
   * 10.51.0.0/22
   * 10.51.4.0/22
   * 10.51.8.0/22

## 2. Setup Traffic Flow throught the Transit Gateway

In this step, traffic will be redirected to the Transit Gateway and we will begin to see how the traffic flows using the default routing table.  At the end of this step we will have traffic between all VPCs working and egress to the internet setup.

1. Update Subnet Route Table **VPC0 RT Public**
   * 10.0.0.0/8 destination Transit Gateway and select the TGW
2. Update Subnet Route Table **VPC0 RT Private**
   * 10.0.0.0/8 destination Transit Gateway and select the TGW
3. Update Subnet Route Table **VPC1 RT Private**
   * 0.0.0.0/0 destination Transit Gateway and select the TGW
4. Update Subnet Route Table **VPC2 RT Private**
   * 0.0.0.0/0 destination Transit Gateway and select the TGW
5. Connect to **ec2Prod**
   * ssh -i "KEYNAME.pem" ec2-user@EC2PRODIP
6. Did this work?  No, lets check th esecurity group on the **VPC1Production** environment.
7. Add rule to the Security Group **NAME** for **SSH** with a source of **10.0.0.0/8**
8. Test it again...  Did it work?  Yes!
9. From **ec2Prod** - Ping **4.2.2.1**... Did it work... No.
10. The subnet route table is ok, let's look at the TGW Route Table... Notice that **0.0.0.0/0** target **VPC0Egress** route has not been created.  Create that route now.
11. From **ec2Prod** Try to ping **4.2.2.1** again.  It should now work.  If not, wait about a minute and it should start to work.
12. From **ec2Prod** run the following command:  **curl ifconfig.me**  This will provide the public IP used.  What IP is this?  Hint, check your NAT Gateway.

## 3. Using Routing Domains

As we discovered in Step 2, there was traffic flowing from the Production to the Development environments.  We want to prevent this from happening.  To do so, we are going to leverage Route Domains to stop traffic between these two environments.

1. Open the Transit Gateway Route Table.  Then delete the 3 associations linked to this route table.
2. Create Outbound Route Tables for the 3 Domains
   1. **Egress TGW RT**
   2. **Prod TGW RT**
   3. **Dev TGW RT**
3. Egress TGW RT
   1. Associate to **VPC0Egress**
   2. Static Routes to create
      * **10.51.4.0/22** to **VPC1Production** TGW Attachment
      * **10.51.8.0/22** to **VPC2Development** TGW Attachment
4. Prod TGW RT
   1. Associate to **VPC1Rpdocution**
   2. Static Routes to create
      * **0.0.0.0/0** to **VPC0Egress** TGW Attachment
5. Dev TGW RT
   1. Associate to **VPC0Development**
   2. Static Routes to create
      * **0.0.0.0/0** to **VPC0Egress** TGW Attachment
6. Connect to **ec2Prod1**
7. From **ec2Prod1** try: **Ping 4.2.2.1** - OK!
8. From **ec2Prod1** try: **ping ec2Dev1IPADDR** - Wait, I can ping the Development systems?
9. Looks like we need a blackhole to fix this issue up...
10. Update TGW Route Table **PROD TGW RT**
   * Add Route **10.51.8.0/22** to **blackhole**
11. Update TGW Route Table **PROD TGW RT**
   * Add Route **10.51.4.0/22** to **blackhole**
12. From **ec2Prod1** - Try to ping **ec2Dev1** - Nice!  It doesn't work now.
13. From **ec2Egress1** - Try to ping **ec2Prod1** - Nice it works!
14. From **ec2Egress1** - Try to ping **Ec2Dev1** - Nice it works!

## 4. Extra Credit - Let's add another region to our Transit Gateway

If you are interested in adding a 2nd region to your Transit Gateway environment, please ensure you have completed the provisioning of the CFN template in the 2nd region and have setup your EC2 key for remote access into the EC2 instance if desired.  At the end of this we will have setup a connection between the two transit Gateways and setup routes to send traffic out from the 2nd region through the egress VPC in Region 1.

1. Ensure new Key is created in the new region (US-East-1)
2. Region 2 - Create a new Transit GW.
   1. Name tag: tgw-region-2
   2. Amazon side ASN: 64613
   3. DNS Support - Enabled
   4. VPN ECMP Support - Enabled
   5. Default Route Table Association - Enabled
   6. Default Route Table Propagation - Enabled
   7. Multi-cast Support - Disabled
3. Region 2 - Connect TGW to VPC
   1. Attachment Name - USE1-Private1
   2. Attachment Name - VPC3Production
4. Region 2 - Connect TGW to Region 1
   1. Select USE-1 TGW
   2. Attachment type = Peering Connection
   3. Copy TGW ID from other region...
   4. Wait for Pending Acceptance
   5. Accept in Region 1 in the TGW Attachments.
5. Region 2 - Subnet Route Table **NAMEHERE** (0.0.0.0/0)
6. Region 2 - Update TGW Route Table
   1. Add Route **0.0.0.0/0** to **TGW Peering Connection**
12. Region 1 - Create Route Table
    1. Name:	Region2 TGW
13. Region 1 - Update VPC0 TGW 10.50.0.0/16 to Peering
14. Region 1 - Update **REGION2 TGW** Route Table - Add **0.0.0.0/0** to **VPC0 Egress**
15. Region 1 - Update Region1 Route Table
    1. Add Asociation: Select Peering Connection.  Might need to remove from the default Route Table.
    2. Add Route - **10.50.0.0/16**
16. Connect to **EC2 Egress** and ping **EC2 VPC3 ip**
17. Connect tp **EC2 Development** and prin **EC2 VPC3 ip** --> This works.  Need to add a blackhole.
18. Region 2 - Update **TGW Route** with **10.51.8.0/22** to **Blackhole**
19. Connect to **EC2 Production** and ping **EC2 VPC3 IP**


## Clean Up - Region 2
Only if Part 4 was setup.

1. Connect to the Second Region.
2. Open VPC - Transit Gateway Attachments
3. Delete both Attachments.
4. Delete the Transit Gateway.
5. Once TGW is deleted, open CloudFormation.
6. Select the stack deployed and choose to delete the stack.
7. Delete the EC2 Key.

## Clean Up - Region 1

1. Connect to the First Region.
2. Open VPC - Transit Gateway Attachments
3. Delete 3 Attachments. (VPC0Egress, VPC1Production and VPC2Development)
4. Delete the Transit Gateway.
5. Once TGW is deleted, open CloudFormation.
6. Select the stack deployed and choose to delete the stack.
7. Delete the EC2 Key.
