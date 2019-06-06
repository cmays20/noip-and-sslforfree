## Setup domain name and wildcard SSL certificate
These directions will document how to purchase a domain name through noip.com and utilize their DNS services to obtain a free wildcard SSL certificate through sslforfree.com.

### Domain name purchase
To check for domain name availability, navigate to https://www.noip.com/domains.
![noip registration](assets/noip-domain-reg.png?raw=true)
Once you find one, add it to cart.  As of this writing, .com domains are $15/yr.
To manage the domain, you will also need their Plus Managed DNS service, https://www.noip.com/managed-dns.
![noip dns](assets/noip-managed-dns.png?raw=true)
This service is $29.95/yr, but will save you $10 on the domain registration when purchased together.

Another option they give you when purchasing a domain is to have private registration: https://www.noip.com/domains/why-private-registration.  This is a $9.95 add on.  The link above explains why you may want to do this option.

At this point,  you can make the purchase and get started.

### Obtain wildcard certificate
A wild card certificate makes life very easy.  The certificate would cover \*.yourdomain.com. This mean you can prepend many sub-domains and they would all be covered by the same certificate, e.g. jenkins.yourdomain.com, sonar.yourdomain.com.

The site https://www.sslforfree.com can be used to obtain a legit wildcard certificate for free.
![sslforfree](assets/sslforfree.png?raw=true)
In the textfield enter:
```
*.yourdomain.com yourdomain.com
```
Then click "Create Free SSL Certificate".  You will need to manually verify that you own the domain.  Click "Manually Verify Domain".  This will bring up the following:
![sslforfree verify](assets/sslforfree-manual-verify.png?raw=true)
The TXT records will need to be entered in noip.com to verify ownership. Login to your noip.com account and go to My Services -> DNS Records.  Find yourdomain.com in the list and click Modify.  Under Advanced Records, you will see TXT, click on that. This will bring you to the following:
![noip txt records](assets/noip-txt-update.png?raw=true)
To enter the two required records, you have to format each record in quotes with a space between the two records. Click "Update" to make the changes. Once this is done, go back to sslforfree and click the "Download SSL Certificate" button. If everything has gone well, this will provide you with 3 certifates (site key, ca-chain and private key) and a way to download them in a zip.

### Using domain and certifate with Univeral Installer
The Univeral Installer can utilize certificates stored in the Amazon Certificate Manager. Through the AWS Console, open "Certificate Manager". NOTE: The certificate manager is region aware, so make sure you have the correct region open. Then click on "Import a Certificate".
![acm import](assets/acm-import.png?raw=true)
Copy and paste the 3 certifates into the correct fields, then click "Review and import". This should verify that you entered the info correctly and allow you to review the wildcard certificate. Finish the import to get the ARN for your new entry. The ARN will be used in the main.tf of the Universal Installer like so:
```
masters_acm_cert_arn = "arn:aws:acm:us-west-2:someaccount:certificate/someidentifier"
```
This will tell the installer to setup the master ELB with your certificate. You can now bring up your cluster. Once the cluster is up and you have the output, take the master ELB address and log back in to https://noip.com.  You need to create a CNAME record that points to the ELB DNS record. Go to My Services -> DNS Records.  Click "Add A Hostname":
![noip add hostname](assets/noip-add-hostname.png?raw=true)
Make sure to select your domain name in the drop down in the upper right. In the Hostname field, you only need to type the sub-domain. For the Master ELB, I would recommend "www".  Then for Hostname Type, click the CNAME radio button.  The Target should be filled in with the DNS record for the Master ELB.  Click "Add Hostname" after this is all entered. It should only take a minute for your changes to go into effect. To confirm it worked, navigate to www.yourdomain.com and see if the site comes up.
> NOTE: the universal installer has an option for the public ELB as well: public_agents_acm_cert_arn. However, as of this writing it didn't work and a bug ticket has been submitted.

### Using domain and certificate with Edge LB
To use your new cerificate with an LB (HAProxy), you will need to create a file with all three keys in it. The order needs to be:
1. The Certificate for your domain
2. The CA Chain
3. The Private Key
You can do this via the command line like this:
```
cat certificate.crt intermediates.pem private.key > ssl-certs.pem
```
Edge LB utilizes file based secrets to obtain the certificate. Enter the secret with the following:
```
dcos security secrets create -f ssl-certs.pem dcos-edgelb/certificate
```
An example pool configuration can be seen here that ties all of this together (certificates and VHosts): [pool-edgelb-cicd.json](assets/pool-edgelb-cicd.json)

### Using domain and certificate with Marathon LB
You will need the ssl-certs.pem file created in the previous section. However, you will need to replace all of the newlines with \n.
```
sed -e ':a' -e 'N' -e '$!ba' -e 's/\n/\\n/g' ssl-certs.pem > ssl-certs-marathon.pem
```
You can now take the text of the output file and use it in the ssl-cert option of the marathon lb install. If there are issues with what has been entered, the log for MLB will show that there was an issue with the certificate. Assuming there are no issues, you can now setup CNAMES to point to the public ELB just like you did for the master address. Then, when setting up apps in Marathon, you would want to set the following labels:
```
"labels":{
  "HAPROXY_GROUP":"external",
  "HAPROXY_0_REDIRECT_TO_HTTPS":"true",
  "HAPROXY_0_VHOST": "yourapp.yourdomain.com"
}
```
