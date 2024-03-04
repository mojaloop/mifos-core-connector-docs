# Mifos Core Connector Sequence Diagrams

```mermaid
sequenceDiagram
    Mifos Core Connector-->>+Core Connector: POST /sendmoney
    Core Connector-->>+Token Adapter: POST /sendmoney
    Token Adapter-->>+SDK Scheme Adapter: POST /sendmoney
    Mifos Core Connector-->>+Core Connector: POST /sendmoney
    Core Connector-->>+Token Adapter: POST /sendmoney
    Token Adapter-->>+SDK Scheme Adapter: POST /sendmoney
```