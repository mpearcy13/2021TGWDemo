# 2021TGWDemo
Learn how AWS Transit Gateway works!


## Prework

1. Create an EC2 Key in Primary and Secondary Regions.
2. Deploy tgwDemoRegion1.yaml CloudFormation template to the primary region.
3. Deploy tgwDemoRegion2.yaml CloudFormation template to the secondary region.
4. Connect with SSH to the Egress EC2 instance via its public IP address.
5. Copy EC2 Keys to the EC2 instance so it can be used to connect to other instances in the environment.
6. Change security on the 2 key files...  chmod 400 KEYNAME.pem

## 1. Setup Transit Gateway
1. Connect to Primary Region.
2. Create Transit GW
  1. Name tag: tgw-ca-1
  2. Amazon Side ASN: 64612
  3. DNS Support: Enabled
  4. VPN ECMP Support: Enabled
  5. Default Route Table Associate: Enabled
  6. Default Route Table Propogation: Enabled
  7. Multi-cast Support: Disabled
3. Create TG Attachment - **VPC0Egress**
  1. Attachment Type: VPC
  2. Attachment name tag: VPC0Egress
  3. DNS Support: Yes
  4. IPv6 Support: No
  5. Select VPC: VPC0Egress
  6. Subnet: SnPbEg1
4. Create TG Attachment - **VPC1Production**
  1. Attachment Type: VPC
  2. Attachment name tag: VPC1Production
  3. DNS Support: Yes
  4. IPv6 Support: No
  5. Select VPC: VPC1Production
  6. Subnet: Select only subnet.
5. Create TG Attachment - **VPC2Development**
  1. Attachment Type: VPC
  2. Attachment name tag: VPC2Development
  3. DNS Support: Yes
  4. IPv6 Support: No
  5. Select VPC: VPC2Development
  6. Subnet: Select only subnet.
6. Open TGW Route Table and review routes.  You should now see 3 Routes
  1. 10.51.0.0/22
  2. 10.51.4.0/22
  3. 10.51.8.0/22

## 2. Setup Traffic Flow throught the Transit Gateway
1. Update Subnet Route Table **RT SnPbEg1**
   * 10.0.0.0/8 destination Transit Gateway and select the TGW
2. Update Subnet Route Table **RT SnPrEg1**
   * 10.0.0.0/8 destination Transit Gateway and select the TGW
3. Update Subnet Route Table **RT VPC1Production**
   * 0.0.0.0/0 destination Transit Gateway and select the TGW
4. Update Subnet Route Table **RT VPC2Development**
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

ISSUE - Need to create a new Route Table in CA1 for this instead of using the default one...

ISSUE - Update SG to include SSH Access to Private2 EC21


1. Deploy CFN for Part 2.
2. Ensure new Key is created in the new region (US-East-1)
3. Update USE1 - Subnet Route Table (0.0.0.0/0)
4. Update CA1 - Public0 Route Table (172.12.0.0/12)
5. Update CA1 - Private1 Route Table ()
6. Create Transit GW
   1. Name tag: tgw-use-1
   2. Amazon side ASN: 64613
   3. DNS Support - Enabled
   4. VPN ECMP Support - Enabled
   5. Default Route Table Association - Enabled
   6. Default Route Table Propagation - Enabled
   7. Multi-cast Support - Disabled
7. Connect TGW to VPC
   1. Attachment Name - USE1-Private1
8. Check Route Propogations...
9. Create Transit GW Attachment...
   1. Select USE-1 TGW
   2. Attachment type = Peering Connection
   3. Copy TGW ID from other region...
   4. Wait for Pending Acceptance
   5. Accept in CA Region.
10. Create Static Route in USE1
    1. CIDR 10.0.0.0/8 using new Peering Attachment.
11. Create STatic Route in CA1 - Outbound
    1. CIDR 172.12.1.0/24 using Peering ATtachment
12. Check Static Routes for VPC Private 1 and VPC Private 2 - Ok...
13. Add Blackhole Route - 172.12.0.0/16 to Private 2 Route

