# Random Vector Track

This track is designed for benchmarking filtered search on random vectors.

**Default Challenge:**

As of the latest update, the default challenge is now `ingest-autoscale`, which benchmarks bulk indexing with configurable steps and simulates auto-scaled workloads. This challenge is designed to test how the system handles bursty or variable ingest rates, using arrays of parameters to simulate changes in load and client concurrency.

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

#### Additional Example: Autoscaling with Multiple Phases

The following parameter file demonstrates autoscaling with four phases, each with different client counts, throughputs, and iteration counts:

```
{
    "vector_index_type": "flat",
     "dims": 1596,
     "as_ingest_clients" : [10,20,10,20],
     "as_ingest_bulk_size": [18,18,18,18],
     "as_ingest_target_throughputs": [8,24,4,25],
     "as_ingest_index_iterations": [100,50000,3000,20000]
}
```

This configuration will run four ingest phases, simulating changes in load and system scaling over time.

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

## Challenges

### ingest-autoscale (default)

This is now the default challenge. It benchmarks bulk indexing with configurable steps, simulating auto-scaled or bursty workloads. You can control the number of clients, bulk sizes, target throughputs, and iteration counts for each phase of the workload. The challenge is defined in `challenges/default.json` and uses the schedule in `challenges/common/ingest-autoscale-schedule.json`.

Example parameters for autoscale:

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

### index-and-search

This challenge is still available but is no longer the default. It creates an index and indexes documents with random content, then runs various search operations.

## Running the Track

To run the random_vector track with autoscaling parameters, use the following example command:

```bash
esrally race --track-path=/root/rally-tracks/random_vector \
  --target-hosts=$eshost \
  --track-params=random-vector-params.json \
  --pipeline=benchmark-only \
  --client-options="use_ssl:true,verify_certs:false,api_key:$apikey" \
  --kill-running-processes
```

- `--track-path`: Path to the local track directory.
- `--target-hosts`: Comma-separated list of Elasticsearch hosts (set `$eshost` to your cluster endpoint).
- `--track-params`: Path to your parameter file (e.g., `random-vector-params.json`).
- `--pipeline`: Use `benchmark-only` for running benchmarks without setup/teardown.
- `--client-options`: Connection options (SSL, cert verification, API key, etc.).
- `--kill-running-processes`: Ensures any previous Rally processes are stopped before starting.

Adjust the parameters as needed for your environment and workload.