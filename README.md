# Deploy two OpenStack regions and Keycloak container

Note that this setup uses a single Keystone and single OpenStack Dashboard
shared across two OpenStack regions.

```bash
# Bootstrap Juju Controller
juju bootstrap localhost lxd

# Deploy keycloak container
juju add-model keycloak
juju deploy ubuntu keycloak

# Deploy bundle
juju add-model openstack
juju deploy ./bundle.yaml

# Enable TLS for Keycloak, NOTE: replace <keycloak-IP>
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem \
  -sha256 -days 3650 -nodes \
  -subj "/C=UK/L=London/O=Canonical/CN=<keycloak-IP>"
juju scp key.pem cert.pem keycloak/0:

# Install Keycloak
juju ssh --model keycloak keycloak/0

sudo apt update && sudo apt upgrade
sudo apt install --yes openjdk-19-jre unzip

wget -O keycloak.zip https://github.com/keycloak/keycloak/releases/download/22.0.4/keycloak-22.0.4.zip

unzip keycloak.zip
cd keycloak-22.0.4

export KEYCLOAK_ADMIN=admin
export KEYCLOAK_ADMIN_PASSWORD=admin
bin/kc.sh start-dev \
  --https-certificate-file=/home/ubuntu/cert.pem \
  --https-certificate-key-file=/home/ubuntu/key.pem
```

# Set up Keycloak

Go to `https://<keycloak-IP>:8443/admin` and log in with admin:admin credentials.

## Create a realm

1. Open the Keycloak Admin Console.
2. Click the word master in the top-left corner, then click Create Realm.
3. Enter `openstack` in the Realm name field.
4. Click Create.

## Create a user

1. Open the Keycloak Admin Console.
2. Click the word master in the top-left corner, then click `openstack`.
3. Click Users in the left-hand menu.
4. Click Add user.
5. Fill in the form with username `keycloak`
6. Click Create.
7. Click Credentials at the top of the page.
8. Fill in the Set password form with a password `keycloak`.
9. Toggle Temporary to Off so that the user does not need update this password at the first login.

## Log in to the Account Console

1. Open the Keycloak Account Console, `https://<keycloak-IP>:8443/realms/openstack/account`
2. Log in with `keycloak:keycloak` credentials.

## Set up OpenStack integration in Keycloak

1. Open the Keycloak Admin Console.
2. Click the word master in the top-left corner, then click `openstack`.
3. Click Clients.
4. Click Create client
5. Fill in the form with the following values:
   - Client type: OpenID Connect
   - Client ID: `openstack`
6. Click Next
7. Confirm that Standard flow is enabled.
8. Click Next.
9. Add Client. Make these changes under Login settings.
   - Set Valid redirect URIs

   In order to read the valid redirect URIs, check ou the values of
   `OIDCRedirectURI` in /etc/apache2/openidc/openidc-location.openid.conf
   file on keystone/0 unit, e.g.: 

    OIDCRedirectURI "http://10.139.101.253:5000/v3/auth/OS-FEDERATION/identity_providers/openid/protocols/openid/websso"
    OIDCRedirectURI "http://10.139.101.253:5000/v3/auth/OS-FEDERATION/websso/openid"

   - Set Web origins to `http://<horizon-IP>/horizon`

   - In "Capability config", enable "Client authentication". This will enable
   "Credentials" tab, when you will learn "Client secret" needed to set up 
   `${CLIENT_SECRET}` in keystone-openidc configuration.

   - In "Capability config", enable Implicit flow, otherwise this error pops up:
    023-10-09 15:21:41,322 ERROR [org.keycloak.services] (executor-thread-3) KC-SERVICES0095: Client is not allowed to initiate browser login with given response_type. Implicit flow is disabled for the client.
    2023-10-09 15:21:41,322 WARN  [org.keycloak.events] (executor-thread-3) type=LOGIN_ERROR, realmId=8e4a26f5-1396-42bb-96fd-2b48c52da12e, clientId=openstack, userId=null, ipAddress=10.139.101.1, error=not_allowed, response_type=id_token, redirect_uri=http://10.139.101.253:5000/v3/auth/OS-FEDERATION/websso/openid, response_mode=fragment
10. Click Save.


# Configure Keystone - Keycloak integration

```bash
# Trust self-signed certificate
juju scp cert.pem keystone/0:
juju run -u keystone/0 \
  -- cp /home/ubuntu/cert.pem /usr/local/share/ca-certificates/keycloak.crt
juju run -u keystone/0 -- update-ca-certificates

CLIENT_ID=openstack 
CLIENT_SECRET=<secret-generated-in-keycloak-credentials-tab>
METADATA_URL="https://<keycloak-IP>:8443/realms/openstack/.well-known/openid-configuration"

juju config keystone-openidc \
    oidc-client-id="${CLIENT_ID}" \
    oidc-client-secret="${CLIENT_SECRET}" \
    oidc-provider-metadata-url="${METADATA_URL}"
```

# Set up OpenStack resources

```bash 
openstack identity provider create \
  --remote-id https://<keycloak-IP>:8443/realms/openstack openid

# Rename auto-generated domain name
openstack domain set --name federated_domain <auto-generated-domain-name>

openstack project create --domain federated_domain federated_project

cat > rules.json <<EOF
[
    {
        "local": [
            {
                "user": {
                    "name": "{0}",
                    "email": "{1}"
                },
                "group": {
                    "domain": {
                        "name": "federated_domain"
                    },
                    "name": "federated_users"
                },
                "projects": [
                {
                    "name": "federated_project",
                    "roles": [
                                 {
                                     "name": "member"
                                 }
                             ]
                }
                ]

            }
        ],
        "remote": [
            {
                "type": "OIDC-preferred_username"
            },
            {
                "type": "OIDC-email"
            }
        ]
    }
]
EOF

openstack mapping create --rules rules.json openid_mapping
openstack federation protocol create openid \
  --mapping openid_mapping --identity-provider openid
```

Append the following lines to the /etc/keystone/keystone.conf 
on the keystone/0 unit (replace `<horizon-IP>` with actual IP). Note that juju
will overwrite .conf file, so you need to stop juju agent systemd service.

```
[federation]
trusted_dashboard = http://<horizon-IP>/horizon/auth/websso/
```

Once this is all set up, youcan now log into OpenStack Dashboard with OIDC using
`keycloak:keycloak` credentials.


# Other example of rules.json

```bash
cat > rules.json <<EOF
[
    {
        "local": [
            {
                "user": {
                    "name": "{0} {1}",
                    "email": "{2}"
                },
                "projects": [
                {
                    "name": "federated_project",
                    "roles": [
                                 {
                                     "name": "member"
                                 }
                             ]
                }
                ]

            }
        ],
        "remote": [
            {
                "type": "OIDC-given_name"
            },
            {
                "type": "OIDC-family_name"
            },
            {
                "type": "OIDC-email"
            }
        ]
    }
]
EOF
```

Alternative profile claims: (https://developer.okta.com/blog/2017/07/25/oidc-primer-part-1#whats-a-scope)

    - name
    - family_name
    - given_name
    - middle_name
    - nickname
    - preferred_username
    - profile
    - picture
    - website
    - gender
    - birthdate
    - zoneinfo
    - locale
    - updated_at



