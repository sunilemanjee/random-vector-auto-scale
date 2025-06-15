# Random Vector Track

This track is designed for benchmarking filtered search on random vectors.

By default, the `flat` vector_index_type is utilized to evaluate the performance
of brute force search over vectors filtered by a partition ID.

## Indexing

To begin indexing, the track initiates `index_clients` clients, each executing `index_iterations` bulk operations of size `index_bulk_size`. 
Consequently, the total number of documents indexed by the track is calculated as follows: `index_clients` * `index_iterations` * `index_bulk_size`.

Each document in the bulk is assigned a random vector of dimensions `dims` and a random partition ID.
The resulting index is sorted on the partition id. This helps make sure vectors are close together when we do filtered searches.

## Search Operations

Search operations involve filtering by a random partition ID and scoring against a random query vector. 
These operations are executed against the index using various DSL flavors, including script score and knn section.

### Parameters

This track accepts the following parameters with Rally 0.8.0+ using `--track-params`:

 - number_of_shards (default: 1)
 - number_of_replicas (default: 0)
 - vector_index_type (default: flat)
 - index_clients (default: 1)
 - index_iterations (default: 1000)
 - index_bulk_size (default: 1000)
 - search_iterations (default: 1000)
 - search_clients (default: 8)
 - dims (default: 128)
 - partitions (default: 1000)

### Burst/Auto-Scaled Workload Simulation

This track supports simulating bursty or auto-scaled workloads by specifying arrays for key parameters. Rally will iterate through these arrays to simulate changes in load.

#### Parameters for Burst Workloads

| Parameter                      | Description                                                                                  |
|--------------------------------|----------------------------------------------------------------------------------------------|
| `vector_index_type`             | Type of vector index (e.g., "flat")                                                          |
| `dims`                          | Number of dimensions for the vector                                                          |
| `partitions`                    | Number of partitions (controls data distribution and parallelism)                            |
| `as_ingest_clients`             | List of client counts for each phase of the workload                                         |
| `as_bulk_size`                  | List of bulk sizes (docs per request) for each phase                                         |
| `as_ingest_target_throughputs`  | List of target bulk requests/sec for each phase                                              |
| `as_parallel_index_iterations`  | List of iteration counts (number of bulks to send) for each phase                           |

#### Key Formulas

- **Total docs/sec** = `throughput` × `bulk_size`
- **Bulk requests per client** = `throughput` ÷ `clients`
- **Docs per client per sec** = (`throughput` ÷ `clients`) × `bulk_size`
- **Total docs** = `iterations` × `bulk_size`
- **Runtime (sec)** = `Total docs` ÷ `Total docs/sec`

#### Example Parameter File

```
{
    "vector_index_type": "flat",
    "dims": 1596,
    "partitions": 1000,
    "as_ingest_clients": [10, 20, 10],
    "as_bulk_size": [18, 18, 18],
    "as_ingest_target_throughputs": [8, 24, 8],
    "as_parallel_index_iterations": [5000, 50000, 5000]
}
```

This simulates:
- 1st phase: ~7,000 docs/sec (10 clients, 8 bulks/sec, 18 docs/bulk)
- 2nd phase: ~20,000 docs/sec (20 clients, 24 bulks/sec, 18 docs/bulk)
- 3rd phase: ~7,000 docs/sec (10 clients, 8 bulks/sec, 18 docs/bulk)

#### Partitions

- More partitions = more parallelism, but also more overhead.
- Tune this based on your cluster size and desired throughput.

The key is balancing all parameters so your system can actually achieve the target throughput without being bottlenecked by client capacity, network, or server resources.

#### Example: Writing 100,000 Documents

If you want to index exactly 100,000 documents, set `index_iterations` and `index_bulk_size` so that:

    Total documents = index_iterations × index_bulk_size

For example, to write 100,000 documents with a bulk size of 20:

```json
{
    "index_clients": 10,
    "index_bulk_size": 20,
    "index_iterations": 5000,
    "index_target_throughput": 25,
    "partitions": 1000
}
```

**Calculation:**
- Total documents = 5,000 iterations × 20 docs/bulk = 100,000 documents

You can adjust `index_bulk_size` and `index_iterations` as long as their product equals your desired total document count.