<h1 align="center">Why Your Pipeline Is Slow or Stalled: Troubleshooting CPU, Memory, and I/O Bottlenecks on a Single Workstation</h1>

<h3 align="center">M. Asaduzzaman Prodhan<sup>*</sup> </h3>

<div align="center"><b> School of Biological Sciences, The University of Western Australia </b></div>

<div align="center"><b> 35 Stirling Highway, Perth, WA 6009, Australia. <sup>*</sup>Correspondence: prodhan82@gmail.com </b></div>

<br />

<p align="center">
  <a href="https://github.com/asadprodhan/Microbiome_Metagenomic_Analysis_Roadmap_From_Reads_to_Abundance_Table#GPL-3.0-1-ov-file"><img src="https://img.shields.io/badge/License-GPL%203.0-yellow.svg" alt="License GPL 3.0" style="display: inline-block;"></a>
  <a href="https://orcid.org/0000-0002-1320-3486"><img src="https://img.shields.io/badge/ORCID-green?style=flat-square&logo=ORCID&logoColor=white" alt="ORCID" style="display: inline-block;"></a>
</p>

<br />

---

## **CONTENT**

- [Learning Objective](#learning-objective)
- [Step 1 — Check pipeline status](#step-1--check-pipeline-status)
- [Step 2 — Diagnose system resources](#step-2--diagnose-system-resources)
  - [uptime](#uptime)
  - [free -h](#free--h)
  - [vmstat 2 5](#vmstat-2-5)
- [System Bottleneck Flow](#system-bottleneck-flow)
- [Step 3 — Identify heavy pipelines](#step-3--identify-heavy-pipelines)
- [Step 4 — Reduce contention](#step-4--reduce-contention)
- [Step 5 — Confirm recovery](#step-5--confirm-recovery)
- [Step 6 — Check active jobs](#step-6--check-active-jobs)
- [Step 7 — Monitor progress](#step-7--monitor-progress)
- [Step 8 — Check task results](#step-8--check-task-results)
- [Step 9 — Check logs](#step-9--check-logs)
- [Step 10 — Resume safely](#step-10--resume-safely)
- [Key Learning Points](#key-learning-points)
- [Recommended Settings](#recommended-settings)
- [Teaching Insight](#teaching-insight)
  
<br />

---

## Learning Objective

Diagnose and resolve stalled Nextflow pipelines caused by CPU, memory, and disk I/O contention in a single-workstation environment.

---

## Step 1 — Check pipeline status

```bash
ps -ef | grep -i nextflow | grep -v grep
```

**Interpretation**
- Processes present → pipeline running  
- No process → pipeline stopped  

---

## Step 2 — Diagnose system resources

```bash
uptime
nproc
free -h
vmstat 2 5
```

### `uptime`
- Shows system load average (1, 5, 15 min)
- Compare with `nproc` (CPU cores)

**Interpretation**
- Load ≈ cores → fully utilised  
- Load >> cores → overloaded  

---

### `free -h`
Example:
```
              total   used   free   buff/cache   available
Mem:          125Gi   38Gi   19Gi   68Gi         87Gi
Swap:         8.0Gi   8.0Gi  0Gi
```

**Key fields**
- `used` → actively used memory  
- `free` → unused memory  
- `buff/cache` → filesystem cache (GOOD, not a problem)  
- `available` → real usable memory  
- `Swap` → overflow memory  

**Interpretation**
- High swap usage → memory pressure ⚠️  
- High cache → normal (do not clear)  
- Low available → risk of swapping  

---

### `vmstat 2 5`
Example columns:
```
r  b   swpd   free   si   so   wa
6 15 8383860 20786332 722 1065 48
```

**Key fields**
- `r` → runnable processes (CPU demand)  
- `b` → blocked processes (I/O wait)  
- `swpd` → swap used  
- `si/so` → swap in/out  
- `wa` → I/O wait  

**Interpretation**
- `wa > 20–30%` → disk bottleneck ⚠️  
- High `b` → processes waiting on disk  
- High `si/so` → active swapping (bad)  
- High `r` → CPU contention  

---

## System Bottleneck Flow

```
        [High CPU Load]
               ↓
        [RAM Saturation]
               ↓
        [Swap Usage ↑]
               ↓
        [Disk I/O Wait ↑]
               ↓
        [Processes Blocked]
               ↓
        [Pipeline Appears Stalled]
```

**Key Insight:** This represents a system-level bottleneck, not a pipeline error.

---

## Step 3 — Identify heavy pipelines

```bash
ps -ef | grep nextflow
```

**Interpretation**
- Assembly + BLAST → resource intensive  
- BLAST-only → comparatively lighter  

---

## Step 4 — Reduce contention

```bash
kill -9 <PID>
```

**Interpretation**
- Frees CPU, memory, and disk I/O  
- Improvement confirms resource contention  

---

## Step 5 — Confirm recovery

```bash
free -h
vmstat 2 5
```

**See how to interpret `free` and `vmstat` outputs in Step 2.**

**Healthy signs**
- Swap stabilised  
- Reduced I/O wait (`wa`)  
- CPU doing real work (`us` increases)  

---

## Step 6 — Check active jobs

```bash
pgrep -af '^blastn '
```

**Interpretation**
- Displays active BLAST processes  
- Confirms execution is ongoing  

---

## Step 7 — Monitor progress

```bash
find work -name ".exitcode" | wc -l
```

**Interpretation**
- Increasing count → pipeline progressing  
- Static count → tasks still running or waiting  

---

## Step 8 — Check task results

```bash
find work -name ".exitcode" -exec cat {} \;
```

**Interpretation**
- `0` → successful execution  
- `130 / 143` → interrupted tasks (not intrinsic errors)  

---

## Step 9 — Check logs

```bash
tail -f .nextflow.log
```

**Interpretation**
- `Submitted` → tasks launching  
- `ERROR` → actual failure  
- `SIGINT` → manual interruption  

---

## Step 10 — Resume safely

```bash
nextflow run main.nf ... -resume
```

**Interpretation**
- Reuses completed tasks  
- Restarts only incomplete or interrupted tasks  

---

## Key Learning Points

- Slow execution does not necessarily indicate failure  
- Swap usage and disk I/O are major bottlenecks  
- Avoid running multiple resource-intensive pipelines simultaneously  
- Always use `-resume` to recover interrupted workflows  
- A single workstation requires manual resource management  

---

## Recommended Settings

| Scenario | Settings |
|--------|---------|
| BLAST only | `--cpus 6 --max_forks 2` |
| Assembly pipeline | `maxForks = 1` |
| Mixed pipelines | Not recommended |

---

## Teaching Insight

This exercise demonstrates:
- Systems-level thinking in bioinformatics workflows  
- Importance of resource-aware pipeline design  
- Practical debugging beyond code-level errors  
