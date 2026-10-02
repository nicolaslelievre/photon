# Local Photon Setup — US Address Standardization

This sets up a local [Photon](https://github.com/komoot/photon) geocoder instance
(native Java, no containers) to standardize a batch of US addresses (e.g. ~100k
records) via its local HTTP API.

## Requirements

- **Java 21+** — check with:
  ```bash
  java -version
  ```
  Install OpenJDK 21 or Eclipse Temurin 21 if missing.
- **Disk space** — Photon's official docs only confirm the **planet-wide** dump at
  ~95 GB. A US-only extract will be considerably smaller, but there's no official
  fixed number — check the actual file size on the download page before you
  commit disk space.
- **SSD/NVMe** strongly recommended for indexing and query throughput.
- **RAM** — official guidance recommends ~64 GB for smooth operation at scale.
  If your machine has less, you can increase the JVM heap with `-Xmx` (see
  Step 3), but keep enough RAM free that the system doesn't start swapping.

## Step 1: Download the Photon Executable

1. Go to the [Photon GitHub Releases page](https://github.com/komoot/photon/releases/latest).
2. Download the latest release JAR (e.g. `photon-1.3.0.jar`).
3. Place it in your working directory, e.g. `~/photon/`.

## Step 2: Download & Extract the US Database Dump

GraphHopper hosts pre-built, "unzip-and-go" Photon database dumps at:

[https://download1.graphhopper.com/public](https://download1.graphhopper.com/public)

1. Open the page and locate the dump that matches your needs (filenames start
   with `photon-db-` and end in `.tar.bz2`). **Confirm the exact current
   filename on the page itself** — don't assume a name like `photon-db-us.tar.bz2`,
   as naming may change between releases.
2. Make sure the filename matches **your Photon jar version** (check the version
   note on the download page).
3. Extract directly into your working directory using `bzip2` or `pbzip2`
   piped into `tar` (do **not** use plain `tar -xjvf`, and do **not** use WinRAR
   on Windows — it's known to corrupt these archives):

   **macOS / Linux:**
   ```bash
   # Standard bzip2
   wget -O - https://download1.graphhopper.com/public/<dump-filename>.tar.bz2 | bzip2 -cd | tar x

   # Faster, if pbzip2 is installed (recommended)
   wget -O - https://download1.graphhopper.com/public/<dump-filename>.tar.bz2 | pbzip2 -cd | tar x
   ```

   **Windows:** Use 7-Zip to unpack the `.tar.bz2` archive (not WinRAR).

4. **Directory check**: after extraction you should have a `photon_data` folder
   sitting in the **same directory** as your `photon-*.jar` — the jar must be
   run from this parent directory.

> **Caution:** When updating to a newer dump later, never unpack it on top of
> an existing `photon_data` directory — this corrupts the data. Extract the new
> dump into a separate directory, swap it in, verify, then delete the old one.

## Step 3: Launch the Local Server

From your working directory:

```bash
cd ~/photon
java -jar photon-1.3.0.jar serve
```

**Memory tip:** if you have RAM to spare and want extra heap for a large batch
job, increase the JVM heap:

```bash
java -Xmx8G -jar photon-1.3.0.jar serve
```

Once loaded, the server listens at `http://localhost:2322`.

## Step 4: Verify & Run Batch Cleaning

**Verify in a browser:**

```
http://localhost:2322/api?q=1600+Pennsylvania+Ave+NW+Washington+DC
```

You should get a structured JSON response with normalized street, city, state,
and zip properties.

**Run the batch script:**

Execute your Python asyncio script against `http://localhost:2322`. Since the
API runs locally against local disk, you can typically raise
`asyncio.Semaphore` well above what you'd use against a remote API (e.g.
100–200 concurrent requests) — tune this based on your own machine's
throughput rather than assuming a fixed processing time.
