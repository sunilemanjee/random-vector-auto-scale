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
| `as_ingest_index_iterations`    | List of iteration counts (number of bulks to send) for each phase                           |
| `parallel_indexing_clients`     | Number of clients for parallel indexing operations                                           |
| `parallel_ingest_target_throughputs` | Target throughput for parallel indexing operations                                      |
| `parallel_indexing_iterations`  | Number of iterations for parallel indexing operations                                        |
| `parallel_indexing_bulk_size`   | Bulk size for parallel indexing operations                                                   |
| `parallel_search_clients`       | Number of clients for parallel search operations                                             |
| `parallel_search_iterations`    | Number of iterations for parallel search operations                                          |

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
    "as_ingest_clients": [10, 20, 10, 20, 10, 20],
    "as_ingest_bulk_size": [18, 18, 18, 18, 18, 18],
    "as_ingest_target_throughputs": [8, 24, 4, 25, 8, 24],
    "as_ingest_index_iterations": [2500, 5000, 1500, 7000, 2200, 10000],
    "parallel_warmup_time_periods": 10,
    "parallel_time_periods": 10,
    "parallel_indexing_clients": 1,
    "parallel_ingest_target_throughputs": 1,
    "parallel_indexing_bulk_size": 10,
    "parallel_search_clients": 10
}
```

This configuration will run six ingest phases with varying client counts, throughputs, and iterations, followed by parallel operations for both indexing and search. The workload simulates:
- Multiple phases of varying intensity (8-25 bulk requests/sec)
- Different client concurrency levels (10-20 clients)
- Consistent bulk sizes (18 documents per bulk)
- Parallel operations with 1 indexing client and 10 search clients

Adjust the parameters as needed for your environment and workload.

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

This is now the default challenge. It benchmarks bulk indexing with configurable steps, simulating auto-scaled or bursty workloads. The challenge includes both sequential auto-scaling phases and parallel operations for both indexing and search. You can control:

1. Auto-scaling parameters:
   - Number of clients
   - Bulk sizes
   - Target throughputs
   - Iteration counts for each phase

2. Parallel operations:
   - Parallel indexing with configurable clients and throughput
   - Parallel search with configurable clients and iterations

The challenge is defined in `challenges/default.json` and uses the schedule in `challenges/common/ingest-autoscale-schedule.json`.

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

To run the random_vector track with autoscaling parameters, you first need to create a parameter file (e.g., `random-vector-params.json`) with your desired settings. For example:

```json
{
    "vector_index_type": "flat",
    "dims": 1596,
    "as_ingest_clients": [10, 20, 10, 20, 10, 20],
    "as_ingest_bulk_size": [18, 18, 18, 18, 18, 18],
    "as_ingest_target_throughputs": [8, 24, 4, 25, 8, 24],
    "as_ingest_index_iterations": [2500, 5000, 1500, 7000, 2200, 10000],
    "parallel_warmup_time_periods": 10,
    "parallel_time_periods": 10,
    "parallel_indexing_clients": 1,
    "parallel_ingest_target_throughputs": 1,
    "parallel_indexing_bulk_size": 10,
    "parallel_search_clients": 10
}
```

Then, use the following example command:

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

This configuration will run six ingest phases with varying client counts, throughputs, and iterations, followed by parallel operations for both indexing and search. The workload simulates:
- Multiple phases of varying intensity (8-25 bulk requests/sec)
- Different client concurrency levels (10-20 clients)
- Consistent bulk sizes (18 documents per bulk)
- Parallel operations with 1 indexing client and 10 search clients

Adjust the parameters as needed for your environment and workload.

# Ingest Autoscale Schedule Parameters

This document describes the parameters used in the ingest autoscale schedule for vector indexing and search operations.

## Vector Index Configuration
- **Vector Index Type**: `flat`
- **Dimensions**: `1596`

## Ingest Parameters
The ingest operations are configured with the following parameters:

### Client Configuration
- **Ingest Clients**: `[10, 20, 10, 20, 10, 20]`
  - Number of clients for each ingest operation phase

### Bulk Size
- **Ingest Bulk Size**: `[18, 18, 18, 18, 18, 18]`
  - Number of documents per bulk request for each phase

### Target Throughput
- **Ingest Target Throughputs**: `[8, 24, 4, 25, 8, 24]`
  - Target throughput (operations per second) for each phase

### Iterations
- **Ingest Index Iterations**: `[2500, 5000, 1500, 7000, 2200, 10000]`
  - Number of iterations for each ingest phase

## Parallel Operations Configuration

### Time Periods
- **Warmup Time Periods**: `10`
- **Time Periods**: `10`

### Parallel Indexing
- **Parallel Indexing Clients**: `1`
- **Parallel Ingest Target Throughputs**: `1`
- **Parallel Indexing Bulk Size**: `10`

### Parallel Search
- **Parallel Search Clients**: `10`

## Schedule Structure
The schedule consists of the following operations in sequence:
1. Create ingest pipeline
2. Delete data stream
3. Delete data stream template
4. Create data stream template
5. Create data stream
6. Check cluster health
7. Multiple ingest operations with varying parameters
8. Parallel operations including:
   - Random bulk indexing
   - KNN filtered search with multiple clients