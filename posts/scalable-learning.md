# Scaling Deep Learning Training on Large Datasets

We often obsess over transformer architectures and optimizer states, but if you can't feed the GPU fast enough, those TFLOPS are wasted. I ran into this wall while building a sequence prediction system that processed high-frequency time series data across thousands of entities. The dataset wasn't "Google scale," but with thousands of entities generating thousands of data points per day across years of history, the raw parquet files quickly accumulated into multiple gigabytes of dense floating-point data—enough to crash standard development approaches.

This post documents the journey from a convenient but brittle approach to a robust, OS-native solution built around memory mapping.

---

## 1. The Naive Approach: "Just Load It"

The standard data science workflow is seductive in its simplicity:

1. Load everything into a DataFrame (`pd.read_parquet`)
2. Convert it to a tensor
3. Start training

**The failure mode** becomes obvious once you're dealing with real-world time series. In my case, I needed to generate sequences of N timesteps each, with variable lookhead windows for supervised learning targets. For N total data points and window size W, naive window expansion multiplies memory usage by roughly W. With multiple features per timestep across thousands of entities, this approach could reliably trigger the Out-Of-Memory (OOM) killer before training even began.

---

## 2. A First Optimization: Lazy File Loading

A seemingly obvious fix is lazy loading: don't load data until `__getitem__` is called.

* **Mechanism**: Store file paths. On each sample, open the parquet file, seek to the relevant rows, decode the data, apply feature engineering, and close the file.
* **Reality**: This shifts the bottleneck rather than removing it.

While modern NVMe drives offer excellent raw throughput, lazy loading performs many small, fragmented reads. Each access triggers:
- Parquet decompression and deserialization
- Forward-fill and backward-fill operations for missing values
- Feature selection and potential engineering
- StandardScaler transformation
- Tensor conversion

When multiplied by `(batch_size × num_workers × steps_per_epoch)`, this per-sample overhead quickly dominates. With my configuration of 256-sample batches and 4 workers with 10× prefetch, that's potentially 10,240 file operations *in flight at once*. The CPU becomes saturated orchestrating I/O and decoding work, while the GPU remains underutilized.

**Lesson:** Reducing I/O latency alone is insufficient. High-throughput training requires amortizing I/O and decoding costs by reading data in large, contiguous chunks and minimizing per-sample overhead.

---

## 3. HDF5: An Easy Solution That Didn't Quite Work

While surveying existing solutions, HDF5 appeared repeatedly as a common choice for large datasets. In theory, it promised structure, compression, and convenience. In practice, it introduced trade-offs that made it a poor fit for high-throughput training.

### The "GIL" of Data Access

Although HDF5 itself is C-based, most Python bindings (like `h5py`) rely on coarse-grained internal locking. With multiple `DataLoader` workers, reads frequently contend on a single lock, effectively serializing access and negating parallelism benefits.

### The Windows `spawn()` Problem

On Linux, `fork()` with Copy-on-Write allows child processes to inherit open file handles cheaply. Windows uses `spawn()`, where each worker starts fresh. HDF5 file handles aren't picklable, so every worker must independently `open()` the file, increasing overhead and lock contention.

For my use case—processing high-frequency time series data where disk was fast and raw throughput mattered more than format convenience—the complexity-to-benefit ratio wasn't worth it.

---

## 4. Letting the OS Do Its Job: Memory Mapping (`mmap`)

The real solution was to strip away abstraction layers and let the operating system manage memory the way it was designed to.

Memory-mapped files (`mmap`) map a file directly into a process's virtual address space:

* **Zero-copy semantics**: The file *is* the memory; no explicit serde, decompression, parsing etc.
* **Demand paging**: Pages load only when accessed
* **Graceful degradation**: When RAM is tight, clean pages are dropped automatically—performance may degrade, but the process stays alive
* **Shared page cache**: Multiple processes accessing the same file share OS-level cache

### A Flat Binary Layout

Complex formats introduce concurrency issues; columnar formats add decompression overhead. What we need is raw, contiguous `float` arrays.

I implemented a **two-pass preprocessing pipeline** producing a single flat binary file:

#### Pass 1: Global Statistics (Scan Phase)

```python
def _fit_scaler_and_get_stats(self):
    scaler = StandardScaler()
    total_rows = 0
    file_row_counts = []

    for file_path in self.raw_files:
        df = self._load_and_clean_df(file_path)  # Forward/backward fill
        df_features = self._feature_engineering(df)  # Select required features
        
        # Process in 100k-row chunks to avoid memory spikes
        for i in range(0, len(df_features), 100000):
            scaler.partial_fit(df_features.iloc[i:i + 100000])
        
        rows_in_file = len(df_features)
        total_rows += rows_in_file
        file_row_counts.append({'file': file_path, 'rows': rows_in_file})
    
    joblib.dump(scaler, self.scaler_path)
    return scaler, total_rows, file_row_counts
```

This pass:
- Streams through all raw parquet files sequentially
- Uses `StandardScaler.partial_fit()` to compute running normalization statistics
- Counts exact row totals without loading everything into memory
- Handles missing data with forward/backward fills during the scan

**Result**: A fitted scaler and exact size estimate for the final binary file.

#### Pass 2: Synthesis (Write Phase)

```python
def _create_unified_memmap_and_index(self, scaler, total_rows, file_row_counts):
    features_mmap_path = os.path.join(self.processed_dir, 'all_features.mmap')
    num_features = len(FEATURES)  # Number of features per timestep
    dtype = np.float32
    
    # Pre-allocate sparse file on disk
    if total_rows > 0:
        total_bytes = total_rows * num_features * np.dtype(dtype).itemsize
        with open(features_mmap_path, 'wb') as f:
            f.seek(total_bytes - 1)
            f.write(b'\0')
    
    # Open as memory-mapped array
    all_features_mmap = np.memmap(
        features_mmap_path, dtype=dtype, mode='r+', 
        shape=(total_rows, num_features)
    )
    
    # Stream and write
    index_map = []
    current_position = 0
    
    for file_info in file_row_counts:
        df = self._load_and_clean_df(file_info['file'])
        df_features = self._feature_engineering(df)
        scaled_data = scaler.transform(df_features).astype(dtype)
        
        rows = file_info['rows']
        all_features_mmap[current_position : current_position + rows] = scaled_data
        
        index_map.append({
            'entity': os.path.basename(file_info['file']),
            'start_row': current_position,
            'end_row': current_position + rows - 1
        })
        current_position += rows
```

This pass:
- Allocates a sparse file on disk using a seek-and-write trick
- Streams through raw data again
- Applies the fitted scaler
- Writes normalized `float32` arrays directly to their byte offsets

A companion `metadata.json` stores the index map plus training parameters:

```json
{
    "total_rows": 1000000000,
    "sequence_length": 3000,
    "num_features": 500,
    "features": ["feature_1", "feature_2", "feature_3", "feature_4", "feature_5"],
    "dtype": "float32",
    "index_map": [
        {"entity": "entity_001", "start_row": 0, "end_row": 2872401},
        {"entity": "entity_002", "start_row": 2872402, "end_row": 5984234}
    ]
}
```

---

## 5. A Subtle Concurrency Gotcha: Fork Safety

PyTorch's `DataLoader` with `num_workers > 0` relies on multiprocessing. On Linux, this uses `fork()`.

**The problem**: When a process forks, child processes inherit file descriptors. If a memory-mapped file is opened in the parent, all workers share the same underlying file descriptor and seek state. Concurrent access leads to incorrect reads or segmentation faults.

**The fix**: Reopen the memory-mapped file inside each worker process.

```python
class OnTheFlyDataset(Dataset):
    def __init__(self, processed_dir):
        self.metadata_path = os.path.join(processed_dir, 'metadata.json')
        with open(self.metadata_path, 'r') as f:
            self.metadata = json.load(f)
        
        self.features_path = os.path.join(processed_dir, 'all_features.mmap')
        self.features_mmap = None  # Initialize per-worker
        # ... other initialization ...
    
    def open_memmap(self):
        """Called once per worker process"""
        shape = (self.metadata['total_rows'], self.metadata['num_features'])
        self.features_mmap = np.memmap(
            self.features_path, 
            dtype=self.metadata['dtype'], 
            mode='r',  # Read-only in workers
            shape=shape
        )

def worker_init_fn(worker_id):
    worker_info = torch.utils.data.get_worker_info()
    dataset = worker_info.dataset
    dataset.open_memmap()  # Each worker gets its own file descriptor
```

This is **critical** when using `if __name__ == '__main__':` protection and `num_workers > 0`. Each worker now has its own file descriptor while still benefiting from shared OS page cache.

---

## 6. Efficient Index Resolution

With all data in a single binary blob, we need efficient sample lookup. The key insight: pre-compute a cumulative index over sequence counts.

```python
class OnTheFlyDataset(Dataset):
    def __init__(self, processed_dir):
        # ... load metadata ...
        
        # Calculate valid sequences per entity
        num_sequences_per_entity = []
        for entity_info in self.entity_info_map:
            rows = entity_info['end_row'] - entity_info['start_row'] + 1
            # Need seq_len + any additional lookahead for labels
            num_sequences = max(0, rows - self.seq_len - self.total_lookahead + 1)
            num_sequences_per_entity.append(num_sequences)
        
        # Cumulative sum enables O(log K) lookup
        self.cumulative_sequences = np.cumsum(num_sequences_per_entity)
        self.total_samples = self.cumulative_sequences[-1]
    
    def __getitem__(self, idx):
        if self.features_mmap is None:
            self.open_memmap()
        
        # Binary search to find which entity this index belongs to
        entity_idx = np.searchsorted(self.cumulative_sequences, idx, side='right')
        
        # Calculate offset within that entity's sequences
        start_of_entity = self.cumulative_sequences[entity_idx - 1] if entity_idx > 0 else 0
        offset = idx - start_of_entity
        
        # Map to absolute row positions
        entity_info = self.entity_info_map[entity_idx]
        sequence_start = entity_info['start_row'] + offset
        sequence_end = sequence_start + self.seq_len
        
        # Extract sequence features
        X_sample = self.features_mmap[sequence_start:sequence_end]
        
        # Calculate label based on your specific task
        # (This part varies depending on your prediction target)
        label = self._compute_label(sequence_end)
        
        return torch.from_numpy(X_sample.copy()).float(), torch.tensor(label, dtype=torch.long)
```

The cumulative index enables O(log K) lookup via binary search, where K is the number of entities—negligible compared to I/O and model computation.

---

## 7. Results and Takeaways

The difference was dramatic:

* **Startup time**: Near-instant once features are preprocessed and persisted; just pointer mapping, no data loading
* **RAM usage**: Flat and predictable; the OS manages paging automatically
* **GPU utilization**: Consistently high with 4 workers and 10× prefetch
* **Throughput**: Processed batches (25.6M samples) without I/O bottlenecks

My final configuration:
```python
BATCH_SIZE = 256
NUM_WORKERS = 4
PREFETCH_FACTOR = 10
```

With `persistent_workers=True` and `pin_memory=True` (for CUDA), the DataLoader maintains a steady stream of GPU-ready batches.

### Key Lessons

1. **Preprocessing is worth it**: The two-pass approach adds upfront cost but eliminates all per-sample overhead during training
2. **Trust the OS**: Memory mapping leverages decades of virtual memory optimization—don't reinvent it
3. **Concurrency matters**: The `worker_init_fn` pattern is essential for fork-safe multiprocessing
4. **Simple formats win**: Raw binary beats complex formats for high-throughput sequential access
5. **Profile everything**: My initial bottleneck wasn't disk I/O—it was repeated parquet decompression

By stripping away unnecessary abstraction layers and working with the OS rather than against it, the training pipeline became dramatically faster.For single-node deep learning on large sequential datasets, a raw binary format backed by `mmap` is **all you need**. 