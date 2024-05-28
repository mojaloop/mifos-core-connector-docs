# Mifos Core Connector for Mojaloop Design Findings 

> The term fineract and Mifos are used interchangeably in this documentation to mean the same thing.

# Overview
This documentation is an account of the experience I had while developing the mifos core connector. The idea is to make sure the steps of how I got everything working and testing is documented and can easily be referenced for someone else to use. 


# Deploying and setting up fineract for development
To setup fineract for development, you will need to do following.

# Prerequisites

The following are required for you to successfully go through this documenation.
- Working knowledge of [docker](https://docs.docker.com/)
- Docker installed on your machine


## Deploy mariadb
Fineract uses mysql as it's default database management system. You will need to deploy an instance of mariadb on your localhost to be able to run fineract successfully.

Run this command to run a local instance of mariadb. Note the parameter `-e MARIADB_ROOT_PASSWORD=mysql` which sets the password for the mariadb instance.

```bash
docker run --name mariadb-11.2 -p 3306:3306 -e MARIADB_ROOT_PASSWORD=mysql -d mariadb:11.2
```

After running this command, you will have a local running instance of mariadb.

## Setup database
Fineact requires some database resources to be in place for it to start successfully.

Clone the fineract repository on your local machine and cd into the repo folder.

```bash
git clone https://github.com/apache/fineract.git; cd fineract
```

After cloning the repository, please execute the following commands at the root of fineract repo folder to create the database resources.

```bash
./gradlew createDB -PdbName=fineract_tenants
```

```bash
./gradlew createDB -PdbName=fineract_default
```

After running these commands, if they executed successfully, the database resources will be created and now we can proceed to run fineract.

## Deploy fineract
After you have deployed an instance of mariadb and created the database resources, you can then deploy the latest stable release of Apache Fineract from Docker Hub.

Run this command to deploy fineract locally.

```bash
docker run --name fineract -p 8443:8443 -p 8080:8080  -e FINERACT_DEFAULT_TENANTDB_PWD=mysql -e FINERACT_DEFAULT_TENANTDB_HOSTNAME=localhost -e FINERACT_SERVER_SSL_ENABLED=false --network="host" apache/fineract
```

# Existing public fineract instances 

There are existing public instances of Apache fineract hosted and maintained by [Mifos Initiative](https://www.mifos.org)

Here is one I used for the development and validation of my api routes 
- [https://demo.mifos.io/](https://demo.mifos.io/)

You can make requests to this base url while referring to the api docs at [Docs](https://demo.mifos.io/api-docs/apiLive.htm)

# Fineract Core Connector Sequence Diagram

## Payee account in Fineract
> Make sure to view this in github to see the rendered sequence diagram. [Issue on Github](https://github.com/mojaloop/project/issues/3711)

```mermaid
sequenceDiagram
    SDK Scheme Adapter->>Core Connector: GET /parties/IBAN/{ID}
    Core Connector->>Core Connector: Validate &Extract Account No from IBAN
    Core Connector->>Apache Fineract: GET /fineract-provider/api/v1/search?query={AccountNo}&resource=savingsaccount
    Apache Fineract-->>Core Connector: Response 200 OK [{}...]
    Core Connector-->>Core Connector: Extract Account Id
    Core Connector->>Apache Fineract: GET /fineract-provider/api/v1/savingsaccounts/{accountId}/
    Apache Fineract-->>Core Connector: Response 200 OK {...}
    Core Connector->>Core Connector: Verify Account is active and can receive deposits
    Alt If Account Active
    Core Connector-->>Core Connector: Extract Client Id
    Core Connector->>Apache Fineract: GET /fineract-provider/api/v1/clients/{clientId}/
    Apache Fineract-->>Core Connector: Response 200 OK {...}
    Core Connector-->>SDK Scheme Adapter: Response 200 OK {...}
    Else If Account Not Active
    Core Connector-->>SDK Scheme Adapter: Response 404 Not Found
    End
    SDK Scheme Adapter->>Core Connector: POST /quoterequests
    Core Connector-->>Core Connector: No Fees for Deposits
    Core Connector->>Core Connector: Validate &Extract Account No from IBAN
    Core Connector->>Apache Fineract: GET /fineract-provider/api/v1/search?query={AccountNo}&resource=savingsaccount
    Apache Fineract-->>Core Connector: Response 200 OK [{}...]
    Core Connector-->>Core Connector: Extract Account Id
    Core Connector->>Apache Fineract: GET /fineract-provider/api/v1/savingsaccounts/{accountId}/
    Apache Fineract-->>Core Connector: Response 200 OK {...}
    Core Connector->>Core Connector: Verify Account is active and can receive deposits
    Core Connector-->>Core Connector: Extract Client Id
    Core Connector->>Apache Fineract: GET /fineract-provider/api/v1/clients/{clientId}/
    Apache Fineract-->>Core Connector: Response 200 OK {...}
    Core Connector-->>Core Connector: Tiered KYC Checks need to be performed here.
    Core Connector-->>SDK Scheme Adapter: Response 200 OK {...}
    SDK Scheme Adapter->>Core Connector: POST /transfers
    Core Connector->>Core Connector: Extract Account No from IBAN
    Core Connector->>Apache Fineract: GET /fineract-provider/api/v1/search?query={AccountNo}&resource=savingsaccount
    Apache Fineract-->>Core Connector: Response 200 OK [{}...]
    Core Connector-->>Core Connector: Extract Account Id
    Core Connector->>Apache Fineract: POST /fineract-provider/api/v1/savingsaccounts/{accountId}/transactions?command=deposit
    Alt If No Error
    Apache Fineract-->>Core Connector: Response 200 OK {...}
    Core Connector-->>SDK Scheme Adapter: Response 200 OK {...}
    Else If Error
    Core Connector-->>SDK Scheme Adapter: Response 500 Server Error
    End
```

# Integration for Payer Flow.

```mermaid
sequenceDiagram
    Dfsp Operations App->>Core Connector: POST /transfer {..} 
    Core Connector->>Fineract: GET /fineract-provider/api/v1/savingsaccounts/{accountId}/
    Fineract-->>Core Connector: Response 200 OK {...}
    Core Connector->> Core Connector: Check Account is active
    Alt If Account is inactive
    Core Connector -->> Dfsp Operations App: Error 500 {...}
    End
    Core Connector->> SDK Scheme Adapter: POST /transfers {AUTO_ACCEPT_QUOTES=false, AUTO_ACCEPT_PARTY=true}
    Alt If Account not found
    SDK Scheme Adapter-->> Core Connector: 400/500/504 {...} 
    Core Connector-->> Dfsp Operations App: 400/500/504 {...}
    End
    SDK Scheme Adapter-->> Core Connector: 200 OK {...}
    Core Connector->> Core Connector: Check Quote Amounts and Account Balance
    Alt If Quote ok
    Core Connector-->> Dfsp Operations App: 200 OK {.Quote.}
    Else If Qoute not ok
    Core Connector-->> Dfsp Operations App: 500 Error {...}
    End
    Dfsp Operations App->> Core Connector: PUT /transfers/{transferID} {acceptQuote=true}
    Core Connector->> Fineract: Reserve funds POST /api/v1/accounttransfers
    Fineract-->> Core Connector: 200 Ok
    Core Connector->> SDK Scheme Adapter: PUT /transfers/{transfersId} {acceptQuote=true}
    Alt If Transfer Successful
    SDK Scheme Adapter-->> Core Connector: 200 OK {...}
    Core Connector->> Fineract:POST /fineract-provider/api/v1/savingsaccounts/{accountId}/transactions?command=withdraw
    Core Connector -->> Dfsp Operations App: 200 OK {...}
    Else if Transfer not Successful
    SDK Scheme Adapter -->> Core Connector: 500/504 Error {...}
    Core Connector ->> Fineract: Refund payer POST /api/v1/accounttransfers
    Core Connector-->> Dfsp Operations App: 500/504 Error {...}
    End
```


# G2P/MOSIP Demo 

The goal for this demo is to illustrate how Digital Public Goods can be used by governments to extend financial aid to 
their citizens in times of distress.

# Scenario 
The government uses a beneficiary portal to register new beneficiaries who are picked from an Identity platform powered by MOSIP. After verifying the identity of a person, a payment token alias is created which is stored in Mojaloop and a DFSP deployed payment token 
adapter.

When payments are being sent from government to a beneficiary. An account lookup is performed against the payment token alias that was created earlier. When the Mojaloop Hub has resolved the DFSP to which the account belongs, it will perform a get parties request to the beneficiaries DFSP.

In this scenario, Apache Fineract is being used as the core banking system for managing people's accounts in the beneficiary's DFSP. Once the DFPS receives a get parties request, it first of all performs a payment token mapping to get the real ID type of the beneficiary and then queries the Core Banking System (Apache Fineract) through a core connector (Mifos Core Connector) to get the 
details about the beneficiary.

Once a party is found or not found, an appropriate response is returned to Mojaloop indicating whether the party was found or not. If the party was found, a request for a quote is prepared by mojaloop and subsequently a transfer can also be initiated by the Government's DFSP.

All incoming deposit requests will go through the payment token adapter such that the real Party ID Type may be substituted in place of the ALIAS.


Here is a sequence diagram to illustrate this interaction at a high level.

# Payment Token Registration Process

```mermaid
sequenceDiagram
    autonumber
    Beneficiary Portal ->> E Signet MOSIP: Get Beneficiary Details from MOSIP 
    Beneficiary Portal ->> Beneficiary Portal: create Payment Token to be mapped to an specific Id Value of IdType X
    Beneficiary Portal ->> Mojaloop Hub: Register the account in a dfsp with type ALIAS
    Beneficiary Portal ->> Payment Token Adapter: Register the payment token in PTA with Id Value and Type
```

# Synchronous Payment Process

The different steps of a Mojaloop payment are divided into 3 sequence diagrams for ease of understanding.

# Get Parties

```mermaid
sequenceDiagram
    autonumber
    GOV Dfsp ->> Mojaloop: Account Lookup with to.IdType ALIAS
    Mojaloop ->> Beneficiary DFSP PTA: GET /parties/ALIAS/ID
    Beneficiary DFSP PTA ->> Beneficiary DFSP PTA: Substitute ALIAS for actual ID.
    Beneficiary DFSP PTA ->> Mifos CC: GET /parties/{IdType}/ID
    Mifos CC ->> Apache Fineract: Search for account.
    Apache Fineract-->>Mifos CC: Return Party Details.
    Mifos CC -->> Beneficiary DFSP PTA: Return Party Details.
    Beneficiary DFSP PTA ->> Beneficiary DFSP PTA: Substitute actual ID for ALIAS
    Beneficiary DFSP PTA -->> Mojaloop: Return Party Details
    Mojaloop --> GOV Dfsp: Return Party Details.
```

# Quotes

```mermaid
sequenceDiagram
    autonumber
    GOV Dfsp ->> Mojaloop: Quote request with to.IdType ALIAS
    Mojaloop ->> Beneficiary DFSP PTA: POST /quoterequests
    Beneficiary DFSP PTA ->> Beneficiary DFSP PTA: Substitute ALIAS for actual ID.
    Beneficiary DFSP PTA ->> Mifos CC: POST /quoterequests
    Mifos CC ->> Apache Fineract: Calculate Quote for Account based on Amount
    Apache Fineract-->>Mifos CC: Return Quote
    Mifos CC -->> Beneficiary DFSP PTA: Return Quote
    Beneficiary DFSP PTA ->> Beneficiary DFSP PTA: Substitute actual ID for ALIAS
    Beneficiary DFSP PTA -->> Mojaloop: Return Quote
    Mojaloop --> GOV Dfsp: Return Quote
```

# Transfers

```mermaid
sequenceDiagram
    autonumber
    GOV Dfsp ->> Mojaloop: Transfer request with to.IdType ALIAS
    Mojaloop ->> Beneficiary DFSP PTA: POST /transfers
    Beneficiary DFSP PTA ->> Beneficiary DFSP PTA: Substitute ALIAS for actual ID.
    Beneficiary DFSP PTA ->> Mifos CC: POST /transfers
    Mifos CC ->> Apache Fineract: Credit beneficiary's account
    Apache Fineract-->>Mifos CC: Return Successful deposit
    Mifos CC -->> Beneficiary DFSP PTA: Return Committed Transfer
    Beneficiary DFSP PTA ->> Beneficiary DFSP PTA: Substitute actual ID for ALIAS
    Beneficiary DFSP PTA -->> Mojaloop: Return Committed Transfer
    Mojaloop --> GOV Dfsp: Return Committed Transfer
```