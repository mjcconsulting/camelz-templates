# Utility-DUO Notes

Notes on how to install and configure DUO to work with DirectoryService and ActiveDirectory

## LDAP Account Setup

### DirectoryService
Create an LDAP account
- Username: LDAP
- In ou=Users,ou=camelz,dc=ad,dc=camelz,dc=io (for Identity VPC)

### ActiveDirectory
Create an LDAP account
- Username: LDAP
- In ou=Users,dc=ad,dc=camelz,dc=camelz,dc=io (for Identity VPC)

## LDAP Testing

### Install OpenLDAP Clients

```bash
sudo yum install -y openldap-clients
```

## Test Connection

```bash
password=MyLDAPPassword
echo -n $password > ~/.ldappw
chmod 400 ~/.ldappw
user=LDAP  # Use this as the LDAP auth user, should match account created above
host=172.21.63.126 # Obtain this from the DirectoryService GUI
base=ou=Users,ou=camelz,dc=ad,dc=camelz,dc=io
ldapsearch -x -b $base -h $host -D $user -y ~/.ldappw '(sAMAccountName=MCrawford)'
```

## Configure DAG


## Backup & Restore
It looks like we can go around the GUI to configure this by using the backup and restore mechanism.

We may need to install the software, then dump it's default config with either no configuration or perhaps with just the password configured via a curl POST, to get our starting point config files.

Then, we can modify those then restore the modified backup and restart docker to pick up the changes.

### Backup

Copy DUO's data directory to a backup location.

```bash
mkdir -p /var/lib/duo/data
docker cp access-gateway:/data /var/lib/duo/data
```

It's possible to extract many config files from within this directory structure.

### Restore

Copy DUO's backup location back into the docker image.

```bash
docker cp /var/lib/duo/data/. access-gateway:data
```

Then have docker recognize the new data.

```bash
docker exec -ti --user=0 access-gateway chown -R www-data:www-data /data
```

## Download DAG Information
To download the certificate:
- https://duo.camelz.io:8443/dag/saml2/idp/certificate.php

To download the XML metadata:
- https://duo.camelz.io:8443/dag/saml2/idp/metadata.php?output=attach

The metadata file is identical between WS1 and WS2 DUO environments, except for the domain name and the signing crt certificate text, which can be replaced by two tokens:
- ${DUO_HTTPS_URL} - The HTTPS URL, for example duo.camelz.com.
    - This must match the SSL key and cert we upload
    - These files can be found




https://duo.camelz.com/dag/saml2/idp/SSOService.php?spentityid=DIDGXI96I58VSEBXQ62J
