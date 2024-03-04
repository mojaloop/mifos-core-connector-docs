# Mifos Core Connector Sequence Diagrams

```mermaid
sequenceDiagram
    Fineract CBS-->>+Core Connector: POST /sendmoney
    Core Connector-->>+Token Adapter: POST /sendmoney
    Token Adapter-->>+SDK Scheme Adapter: POST /sendmoney
    Fineract CBS-->>+Core Connector: POST /sendmoney
    Core Connector-->>+Token Adapter: POST /sendmoney
    SDK Scheme Adapter-->>+Token Adapter: POST /sendmoney
```