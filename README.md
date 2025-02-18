# F5 Migration SOP

## Introduction
This document provides a step-by-step Standard Operating Procedure (SOP) for migrating an F5 BIG-IP configuration from an old device to a new device. This guide is designed to be simple and easy to follow for all users.

## Problem Statement
Organizations often need to migrate F5 BIG-IP configurations due to hardware upgrades, cloud transitions, or disaster recovery. However, manual migration can be error-prone and time-consuming.

## Justification
To ensure a smooth migration, this SOP provides:
- A structured approach to migrating configurations.
- A backup and verification mechanism.
- A tested and automated process for merging configurations.

## Steps for Migration

### Step 1: Prepare the New F5 Device
1. License and provision the new F5 device.
2. Configure basic settings:
   - Management IP
   - Hostname
   - Interface settings
   - Admin password

### Step 2: Backup and Save UCS File from Old F5
1. Log into TMSH:
   ```sh
   ssh admin@<old-f5-ip>
   ```
2. Reset the master key to encrypt passwords correctly:
   ```sh
   modify sys crypto master-key prompt-for-password
   ```
3. Verify the master key (optional):
   ```sh
   f5mku -K
   ```
4. Save the configuration:
   ```sh
   save sys config
   ```
5. Create a UCS backup file:
   ```sh
   save sys ucs /var/local/ucs/<backup_filename>.ucs
   ```

### Step 3: Transfer UCS File to New F5
1. Copy the UCS file to your local machine:
   ```sh
   scp admin@<old-f5-ip>:/var/local/ucs/<backup_filename>.ucs .
   ```
2. Copy the UCS file to the new F5:
   ```sh
   scp <backup_filename>.ucs admin@<new-f5-ip>:/var/local/ucs/
   ```

### Step 4: Restore UCS on New F5
1. Log into the new F5 device:
   ```sh
   ssh admin@<new-f5-ip>
   ```
2. Restore the UCS backup:
   ```sh
   load sys ucs /var/local/ucs/<backup_filename>.ucs
   ```
3. Set the master key to match the old F5:
   ```sh
   modify sys crypto master-key prompt-for-password
   ```
4. Verify the master key (optional):
   ```sh
   f5mku -K
   ```
5. Save the configuration:
   ```sh
   save sys config
   ```

### Step 5: Extract UCS for Custom Migration
1. Create a directory and extract UCS contents:
   ```sh
   mkdir extracted_ucs && cd extracted_ucs
   tar -xvzf ../<backup_filename>.ucs
   ```
2. Identify required configuration files:
   - `bigip.conf`
   - `bigip_base.conf`
   - `bigip_local.conf`
   - `ssl.crt/`
   - `ssl.key/`

### Step 6: Modify and Merge Configuration
1. Move `bigip.conf` to `/var/tmp/`:
   ```sh
   mv extracted_ucs/config/bigip.conf /var/tmp/bigip.conf
   ```
2. Perform a dry run to verify the configuration:
   ```sh
   tmsh load sys config merge file /var/tmp/bigip.conf verify
   ```
3. If no errors, merge the configuration:
   ```sh
   tmsh load sys config merge file /var/tmp/bigip.conf
   ```
4. Save the configuration:
   ```sh
   save sys config
   ```

### Step 7: Migrate SSL Certificates
1. Move SSL certificates and keys:
   ```sh
   mv extracted_ucs/config/ssl.crt/f5_api_com.crt /config/ssl/ssl.crt/
   mv extracted_ucs/config/ssl.key/f5_api_com.key /config/ssl/ssl.key/
   ```
2. Register the SSL certificate and key:
   ```sh
   tmsh create sys file ssl-cert /Common/f5_api_com.crt source-path file:///config/ssl/ssl.crt/f5_api_com.crt
   tmsh create sys file ssl-key /Common/f5_api_com.key source-path file:///config/ssl/ssl.key/f5_api_com.key
   ```

### Step 8: Validate and Restart Services
1. Restart the configuration service:
   ```sh
   tmsh restart sys service restjavad
   tmsh restart sys service restnoded
   ```
2. Verify MCPD status:
   ```sh
   tmsh show sys mcp
   ```
3. Save final configuration:
   ```sh
   save sys config
   ```

### Step 9: Verify Cloud Failover Extension (CFE)
1. Check current CFE configuration:
   ```sh
   curl -su admin: -X GET http://localhost:8100/mgmt/shared/cloud-failover/declare | jq .
   ```
2. Update the failover label and next-hop addresses:
   ```sh
   vi cfe.json
   ```
   Update with:
   ```json
   {
     "class": "Cloud_Failover",
     "environment": "aws",
     "failoverRoutes": {
       "defaultNextHopAddresses": {
         "items": [
           "10.36.48.180",
           "10.36.49.228"
         ]
       }
     }
   }
   ```
3. Apply the new CFE configuration:
   ```sh
   curl -su admin: -X POST -d @cfe.json http://localhost:8100/mgmt/shared/cloud-failover/declare | jq .
   ```

## Conclusion
Following this SOP ensures a smooth, error-free migration of F5 BIG-IP configurations. If any issues arise, verify logs and configurations before proceeding with changes.
