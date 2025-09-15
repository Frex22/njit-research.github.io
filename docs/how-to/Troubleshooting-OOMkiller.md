# HPC Guide: Troubleshooting Out-of-Memory (OOM) Errors

### 1. Understanding the OOM Killer

*   **What it is:** The Out-of-Memory (OOM) Killer is a Linux kernel process that activates when a compute node is critically low on physical memory.
*   **Its Purpose in HPC:** To protect the node from crashing. By "killing" one memory-intensive process, it saves all other user jobs running on that same node. This is a stability feature, not a bug.
*   **How it Works:** In an HPC environment, your job scheduler (e.g., Slurm) assigns your job a specific memory limit (a "cgroup"). If your job exceeds this limit, the OOM Killer is invoked to terminate it.

---

### 2. Step 1: Confirming an OOM Kill

Before troubleshooting, verify that an OOM event was the cause of failure. Look for these signs:

*   **Exit Code:** The job failed with an exit code of **`137`**. (This means it was terminated by `SIGKILL`, signal 9).
*   **Scheduler Log Files (`slurm-JOBID.out`):** The output file will contain an explicit message.
    ```
    slurmstepd: error: Job 123456 exceeded memory limit (16500 MB > 16384 MB), being killed
    slurmstepd: error: Out of Memory
    ```
*   **Scheduler Accounting Commands:** Use these commands to get a post-mortem report.
    *   **`sacct` command:**
        ```bash
        sacct -j <Your-Job-ID> -o JobID,State,ExitCode,MaxRSS,ReqMem
        ```
        **What to look for:** The `MaxRSS` (peak memory used) will be equal to or slightly higher than `ReqMem` (memory requested).
    *   **`seff` command (highly recommended):**
        ```bash
        seff <Your-Job-ID>
        ```
        **What to look for:** A "Memory Efficiency" percentage over 100% and a clear statement that the job was killed by OOM.

---

### 3. Common OOM Scenarios and Solutions

#### Scenario A: Simple Memory Under-Allocation

*   **Symptom:** Job fails quickly. `MaxRSS` is slightly higher than `ReqMem`.
*   **Cause:** The application's peak memory usage was simply higher than the limit requested in the submission script.
*   **Solution:**
    1.  Use the `MaxRSS` value from the failed job as your new baseline.
    2.  Calculate a new, safer memory request: `New Request = MaxRSS + 20%` buffer.
    3.  Update the memory flag in your submission script (e.g., `#SBATCH --mem=24G`) and resubmit.

---

#### Scenario B: Python/Pandas Data Loading

*   **Symptom:** A Python script fails when reading a large file (e.g., CSV, JSON), even when requested memory is 2-3x the file's size on disk.
*   **Cause:** In-memory data structures are far less efficient than their on-disk representation. A 10 GB CSV file can easily expand to 40-50 GB of RAM when loaded into a Pandas DataFrame.
*   **Solution (Do not just request more memory):**
    1.  **Process in Chunks:** Avoid loading the entire file at once. Use an iterator.
        ```python
        # Bad: Loads everything into RAM
        df = pd.read_csv("huge_file.csv")

        # Good: Processes the file in manageable pieces
        for chunk in pd.read_csv("huge_file.csv", chunksize=1000000):
            process(chunk) # Your logic here
        ```
    2.  **Use Efficient Libraries:** For very large data, consider using libraries like **Dask** or **Polars**, which are designed for out-of-core (larger than RAM) computation.

---

#### Scenario C: Multi-Processing Memory Multiplication

*   **Symptom:** A single-core job runs fine, but the multi-core version is OOM-killed, even after scaling the memory request linearly with the core count.
*   **Cause:** When a parent process creates workers (e.g., with Python's `multiprocessing.Pool`), each worker process gets a **copy** of the parent's memory. If a large object (like a model or dataset) is loaded before the workers are created, its memory usage is multiplied by the number of workers.
*   **Solution:**
    1.  **Load Data *Inside* the Worker:** Initialize large objects and load data within the worker function itself, not in the main script's global scope. This isolates the memory to each process.
        ```python
        # Bad Pattern
        large_model = load_model() # Loaded once, copied to all workers
        with Pool(processes=8) as pool:
            pool.map(process_data, my_items)

        # Good Pattern
        def process_data_with_init(item):
            large_model = load_model() # Loaded independently in each worker
            return large_model.predict(item)

        with Pool(processes=8) as pool:
            pool.map(process_data_with_init, my_items)
        ```

---

#### Scenario D: Memory Leaks in Long-Running Jobs

*   **Symptom:** A job runs successfully for hours or days, then fails from OOM. Monitoring shows its memory usage climbing steadily over time.
*   **Cause:** The application continuously allocates new memory inside a loop but never releases the old memory.
*   **Solution:**
    1.  **Verify the Leak:** While the job is running, monitor its memory usage periodically with `sstat -j <Job-ID> --format=MaxRSS`. A constantly increasing value confirms a leak.
    2.  **Inspect the Code:**
        *   **Python:** Look for a list or dictionary that is appended to inside a main loop but never cleared or written to disk.
        *   **C/C++/Fortran:** This is a classic `malloc()` without a matching `free()`. Use a memory profiling tool like **Valgrind (Massif)** to find the unreleased allocations.
    3.  **Refactor:** Modify the code to write results to a file periodically instead of collecting them all in a single in-memory object.

---

### 4. Quick Troubleshooting Reference Table

| Symptom                                                     | Likely Cause                                  | Recommended Solution                                                                                                   |
| ----------------------------------------------------------- | --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Job fails quickly, `ExitCode 137`.                          | Simple Under-allocation                       | Increase `--mem` request to `MaxRSS + 20%`.                                                                            |
| Python/Pandas script dies reading a large file.             | Loading entire file into memory at once.      | Use `pd.read_csv(..., chunksize=...)` to process the file in pieces.                                                   |
| Multi-core job uses far more memory than expected.          | Memory is copied to each worker process.      | Load large data/models *inside* the worker function, not before creating the process pool.                             |
| Job runs for a long time, then fails with OOM.              | Software memory leak.                         | Inspect code for lists/objects that grow in a loop. Periodically write results to disk instead of holding them in RAM. |
| MPI job fails, and `sacct` shows rank 0 used all the memory. | Rank 0 is an I/O bottleneck.                  | Refactor to use parallel I/O (e.g., MPI-IO). Have each rank read its own portion of the data directly from the disk.  |
