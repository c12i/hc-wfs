# Holochain Workflows

## Startup

```mermaid
flowchart TB
    A[Start Holochain Binary] --> B[Load Config Files]
    B --> C[Connect to Lair Keystore]

    C -->|Not Running| D[Start Lair Process]
    C -->|Running| E[Unlock Lair]

    D --> E
    E -->|Failed| F[Wait for Passphrase]
    F --> E

    E -->|Success| G[Initialize SQLite Databases]
    G --> H[Setup Networking Layer]
    H --> I[Initialize Conductor]
    I --> J[Start Admin Interface]
    J --> K[Load DNA Files]
    K --> L[Create Cell Instances]
    L -->|Get Agent Keys| M[Query Lair for Keys]
    M --> N{Check for Running Apps}
    N -->|Yes| O[Start App Interface]
    N -->|No| P[Wait for Admin Commands]
    O --> Q[Start WebSocket Interface]
    P --> Q
    Q --> R[Ready for Connections]

    style C fill:#f9f,stroke:#333,stroke-width:2px
    style D fill:#f9f,stroke:#333,stroke-width:2px
    style E fill:#f9f,stroke:#333,stroke-width:2px
    style F fill:#f9f,stroke:#333,stroke-width:2px
    style M fill:#f9f,stroke:#333,stroke-width:2px
    style G fill:#87CEEB,stroke:#333,stroke-width:2px
```

### Key Components

- **Lair Keystore Integration** (highlighted in pink):

  - Lair is Holochain's secure key management system
  - Handles agent key creation and storage
  - Requires passphrase authentication for access
  - Must be running before Holochain can access keys

- **Data Storage** (highlighted in blue):
  - SQLite databases store local state and DHT data
  - Initializes storage for source chains and DHT operations
  - Maintains persistent state across restarts

## Check for Running hApps

```mermaid
flowchart TB
    A[Check for Running Apps] --> B[Query SQLite State DB]
    B --> C{Apps Found?}

    C -->|Yes| D[For Each App]
    C -->|No| E[Wait for Admin Commands]

    D --> F{Check App Status}
    F -->|Running| G[Load App Configuration]
    F -->|Disabled/Paused| H[Skip App]

    G --> I[Verify DNA Integrity]
    I -->|Success| J[Start App Interface]
    I -->|Failure| K[Mark App as Error]

    J --> L[Configure Network Interfaces]
    L --> M[Initialize App Websocket]

    H --> N[Ready for Admin Commands]
    K --> N
    M --> O[Ready for Connections]

    style B fill:#87CEEB,stroke:#333,stroke-width:2px
```

### Key Components

- **State Management** (highlighted in blue):

  - SQLite queries determine which apps were previously running
  - Maintains app configurations and state between sessions

- **Recovery Process**:
  - Automatically restarts previously running applications
  - Verifies DNA integrity before startup
  - Establishes necessary network interfaces

## hApp Installation and Genesis

```mermaid
flowchart TB
    subgraph "App Installation"
        A[Install App Request] --> B[Process DNA Files]
        B --> C[Store WASM/DNA Records]
        C --> D[Create Cell ID]
        D --> E{Start Now?}
        E -->|No| F[Store Install Record]
        F --> G[Wait for Start Command]
    end

    subgraph "Cell Start Process"
        H[Start Cell Command] --> I{Genesis Exists?}
        I -->|No| J[Perform Genesis]
        I -->|Yes| K[Load Existing State]

        J --> L[Create Genesis Entries]
        L --> M[Run Init Callbacks]
        M --> N[Join DHT]

        K --> N
        N --> O[Cell Running]
    end

    E -->|Yes| H
    G --> H

    style C fill:#87CEEB,stroke:#333,stroke-width:2px
    style F fill:#87CEEB,stroke:#333,stroke-width:2px
    style J fill:#FFB6C1,stroke:#333,stroke-width:2px
    style L fill:#FFB6C1,stroke:#333,stroke-width:2px
    style M fill:#FFB6C1,stroke:#333,stroke-width:2px
```

### Key Components

- **Storage Operations** (highlighted in blue):

  - Stores DNA and WASM modules
  - Maintains installation records
  - Tracks cell status and configuration

- **Genesis Process** (highlighted in pink):
  - One-time initialization of new cells
  - Creates initial entries in the source chain
  - Runs initialization callbacks
  - Critical for establishing cell state

## DNA Installation

```mermaid
flowchart TB
    A[Read .dna file] --> B[Parse DNA Manifest]
    B --> C[Extract WASM]
    B --> D[Extract DNA Manifest]

    C --> E[Hash WASM]
    D --> F[Process DNA Properties]

    E --> G{WASM in Database?}
    G -->|No| H[Store WASM in SQLite]
    G -->|Yes| I[Use Existing WASM]

    F --> J[Create DNA Record]
    H --> J
    I --> J

    J --> K[Store DNA Record]

    style H fill:#87CEEB,stroke:#333,stroke-width:2px
    style K fill:#87CEEB,stroke:#333,stroke-width:2px
```

### Key Components

- **Storage Operations** (highlighted in blue):

  - Deduplicates WASM modules
  - Stores DNA records with properties
  - Maintains integrity through hashing

- **Key Elements**:
  - DNA manifest defines application properties
  - WASM modules contain application logic
  - Properties configure DNA behavior

## Zome Call

```mermaid
flowchart TB
    subgraph "Entry Point"
        A[App Interface Call] --> B[Call Validation]
        B --> C[Serialize Input]
    end

    subgraph "Conductor Layer"
        C --> D[Route to Cell]
        D --> E[Build Host Context]
        E --> F[Prepare Ribosome]
    end

    subgraph "WASM Execution"
        F --> G[Initialize WASM]
        G --> H[Call Zome Function]

        H --> I{Host Function Called?}
        I -->|Yes| J[Suspend WASM]
        J --> K[Execute Host Function]
        K --> L[Resume WASM]
        L --> I

        I -->|No| M[Complete Function]
    end

    subgraph "State Management"
        K --> N[Source Chain Ops]
        N --> O[DHT Ops Queue]
    end

    subgraph "Return Flow"
        M --> P[Serialize Output]
        P --> Q[Clean Resources]
        Q --> R[Return Result]
    end

    style B fill:#FFB6C1,stroke:#333,stroke-width:2px
    style E fill:#90EE90,stroke:#333,stroke-width:2px
    style F fill:#F0E68C,stroke:#333,stroke-width:2px
    style H fill:#F0E68C,stroke:#333,stroke-width:2px
    style K fill:#90EE90,stroke:#333,stroke-width:2px
    style N fill:#87CEEB,stroke:#333,stroke-width:2px
    style O fill:#87CEEB,stroke:#333,stroke-width:2px
```

### Key Components

- **Validation Layer** (highlighted in pink):

  - Ensures calls are properly formatted
  - Validates permissions and capabilities

- **Host Functions** (highlighted in green):

  - Provides interface between WASM and host
  - Manages core Holochain capabilities

- **WASM Execution** (highlighted in yellow):

  - Runs application logic in sandbox
  - Manages state transitions
  - Handles host function calls

- **State Management** (highlighted in blue):
  - Records operations to source chain
  - Queues DHT operations
  - Ensures data consistency

## Validation Operations

```mermaid
flowchart TB
    subgraph "Source Chain Operation"
        A[Zome Function] --> B[Create Entry/Link]
        B --> C[Validate Create]
        C --> D[Queue Source Chain Ops]
        D --> E[Queue DHT Ops]
    end

    subgraph "DHT Operations Queue"
        E --> F[Store Pending Ops]
        F --> G{Op Type?}
        G -->|StoreEntry| H[Store Entry Op]
        G -->|StoreRecord| I[Store Record Op]
        G -->|RegisterAgentActivity| J[Register Activity Op]
        G -->|CreateLink| K[Create Link Op]
    end

    subgraph "Network Layer"
        H & I & J & K --> L[Schedule Op Broadcast]
        L --> M[Send to Neighbors]
        M --> N[Receive Op]
    end

    subgraph "Validation Flow"
        N --> O[Queue Validation]
        O --> P[Run Validation Callback]
        P --> Q{Valid?}
        Q -->|Yes| R[Accept & Store]
        Q -->|No| S[Reject & Warn]

        R --> T[Add to Integrated Ops]
        S --> U[Add to Invalid Ops]
    end

    style C fill:#FFB6C1,stroke:#333,stroke-width:2px
    style P fill:#FFB6C1,stroke:#333,stroke-width:2px
    style D fill:#87CEEB,stroke:#333,stroke-width:2px
    style E fill:#87CEEB,stroke:#333,stroke-width:2px
    style F fill:#87CEEB,stroke:#333,stroke-width:2px
    style R fill:#87CEEB,stroke:#333,stroke-width:2px
```

### Key Components

- **Validation Checks** (highlighted in pink):

  - Runs validation callbacks
  - Ensures data integrity
  - Maintains network consensus

- **Storage Operations** (highlighted in blue):
  - Queues operations for processing
  - Stores validated data
  - Tracks invalid operations
  - Maintains operation status

## Best Practices

1. **Error Handling**:

   - Monitor validation results
   - Handle rejected operations gracefully
   - Implement retry mechanisms for network operations

2. **Performance**:

   - Batch DHT operations when possible
   - Implement efficient validation callbacks
   - Monitor queue sizes and processing times

3. **Security**:

   - Secure Lair keystore access
   - Implement thorough validation rules
   - Monitor for invalid operations

4. **Maintenance**:
   - Regular monitoring of SQLite databases
   - Clean up invalid operations
   - Monitor network connectivity
