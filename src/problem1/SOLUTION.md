flowchart TB
    %% --- Client & Edge Tier ---
    subgraph Edge_Tier["Edge & Security Layer"]
        User["User / Trader (Web & Mobile)"]
        CF["AWS CloudFront (CDN) & AWS WAF"]
        R53["Route 53 (DNS / Latency Routing)"]
    end

    %% --- AWS Cloud Boundary ---
    subgraph AWS_Cloud["AWS Cloud Region (Multi-AZ)"]
        
        subgraph Public_Subnets["Public Subnets"]
            NLB["Network Load Balancer (NLB) <br/> <i>Handles TCP / WebSockets & REST</i>"]
        end

        subgraph Private_Compute["Private Subnets: Amazon EKS Cluster"]
            direction TB
            subgraph EKS_Ingress["API & Gateway Tier"]
                APIGW["API Ingress Service <br/> <i>Auth, Validation, Rate Limiting</i>"]
                WSS["WebSocket Gateway Pods <br/> <i>Real-time Price & Depth Streaming</i>"]
            end

            subgraph EKS_Core["Core Trading Tier"]
                ME["Matching Engine Pods <br/> <i>In-Memory Order Matching (Per-Pair)</i>"]
                Worker["Accounting & Settlement Worker <br/> <i>Balance Deduction & Trade Clearing</i>"]
            end
        end

        subgraph Private_Data["Isolated Data & Messaging Subnets (Multi-AZ)"]
            direction TB
            Stream["Amazon MSK / Redis Streams <br/> <i>FIFO Order Queue (Partitioned by Pair)</i>"]
            RedisCache["Amazon ElastiCache for Redis <br/> <i>Orderbook Depth Snapshot & Tickers</i>"]
            AuroraDB[("Amazon Aurora PostgreSQL <br/> <i>User Ledger, Balances, Trade History</i>")]
            S3Cold[("Amazon S3 <br/> <i>Audit Logs & Historical Archives</i>")]
        end

    end

    %% --- Traffic Flows ---
    User -->|DNS Lookup| R53
    User -->|Static Assets / Chart Data| CF
    User -->|Order Placement & WS Stream| NLB

    NLB -->|REST API / Orders| APIGW
    NLB -->|WebSocket Connections| WSS

    APIGW -->|1. Async Order Event| Stream
    Stream -->|2. Consume strictly-ordered orders| ME

    ME <-->|3. Update Orderbook Depth / Ticker| RedisCache
    RedisCache -.->|4. Pub/Sub Price Ticks| WSS

    ME -->|5. Trade Executed Event| Stream
    Stream -->|6. Consume Trade Match| Worker

    Worker -->|7. ACID Double-Entry Ledger| AuroraDB
    Worker -.->|Async Archive| S3Cold

    %% Styling
    classDef edge fill:#e1f5fe,stroke:#0288d1,stroke-width:2px;
    classDef compute fill:#ede7f6,stroke:#512da8,stroke-width:2px;
    classDef data fill:#e8f5e9
