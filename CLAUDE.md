# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a block storage benchmark suite implementing the SNIA Solid State Storage (SSS) Performance Test Specification (PTS) Enterprise v1.1. The suite uses `fio` (Flexible I/O tester) for actual storage testing, PHP for automation and test orchestration, and generates reports with gnuplot and headless Chrome.

**Fork context:** This is the `ixsystems` branch, a fork of [cloudharmony/block-storage](https://github.com/cloudharmony/block-storage) with FreeBSD/TrueNAS-specific adaptations. Key differences from upstream include `camcontrol sanitize` support, `diskinfo -v` for SSD detection, and `sysctl`-based system info collection.

**Key Specifications:**
- SNIA PTS specification: https://www.snia.org/sites/default/files/SSS_PTS_Enterprise_v1.1.pdf
- Tests measure block storage performance across IOPS, throughput, latency, and other characteristics
- Designed for cloud/virtualized environments with limited hardware access

## Running Tests

```bash
# Run IOPS test on single device
sudo ./run.sh --target=/dev/sdc --test=iops

# Run multiple tests on multiple devices
sudo ./run.sh --target=/dev/sdc --target=/dev/sdd --test=iops --test=throughput --test=latency --threads=4 --oio_per_thread=32

# Run test with metadata for report generation
sudo ./run.sh \
  --meta_compute_service "Google Compute Engine" \
  --meta_provider Google \
  --meta_region us-central1 \
  --meta_instance_id n1-standard-16 \
  --meta_storage_config "SSD PD" \
  --target=/dev/sdb \
  --test=iops \
  --threads=4 \
  --verbose

# Skip purge and preconditioning (faster, but not SNIA compliant)
sudo ./run.sh --target=/dev/sdc --test=iops --nopurge --noprecondition

# FreeBSD/TrueNAS NVMe example
sudo ./run.sh --target=/dev/nvd0 --test=iops --test=throughput --threads=8 --verbose
```

All parameters can also be set via `bm_param_`-prefixed environment variables (e.g., `export bm_param_test=iops`).

### Saving Results

```bash
./save.sh                          # Save to CSV (default)
./save.sh /path/to/results         # Save from specific directory
./save.sh --db postgresql --db_user dbuser --db_pswd dbpass --db_host db.example.com --db_name benchmarks
```

### Dependencies

```bash
# Ubuntu/Debian
sudo apt-get install gnuplot php-cli util-linux zip fio

# CentOS/RHEL
sudo yum install fio gnuplot php-cli util-linux zip

# FreeBSD/TrueNAS
# fio, gnuplot, php available via pkg; camcontrol and diskinfo are base system
```

PDF reports require headless Chrome/Chromium (`chrome --headless --print-to-pdf`). Gnuplot SVG charts are embedded directly in the HTML reports.

## Architecture

### Test Lifecycle

Execution follows a three-phase approach per SNIA specifications:

1. **Purge Phase**: Prepares storage to a known state
   - Order of precedence: ATA secure erase -> `camcontrol sanitize` (FreeBSD) -> TRIM (SSD only) -> zero fill
   - Controlled by `--nopurge`, `--nosecureerase`, `--notrim`, `--nozerofill` flags

2. **Workload Independent Preconditioning (WIPC)**: Sequential writes across full LBA range
   - Default: 2X 128K sequential write passes
   - Controlled by `--noprecondition`, `--precondition_passes` flags

3. **Workload Dependent Preconditioning & Testing (WDPC)**: Test-specific conditioning and measurement
   - Runs test workload until steady state is achieved
   - Steady state: 5 consecutive rounds within variance threshold (default +-10%)
   - Checks both data excursion (max-min) and slope excursion (linear regression)
   - Max rounds: `--ss_max_rounds` (default 25), variance: `--ss_verification` (default 10%)

### Class Hierarchy

```
BlockStorageTest (abstract base class)
  Implements: purge(), wipc(), wdpc(), steady state detection, report generation
  Must implement: getReportContent(), getReportSections(), getSubtitle(), jobMetrics()

  BlockStorageTestIops      - IOPS across block sizes (512b-1m) and read/write mixes
  BlockStorageTestThroughput - 128k/1m sequential read/write throughput (MB/s)
  BlockStorageTestLatency    - Response times (0.5k, 4k, 8k) at queue depth 1
  BlockStorageTestWsat       - Write Saturation - continuous 4k writes over time
  BlockStorageTestHir        - Host Idle Recovery - background garbage collection
  BlockStorageTestXsr        - Cross Stimulus Recovery (NOT IMPLEMENTED)
  BlockStorageTestEcw        - Enterprise Composite Workload (NOT IMPLEMENTED)
  BlockStorageTestDirth      - Demand Intensity Response Time Histogram (NOT IMPLEMENTED)
```

### Core Files

**Entry points:**
- `run.sh` -> `lib/run.php` - Test execution (validates args, creates test controller, runs purge/wipc/wdpc lifecycle, generates reports)
- `save.sh` -> `lib/save.php` - Result persistence (CSV, BigQuery, MySQL, PostgreSQL, Librato, S3/GCS/Azure)

**Core logic:**
- `lib/BlockStorageTest.php` - Abstract base class: FIO command generation, steady state detection, report generation, purging
- `lib/BlockStorageTest{Iops,Throughput,Latency,Wsat,Hir}.php` - Test implementations
- `lib/util.php` - System info collection, file operations, formatting, shell execution helpers

**Configuration:**
- `benchmark.ini` - Benchmark metadata (name, version, artifact patterns for save)
- `run-params.ini` - Parameter definitions with defaults and validation regexes

**Result persistence backends:** `lib/save/BenchmarkDb*.php` (BigQuery, MySQL, PostgreSQL, Callback, Librato, Store)
**Cloud storage archivers:** `lib/save/BenchmarkArchiver{S3,Google,Azure}.php`

### Key Concepts

**Targets**: Block devices (`/dev/sdc`, `/dev/nvd0`) or mounted volumes (`/mnt/volume`)
- Devices: Full LBA range used, data is erased
- Volumes: Uses free space minus 100MB buffer, creates test files

**Threads and Queue Depth**:
- `--threads`: Concurrent fio jobs per target (default: CPU core count, capped by `--threads_per_target_max=8` and `--threads_per_core_max=2`)
- `--oio_per_thread`: Queue depth per thread (default: 64)
- Total outstanding I/O = targets x threads x oio_per_thread

**Test Output**:
- `[test].json` - Test results summary
- `fio-[test].json` - Detailed fio job results for all rounds
- `report.zip` - HTML report with embedded SVG graphs (gnuplot)
- `report.pdf` - PDF version (via headless Chrome)

## FreeBSD/TrueNAS Specifics (ixsystems branch)

This branch contains platform-specific code paths activated when `uname -s` returns `FreeBSD`:

**SSD detection** (`BlockStorageTest::isRotational()`): Uses `diskinfo -v | grep RPM` instead of Linux `/sys/block/*/queue/rotational`. Output must be `trim()`'d and cast to `(int)` for correct comparison (see BUGFIXES.md).

**Purge via camcontrol** (`BlockStorageTest::purge()`): Uses `camcontrol sanitize` for ATA/SCSI drives on FreeBSD. Detects sanitize capability before attempting. Uses 3-parameter `exec()` to capture exit codes correctly.

**System info** (`util.php::get_sys_info()`): Uses `sysctl -n hw.model`, `sysctl -n hw.ncpu`, `sysctl -n dev.cpu.0.freq`, `sysctl -n hw.physmem` for CPU/memory detection.

## Development Notes

### PHP Version Compatibility

Requires PHP 7.0+ (`__construct()` constructors). PHP 8+ compatibility fixes have been applied.

### FIO Integration

FIO job files are generated dynamically. Key mappings:
- `--oio_per_thread` -> fio `--iodepth`
- `--threads` -> fio `--numjobs`
- FIO version 2.1+ required (validated via `BlockStorageTest::validateFio()`)

### Common Bug Patterns

Based on fixes in BUGFIXES.md, watch for these patterns when modifying FreeBSD code paths:
- **Always `trim()` shell command output** before comparisons (`shell_exec()` returns trailing newlines)
- **Use `(int)` casts** for numeric comparisons from string sources (PHP loose comparison pitfalls)
- **Use 3-parameter `exec($cmd, $output, $exitcode)`** to capture exit codes (single-parameter `exec()` returns last output line, not exit code)
- **Variable naming**: the codebase uses `$sys_info` (with underscore) in `get_sys_info()` - not `$sysinfo`

### Extending with New Tests

1. Create `lib/BlockStorageTest{Name}.php` extending `BlockStorageTest`
2. Implement abstract methods: `getReportContent()`, `getReportSections()`, `getSubtitle()`, `jobMetrics()`, `getTestParameters()`, `getSetupParameters()`
3. Override `wdpc()` to define test-specific workload sequences
4. Add test identifier to `BlockStorageTest::getSupportedTests()`
5. Update `BlockStorageTest::getTestController()` to instantiate your test class

### Large Drive Testing (100TB+)

Validated on 122.88TB NVMe drives. Key considerations:
- Default 24h `--timeout=86400` is insufficient for drives >150TB; use `--timeout=604800` (7 days)
- WIPC scales linearly with capacity (~25.5h for 122TB at 2.5GB/s)
- `--ss_max_rounds` is capped at 100 in validation; increase for very large drives
- `--active_range` can reduce tested LBA range (faster but less accurate)
- `--precondition_time` can set a fixed WIPC duration instead of full passes
