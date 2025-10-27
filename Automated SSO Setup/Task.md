# Drupal and Keycloak on seperate VMs

## 1. Setup VM for both Drupal and Keycloak

Refer **'Seprated VM Setup/Setting Up Host-only Adapter'**, it covers VM setup followed by a proper Host-Only network connection that allows us to get a static ip and is useful in SSO setup.

## 2. Setup drupal and keycloak VM

Install keycloak and drupal on respective VMs, Refer **'Seprated VM Setup/Server A - Keycloak Setup'** and **'Seprated VM Setup/Server B - Openplc website setup'**

## 3. Keycloak script

Create dir and set permissions

```
mkdir ~/keycloak
cd keycloak
```

Create the following files with given contents

config.json

```json
[
  {
    "keycloak_realm": "",
    "keycloak_base": "",
    "keycloak_user": "",
    "keycloak_password": "",
    "drupal_base": "",
    "keycloak_client_id": ""
  }
]
```

client.json

```json
{
  "clientId": "",
  "enabled": true,
  "publicClient": false,
  "protocol": "openid-connect",
  "clientAuthenticatorType": "client-secret",
  "standardFlowEnabled": true,
  "redirectUris": [""]
}
```

Add the script

```sh
wget 
```

Fix permissions and run the script

```sh
sudo chown -R keycloak:$USER .
sudo chmod -R g+w .
chmod +x keycloak
./keycloak
```

This will generate secret.json file copy the contents of the file

## 4. Drupal script

Create a dir and copy required files

```sh
mkdir drupal
cd drupal
```

Add the file which you got from keycloak script (optional we have other way too).


```json
[ {
  "clientId" : <your_clientId>,
  "secret" : <your_clientSecret>
} ]
```

Add script file

```sh
wget 
```

Run the script file

```sh
chmod +x drupal
./drupal -ci clientId -s 'your_clientSecret' -v
# or using long options:
./drupal --client-id clientId --secret 'your_clientSecret' --verbose
```

## 5. Test SSO

Open your drupal site and try to login