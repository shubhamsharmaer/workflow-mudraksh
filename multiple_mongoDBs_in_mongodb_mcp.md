## Step 1: Update the librechat.yaml file

```
YML
mcpServers:
  sales_db:
    name: Sales Database
    # Reference the Docker service in the librechat.yaml
    host: http://mongodb-mcp:8124
    toolNames:
      - connect
      - find
      - list-collections
  warehouse_db:
    name: Warehouse Database
    host: http://mongodb-mcp_warehouse:8124
    toolNames:
      - connect
      - find
      - list-collections
```

## Step 2: Update the docker-compose.override.yaml file

- Duplicate the mongodb-mcp service.
  Make a copy of your existing mongodb-mcp service and give it a unique name, such as mongodb-mcp_warehouse.
- Use different environment variables.
  Define a unique environment variable for each service, and use it to pass the connection string. This keeps sensitive credentials secure.
- Adjust the port.
  To avoid port conflicts, expose a different port for each service in the ports section. 

```
YML
version: '3.4'
services:
  # MCP Service for the sales database
  mongodb-mcp_sales:
    image: mongodb/mongodb-mcp-server:latest
    container_name: mongodb-mcp_sales
    restart: always
    command: --transport http --httpHost 0.0.0.0 --httpPort 8124
    environment:
      # Use the environment variable from the .env file
      - MDB_MCP_CONNECTION_STRING=${MDB_MCP_CONNECTION_STRING_SALES}
      - MDB_MCP_READ_ONLY=true
    # Expose the service on a unique port
    ports:
      - "8124:8124"

  # MCP Service for the warehouse database
  mongodb-mcp_warehouse:
    image: mongodb/mongodb-mcp-server:latest
    container_name: mongodb-mcp_warehouse
    restart: always
    command: --transport http --httpHost 0.0.0.0 --httpPort 8124
    environment:
      # Use a new environment variable for this database
      - MDB_MCP_CONNECTION_STRING=${MDB_MCP_CONNECTION_STRING_WAREHOUSE}
      - MDB_MCP_READ_ONLY=true
    # Expose the service on a unique port
    ports:
      - "8125:8124" # Use a different host port to prevent conflict

  api:
    depends_on:
      # The API service now depends on both MCP services
      - mongodb-mcp_sales
      - mongodb-mcp_warehouse
    extra_hosts:
      - "host.docker.internal:host-gateway"
    volumes:
      - type: bind
        source: ./librechat.yaml
        target: /app/librechat.yaml
```

## Step 3: Update the .env file
- Open your .env file.
  Add the new environment variables with the appropriate MongoDB connection strings

```
# Add new connection strings to your .env file
MDB_MCP_CONNECTION_STRING_SALES=mongodb://user:pass@host1:27017/sales_db
MDB_MCP_CONNECTION_STRING_WAREHOUSE=mongodb://user:pass@host2:27017/warehouse_db
```
