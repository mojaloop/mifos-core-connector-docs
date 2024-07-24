Payee Get Parties sequence diagram

```mermaid
sequenceDiagram
    SDK Scheme Adapter->>Core Connector: GET /parties/IBAN/{ID}
    Core Connector ->> Core Connector: Check request parameters and request body
    Alt If required fields missing
    Core Connector-->>SDK Scheme Adapter: Response 400 Bad Request
    End
    Core Connector->>Core Connector: Check ID Type, Validate & Extract Account No from ID
    Alt If Id Type invalid
    Core Connector-->>SDK Scheme Adapter: Response 400 Bad Request
    End
    Core Connector->>Apache Fineract: GET /fineract-provider/api/v1/search?query={AccountNo}&resource=savingsaccount
    Apache Fineract-->>Core Connector: Response [{}...]
    Core Connector ->> Core Connector: Check response
    Alt If Response is not Success
    Core Connector -->> SDK Scheme Adapter: Response 500: ML Code: 5000
    End
    Core Connector-->>Core Connector: Extract Account Id
    Core Connector->>Apache Fineract: GET /fineract-provider/api/v1/savingsaccounts/{accountId}/
    Apache Fineract-->>Core Connector: Response {...}
    Alt If Response is not Success
    Core Connector -->> SDK Scheme Adapter: Response 500: ML Code: 5000
    End
    Core Connector->>Core Connector: Verify Account is active and can receive deposits
    Alt If Account Active
    Core Connector-->>Core Connector: Extract Client Id
    Core Connector->>Apache Fineract: GET /fineract-provider/api/v1/clients/{clientId}/
    Apache Fineract-->>Core Connector: Response 200 OK {...}
    Core Connector-->>SDK Scheme Adapter: Response 200 OK {...}
    Else If Account Not Found
    Core Connector-->>SDK Scheme Adapter: Response 404 Not Found : ML Code : 5107
    Else If Account Not Active
    Core Connector-->>SDK Scheme Adapter: Response 404 Not Found : ML Code : 5107
    End
```
