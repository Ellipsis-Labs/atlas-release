# Atlas Releases

### What is the Atlas RPC Node

Atlas RPC nodes are bridges between end users and our Atlas sequencer.

It provides a Solana-compatible JSON-RPC and websocket endpoint.

All transactions sent to the RPC node will be forwarded to our sequencer.

Once the sequencer processed those transactions, they would be sent back to the RPC node by a message bus (Redis for now), the RPC node would replay the transaction to make sure that the sequencer is not malicious.

### Machine spec:

At least 32 cores, 64G memory, and 100G disk
`linux/amd64` or `linux/arm64`

### Upstream connections / External Dependencies:

Redis(Message Bus): `redis://redis-testnet.atlas.xyz:6379/`

Postgres(Serve historical data): `postgresql://public_access:cfbea91fe55e79be93c69c7552d8c8114e1@postgres-testnet.atlas.xyz:5432/svm_node`

Sequencer: `http://testnet.atlas.xyz:3002`  (API Key required)

### Start the server from binary:

```bash
atlas-replay-node --mode=rpc \
    --log-level=info \
    --redis-url='redis://redis-testnet.atlas.xyz:6379/' \
    --server-url='https://testnet.atlas.xyz:3002/' \
    --postgres-url='postgresql://public_access:cfbea91fe55e79be93c69c7552d8c8114e12@postgres-testnet.atlas.xyz:5432/svm_node' \
    --api-key=<YOUR API KEY> \
    --num-async-threads=2
```

To save logs elsewhere, pass the argument `--log-dir`. The log files on disk will be rotated daily.

### Start from docker

```bash
docker run -d \
        --name atlas-rpc \
        -p 8899:8899 -p 8900:8900 \
        ghcr.io/ellipsis-labs/atlas-replay-node:v0.0.1 \
        --mode=rpc \
        --log-level=info \
        --redis-url='redis://redis-testnet.atlas.xyz:6379/' \
        --server-url='https://testnet.atlas.xyz:3002/' \
        --postgres-url='postgresql://public_access:cfbea91fe55e79be93c69c7552d8c8114e12@postgres-testnet.atlas.xyz:5432/svm_node' \
        --api-key=<YOUR API KEY> \
        --num-async-threads=2
```

### Kubernetes template

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: atlas-rpc
  labels:
    app: atlas-rpc
spec:
  replicas: 3
  selector:
    matchLabels:
      app: atlas-rpc
  template:
    metadata:
      labels:
        app: atlas-rpc
    spec:
      containers:
      - name: atlas-rpc
        image: ghcr.io/ellipsis-labs/atlas-replay-node:v0.0.1
        ports:
        - containerPort: 8899
        - containerPort: 8900
        env:
        args:
        - --mode=rpc
        - --log-level=info
        - --redis-url=redis://redis-testnet.atlas.xyz:6379/
        - --server-url=https://testnet.atlas.xyz:3002/
        - --api-key=<YOUR API KEY>
        - --postgres-url=postgresql://public_access:cfbea91fe55e79be93c69c7552d8c8114e12@postgres-testnet.atlas.xyz:5432/svm_node
        - --num-async-threads=2
        resources:
          requests:
            cpu: 24
            memory: 32Gi
          limits:
            cpu: 24
            memory: 32Gi
        livenessProbe:
          httpGet:
            path: /liveness
            port: 8899
          initialDelaySeconds: 5
          periodSeconds: 3
        readinessProbe:
          httpGet:
            path: /readiness
            port: 8899
          initialDelaySeconds: 5
          periodSeconds: 1
```

### Misc

- Historical state storage
    
    Right now, the historical state is saved in our Postgres database. 
    
    In the future, once we make our transaction ledger public, anyone can replay the ledger, rebuild the historical state, and store it in a private database.
    
- Re: external dependencies, some may change - We’re planning to migrate our Redis to Kafka soon, and the historical storage layer hasn’t been finalized yet.
