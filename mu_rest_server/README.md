# Multi User REST Server Hyperledger Composer Network
 
## Setting up Single Use REST Server and Playground to run on IBM Cloud 

Follow the directions in [Deploying a Business Network](https://console.bluemix.net/docs/services/blockchain/develop_starter.html#deploying-a-business-network) to install the .bna file on the IBM Blockchain Starter Plan Service, substituting `<network name>` for vehicle-manufacturing-network. Keep track of the `admin@<network name>.card`. It will be used later.

### Setup the Cloudant Database Wallet

- Follow the [instructions](https://console.bluemix.net/docs/services/Cloudant/tutorials/create_service.html#creating-a-cloudant-nosql-db-instance-on-ibm-cloud) to create a Cloudant Service
- Once created, create [service credentials](https://console.bluemix.net/docs/services/Cloudant/tutorials/create_service.html#the-service-credentials) 
- Once the credential has been created, save a copy as a json file, call it `cloudant.json`
- Edit the `cloudant.json` file to look like the following:

```
{
    "composer": {
        "wallet": {
            "type": "composer-wallet-cloudant",
            "options": {
	        "host": "<hostname>",
                  "password": "*************************************************",
                  "port": 443,
                  "url": "<cloudant URL>",
                  "username": "<username>",
                  "database": "wallet"
            }
        }
    }
}
```
Make sure the module that provides the support to connect from Composer to Cloudant is installed.
```
npm install -g composer-wallet-cloudant@0.0.13
```
Create the Cloudant database using the value in the JSON for the url field:
```
curl -X PUT <CLOUDANT_URL>/wallet
```
Set the NODE_CONFIG environment variable on your machine using the contents of your `cloudant.json` file with new lines removed:
```
export NODE_CONFIG=$(awk -v RS= '{$1=$1}1' < cloudant.json)
```
Import the admin card to the Cloudant service:
```
composer card import -f admin@<network name>.card
```
### Deploy the REST Server
Log in to Cloud Foundry and select the space you deployed your Blockchain service to:
```
bx cf login -sso 
bx target -o "<org>" -s <space> -g <group>
```
Push the REST server using the docker image:
```
bx cf push our-rest-server --docker-image hyperledger/composer-rest-server:0.20.2 -i 1 -m 256M --no-start --no-manifest --random-route
```
Set environment variable for the REST server:
```
bx cf set-env our-rest-server NODE_CONFIG "$NODE_CONFIG"
bx cf set-env our-rest-server COMPOSER_CARD admin@<network name>
bx cf set-env our-rest-server COMPOSER_WEBSOCKETS true
bx cf set-env our-rest-server COMPOSER_MULTIUSER false
```
Start the server
```
bx cf start our-rest-server
```
### Deploy the Playground 
```
bx cf push our-playground --docker-image hyperledger/composer-playground:0.20.2 -i 1 -m 256M --no-start --random-route --no-manifest
bx cf set-env our-playground NODE_CONFIG "$NODE_CONFIG"
bx cf start our-playground
```  
## Running Kubernetes REST Server in Multi User Mode

Reference links: 
https://hyperledger.github.io/composer/latest/integrating/enabling-rest-authentication, https://hyperledger.github.io/composer/latest/integrating/deploying-the-rest-server.html

### Authenticate

 - sign into github to authenticate
 - go to `<external kube cluster ip address>:3000/auth/github` to run as an authenticated user
 
### Import cards

 - execute POST/wallet/import using card (`admin@<network name>`) provided. Make sure this card has System Admin privileges.
 - execute POST/wallet/{name}/setDefault to set card as the default
 
### Create a new participant and create an identity and card for them

Using the REST Server UI:

 - execute POST/Developer to create a Developer participant
 - issue a new identity for the new Developer by running a curl command. This can't be done using the REST UI because the response needs to be captured to create the new card to import:
```
curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/octet-stream' --header 'X-Access-Token: <Access Token>' -d '{ "participant": "org.ibm.<namespace>.#Developer", "userID": "NewDevID", "options": {"card":"admin@<network name>"} }' 'http://<external kube cluster ip address>:3000/api/system/identities/issue' > NewDev.card
```
 - execute POST/wallet/import to import the NewDev.card file
 
Using the Composer CLI:
 ```
composer participant add -c admin@<network name> -d '{"$class":"org.ibm.<namespace>.Developer","id":"NewDev", "name":"Dev Name"}'
composer identity issue -c admin@<network name> -f NewDev.card -u NewDev -a "resource:org.ibm.<namespace>.Developer#NewDev"
composer card import -f NewDev.card
```
Execute POST/wallet/{name}/setDefault to change to new identity. ACL rules for *Developer* Participant associated with identity will be in place.

## Setup Demo

Invoke the **SetupDemo** transaction if available to prefill the business network
 
## Error checking
  - within Basic Angular App, "admin" id is assumed, so participant designated Access Control Rules are not applicable
  - if using the Angular App, text must be entered for all updatable fields in the transaction
 
