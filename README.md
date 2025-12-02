Zextras Carbonio/Zimbra Let's Encrypt Certificate Deployment & Automation Fix
This guide provides the necessary steps to manually fix the common ERROR: Unable to validate certificate chain when deploying a Let's Encrypt certificate on Zextras Carbonio (or modern Zimbra installations) and includes a permanent automation script to handle renewals.

1. The Manual Fix: "Unable to Get Issuer Certificate"
The error occurs because the zmcertmgr utility often requires the ISRG Root X1 to be explicitly included in the CA chain file (commercial_ca.crt) for verification to pass. Certbot's chain.pem only contains the intermediate.

Run the following steps as root to correctly stage the certificate files:

Bash

# Set variables for easy use
CERT_PATH="/opt/zextras/common/certbot/etc/letsencrypt/live/zangomediclinic.com"
COMM_DIR="/opt/zextras/ssl/carbonio/commercial"

# 1. Navigate to the commercial staging directory
cd $COMM_DIR

# 2. Copy the private key and main certificate
cp $CERT_PATH/privkey.pem commercial.key
cp $CERT_PATH/cert.pem commercial.crt
chmod 640 commercial.key

# 3. Create the robust commercial_ca.crt (Intermediate + Root)
# a. Start with the Intermediate chain (chain.pem)
cat $CERT_PATH/chain.pem > commercial_ca.crt
# b. Append the ISRG Root X1 (Fixes the validation error)
curl -k https://letsencrypt.org/certs/isrgrootx1.pem.txt >> commercial_ca.crt

# 4. Correct ownership
chown zextras:zextras commercial.key commercial.crt commercial_ca.crt
2. Manual Deployment (Verification)
Once the files are staged, deploy them as the zextras user:

Bash

sudo su - zextras
/opt/zextras/bin/zmcertmgr deploycrt comm $COMM_DIR/commercial.crt $COMM_DIR/commercial_ca.crt
exit
3. Service Restart
Restart the relevant systemd targets to apply the certificate:

Bash

sudo systemctl restart carbonio-directory-server.target carbonio-appserver.target carbonio-proxy.target carbonio-mta.target
⚙️ Automation: The Renewal Hook Script
Since Certbot renews certificates every 60–90 days, we must automate the deployment. We will use a Certbot hook script that runs after a successful renewal.

1. Create the Deployment Script
Create a script named zmcertmgr_deploy.sh in the /etc/letsencrypt/renewal-hooks/deploy directory (as root):

Bash

sudo nano /etc/letsencrypt/renewal-hooks/deploy/zmcertmgr_deploy.sh
Paste the following script content:

Bash

#!/bin/bash

# Zextras/Carbonio Certbot Deployment Hook
# Script to run after successful certificate renewal.

# --- Configuration ---
DOMAIN="zangomediclinic.com"
CERT_LIVE_DIR="/opt/zextras/common/certbot/etc/letsencrypt/live/$DOMAIN"
COMMERCIAL_DIR="/opt/zextras/ssl/carbonio/commercial"
LOG_FILE="/var/log/carbonio/certbot_deploy.log"
DATE_STAMP=$(date "+%Y-%m-%d %H:%M:%S")

# Check if the required files exist
if [ ! -f "$CERT_LIVE_DIR/fullchain.pem" ]; then
    echo "$DATE_STAMP - ERROR: Certificate files not found at $CERT_LIVE_DIR. Exiting." >> "$LOG_FILE"
    exit 1
fi

echo "$DATE_STAMP - Renewal successful. Starting Zextras deployment process." >> "$LOG_FILE"

# --- Deployment Preparation (Run as root) ---

# 1. Stage the files in the commercial directory.
# Note: Certbot's fullchain.pem is not sufficient for zmcertmgr.
# We explicitly combine chain.pem (Intermediate) and ISRG Root X1.
cp "$CERT_LIVE_DIR/privkey.pem" "$COMMERCIAL_DIR/commercial.key"
cp "$CERT_LIVE_DIR/cert.pem" "$COMMERCIAL_DIR/commercial.crt"

# 2. Create the robust commercial_ca.crt (Intermediate + Root)
cat "$CERT_LIVE_DIR/chain.pem" > "$COMMERCIAL_DIR/commercial_ca.crt"
# Append ISRG Root X1 (crucial for zmcertmgr verification)
curl -k https://letsencrypt.org/certs/isrgrootx1.pem.txt >> "$COMMERCIAL_DIR/commercial_ca.crt"

# 3. Set correct ownership and permissions
chown zextras:zextras "$COMMERCIAL_DIR/commercial.key" "$COMMERCIAL_DIR/commercial.crt" "$COMMERCIAL_DIR/commercial_ca.crt"
chmod 640 "$COMMERCIAL_DIR/commercial.key"

echo "$DATE_STAMP - Files staged and permissions set." >> "$LOG_FILE"

# --- Deployment Execution (Run as zextras) ---

# 4. Use su to run the deployment command as zextras
su - zextras -c "/opt/zextras/bin/zmcertmgr deploycrt comm $COMMERCIAL_DIR/commercial.crt $COMMERCIAL_DIR/commercial_ca.crt >> $LOG_FILE 2>&1"

# 5. Restart services via systemd targets
echo "$DATE_STAMP - Deployment complete. Restarting services..." >> "$LOG_FILE"
systemctl restart carbonio-directory-server.target carbonio-appserver.target carbonio-proxy.target carbonio-mta.target

echo "$DATE_STAMP - Deployment process finished." >> "$LOG_FILE"
2. Set Permissions
Make the script executable:

Bash

sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/zmcertmgr_deploy.sh
3. Test the Automation
Run a simulated renewal to test the new hook:

Bash

sudo /opt/zextras/libexec/certbot renew --dry-run
If the dry run completes successfully and the deployment steps execute (check the output or the $LOG_FILE), your automated renewal is set!
