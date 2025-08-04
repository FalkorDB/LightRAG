# FalkorDB Integration with LightRAG

LightRAG now supports [FalkorDB](https://falkordb.com/) as a graph storage backend. FalkorDB is a high-performance graph database built on Redis that supports Cypher queries.

## Prerequisites

1. **FalkorDB Server**: You need a running FalkorDB instance
   ```bash
   # Using Docker (recommended)
   docker run -p 6379:6379 falkordb/falkordb
   
   # Or install locally following FalkorDB documentation
   ```

2. **Python Dependencies**: The `falkordb` package will be automatically installed by LightRAG

## Configuration

### Option 1: Environment Variables

Create a `.env` file or set environment variables:

```bash
# Graph storage type
LIGHTRAG_GRAPH_STORAGE=FalkorDBStorage

# FalkorDB connection
FALKORDB_HOST=localhost
FALKORDB_PORT=6379
FALKORDB_GRAPH_NAME=lightrag_graph

# Optional authentication (if required)
# FALKORDB_PASSWORD=your_password
# FALKORDB_USERNAME=your_username

# Optional workspace for multi-tenancy
# FALKORDB_WORKSPACE=my_workspace
```

### Option 2: Configuration File

Create a `config.ini` file:

```ini
[falkordb]
host = localhost
port = 6379
graph_name = lightrag_graph
# password = your_password
# username = your_username
```

## Usage

### Basic Usage

```python
import os
from lightrag import LightRAG, QueryParam

# Configure for FalkorDB
os.environ["LIGHTRAG_GRAPH_STORAGE"] = "FalkorDBStorage"
os.environ["FALKORDB_HOST"] = "localhost"
os.environ["FALKORDB_PORT"] = "6379"
os.environ["FALKORDB_GRAPH_NAME"] = "my_knowledge_graph"

# Initialize LightRAG
rag = LightRAG(
    working_dir="./my_rag",
    llm_model_func=your_llm_function,
    embedding_func=your_embedding_function,
)

# Insert documents
await rag.ainsert("Your text content here...")

# Query the knowledge graph
response = await rag.aquery(
    "Your question here",
    param=QueryParam(mode="hybrid")
)
```

### Advanced Configuration

```python
from lightrag.kg.falkordb_impl import FalkorDBStorage
import numpy as np

async def embedding_func(texts):
    # Your embedding implementation
    return np.random.rand(len(texts), 384)

# Manual storage configuration
global_config = {
    "embedding_batch_num": 10,
    "max_graph_nodes": 1000,
    "working_dir": "./storage"
}

storage = FalkorDBStorage(
    namespace="my_project",
    global_config=global_config,
    embedding_func=embedding_func,
    workspace="production"  # Optional workspace
)

await storage.initialize()
# Use storage...
await storage.finalize()
```

## Features

### Supported Operations

- **Node Operations**: Create, read, update, delete nodes with properties
- **Edge Operations**: Create, read, update, delete relationships with properties
- **Batch Operations**: Efficient bulk operations for better performance
- **Graph Traversal**: Knowledge graph retrieval with configurable depth
- **Multi-tenancy**: Workspace isolation for different projects

### Performance Optimizations

- **Async/Sync Bridge**: Uses ThreadPoolExecutor to handle FalkorDB's synchronous API
- **Batch Queries**: Groups operations to reduce round trips
- **Connection Pooling**: Reuses connections efficiently
- **Retry Logic**: Handles transient errors with exponential backoff

### Graph Operations

```python
# Node operations
await storage.upsert_node("entity_id", {
    "entity_id": "entity_id",
    "description": "Description",
    "entity_type": "Person",
    "keywords": "keyword1,keyword2"
})

node = await storage.get_node("entity_id")
exists = await storage.has_node("entity_id")
degree = await storage.node_degree("entity_id")

# Edge operations  
await storage.upsert_edge("source_id", "target_id", {
    "relationship": "knows",
    "weight": 1.0,
    "description": "Connection description"
})

edge = await storage.get_edge("source_id", "target_id")
exists = await storage.has_edge("source_id", "target_id")

# Batch operations
nodes = await storage.get_nodes_batch(["id1", "id2", "id3"])
degrees = await storage.node_degrees_batch(["id1", "id2", "id3"])

# Knowledge graph retrieval
kg = await storage.get_knowledge_graph("starting_node", max_depth=2, max_nodes=100)
```

## Testing

The FalkorDB implementation includes comprehensive tests:

```bash
# Test structure and integration
python -c "
from lightrag.kg import verify_storage_implementation
verify_storage_implementation('GRAPH_STORAGE', 'FalkorDBStorage')
print('✓ FalkorDB integration verified')
"

# Run with existing test framework
LIGHTRAG_GRAPH_STORAGE=FalkorDBStorage python tests/test_graph_storage.py
```

## Comparison with Other Graph Storages

| Feature | FalkorDB | Neo4J | NetworkX |
|---------|----------|-------|----------|
| **Persistence** | Redis-based | Native DB | In-memory |
| **Query Language** | Cypher | Cypher | Python API |
| **Performance** | High | High | Medium |
| **Scalability** | High | Very High | Low |
| **Setup Complexity** | Low | Medium | None |
| **Memory Usage** | Low | Medium | High |

## Troubleshooting

### Connection Issues

```python
# Test connection
import falkordb

try:
    db = falkordb.FalkorDB(host='localhost', port=6379)
    graph = db.select_graph('test')
    result = graph.query("RETURN 1")
    print("✓ FalkorDB connection successful")
except Exception as e:
    print(f"❌ Connection failed: {e}")
```

### Common Issues

1. **Connection Refused**: Ensure FalkorDB is running on the specified host/port
2. **Authentication Error**: Check username/password if authentication is enabled
3. **Graph Not Found**: FalkorDB will create graphs automatically, but check permissions
4. **Memory Issues**: Monitor Redis memory usage, configure appropriately

### Debugging

Enable debug logging:

```python
import logging
logging.getLogger("falkordb").setLevel(logging.DEBUG)
logging.getLogger("lightrag").setLevel(logging.DEBUG)
```

## Migration

### From NetworkX

NetworkX data is in-memory and needs to be re-indexed:

```python
# No direct migration - re-process your documents
# NetworkX storage will be replaced automatically
```

### From Neo4J

Both use Cypher, but some syntax differences exist. Data export/import may be possible through Cypher dump/restore.

## Best Practices

1. **Graph Naming**: Use descriptive graph names for different projects
2. **Workspace Isolation**: Use workspaces for multi-tenant scenarios  
3. **Connection Management**: Always call `await storage.finalize()` when done
4. **Error Handling**: Implement retry logic for production applications
5. **Monitoring**: Monitor Redis memory usage and performance metrics
6. **Backup**: Regular backup of Redis data for production systems

## Example Projects

See the `/examples` directory for complete example applications:

- `examples/falkordb_example.py` - Basic usage example
- `examples/graph_visual_with_falkordb.py` - Graph visualization (coming soon)

## Contributing

When adding features to FalkorDB integration:

1. Follow the same patterns as other graph storage implementations
2. Add corresponding tests in the test suite
3. Update this documentation
4. Ensure compatibility with the existing BaseGraphStorage interface