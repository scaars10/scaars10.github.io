# Exploring Scalable Deep Learning Training on Large Datasets

We often obsess over transformer architectures and optimizer states, but if you can’t feed the GPU fast enough, those TFLOPS are wasted. I ran into this wall while training sequence models as both the number of data sources and the volume of my data kept growing. The dataset wasn’t “Google scale,” but it was large enough (multiple gigabytes of dense floating‑point data) to crash a standard development machine and choke a naive disk-based loader. Fortunately I had enough RAM at the time to get around with simple workarounds but I knew I needed a more robust solution.


This post documents the journey from a convenient but brittle approach to a robust, OS-native solution built around memory mapping.

---

## 1. The Naive Approach: “Just Load It”

The standard data science workflow is seductive in its simplicity:

1. Load everything into a DataFrame (`pd.read_parquet`)
2. Convert it to a tensor
3. Start training

**The failure mode** becomes obvious once datasets grow beyond toy scale. Python objects carry significant overhead, and even NumPy arrays become expensive when building sliding windows for sequence models. For a time series of length (N) and window size (W), naive window expansion multiplies memory usage by roughly (W).

This approach reliably runs into the Out‑Of‑Memory (OOM) killer long before training begins.

---

## 2. A First Optimization: Lazy File Loading

A seemingly obvious fix is lazy loading: don’t load data until `__getitem__` is called.

* **Mechanism**: Store file paths or offsets. On each sample or batch, open the file, seek to the relevant region, read the required slice, decode it (e.g., CSV/Parquet → NumPy), and close the file.
* **Reality**: This shifts the bottleneck rather than removing it.

While modern NVMe drives offer excellent raw throughput, lazy loading typically performs many small, fragmented reads. The cost is not the `open()` call itself (which is usually microseconds) but the repeated decoding, allocation, and filesystem bookkeeping that follows. Each access triggers page cache checks, potential page faults, and format parsing, all of which are CPU-heavy operations.

When multiplied by `(batch_size × num_workers × steps_per_epoch)`, this per-sample overhead quickly dominates. The CPU becomes saturated orchestrating I/O and decoding work, while the GPU remains underutilized, waiting for batches to arrive.

**Lesson:** Reducing I/O latency alone is insufficient. High-throughput training requires amortizing I/O and decoding costs by reading data in large, contiguous chunks and minimizing per-sample overhead.

---

## 3. HDF5: An Easy Solution That Didn’t Quite Work

While surveying existing solutions, HDF5 appeared repeatedly as a common and well-established choice for large datasets. In theory, it promised structure, compression, and convenience. In practice, it introduced trade-offs that made it a poor fit for high-throughput training.

**Lesson:** HDF5 is excellent for archival and portability, but painful for concurrent, high-throughput deep learning, especially on Windows.

1. **The “GIL” of Data Access**
   Although HDF5 itself is implemented in C, most Python bindings (such as `h5py`) rely on coarse-grained internal locking to ensure safety. With multiple `DataLoader` workers, reads frequently contend on a single lock, effectively serializing access and negating the benefits of parallelism.

2. **The Windows `spawn()` Problem**
   On Linux, `fork()` with Copy‑on‑Write allows child processes to inherit open file handles cheaply. Windows uses `spawn()`, where each worker starts as a fresh Python interpreter. HDF5 file handles are not picklable, so they cannot be shared across processes. Every worker must independently `open()` the file, increasing overhead and amplifying internal lock contention.

In my case, disk was fast and raw throughput mattered more than format convenience. The complexity-to-benefit ratio of making HDF5 work reliably with PyTorch’s `DataLoader` on Windows simply wasn’t worth it.

---

## 4. Letting the OS Do Its Job: Memory Mapping (`mmap`)

The real solution was to rely on the convenience and feature sets of the operating system and let it manage memory the way it was designed to.

Memory-mapped files (`mmap`) map a file directly into a process’s virtual address space:

* **Zero-copy semantics**: The file is the memory; no explicit reads into user-space buffers.
* **Demand paging**: Pages are loaded only when accessed.
* **Graceful eviction**: When RAM is under pressure, clean pages are dropped automatically. Performance may degrade, but the process stays alive.

### A Flat Binary Layout

To make this work efficiently, the data format must be simple. Complex formats like HDF5 introduce concurrency issues; columnar formats like Parquet add decompression overhead. What we need instead is raw, contiguous bytes.

I implemented a **two-pass preprocessing pipeline** to produce a single flat binary file.

#### Pass 1: Global Statistics (Scan Phase)

Normalization requires global statistics, but loading the entire dataset is infeasible.

* Stream through the raw files
* Use `StandardScaler.partial_fit()` to compute running statistics
* Count total rows

**Result**: A fitted scaler and an exact size estimate for the final binary file.

#### Pass 2: Synthesis (Write Phase)

* Allocate a large sparse file (`all_features.mmap`) on disk
* Stream through the raw data again
* Apply normalization
* Write `float32` arrays directly into their assigned byte offsets

A companion `metadata.json` acts as a lightweight index, mapping high-level identifiers (e.g., stock symbols) to byte ranges in the binary file.

---

## 5. A Subtle Concurrency Gotcha: Fork Safety

PyTorch’s `DataLoader` relies on multiprocessing. On Linux, this means `fork()`.

**The problem:**
When a process forks, child processes inherit file descriptors. If a memory-mapped file is opened in the parent, all workers share the same underlying file descriptor and seek state. Concurrent access can lead to incorrect reads or hard-to-debug segmentation faults.

**The fix:**
Reopen the memory-mapped file inside each worker process.

```python
def worker_init_fn(worker_id):
    dataset = torch.utils.data.get_worker_info().dataset
    dataset.open_memmap()  # reopen after fork
```

Each worker now has its own file descriptor while still benefiting from shared OS page cache.

---

## 6. Efficient Index Resolution

With all data stored in a single binary blob, individual samples still need to be resolved efficiently. I used a cumulative index over sequence counts:

* Build a cumulative sum array of sequence counts per symbol
* Use `np.searchsorted` to locate the owning segment

This lookup runs in `O(log K)` time, where `K` is the number of symbols—negligible compared to I/O and model computation.

---

## 7. Results and Takeaways

The difference was dramatic:

* **Startup time**: Near-instant; just pointer mapping
* **RAM usage**: Flat and predictable; terabytes of data with tens of gigabytes of RAM
* **GPU utilization**: Consistently high; data loading is no longer the bottleneck

By stripping away unnecessary abstraction layers and moving closer to the OS, the training pipeline became much faster. For single-node deep learning on large datasets, a raw binary format backed by `mmap` is **all you need**.
