<h1 align="center">Why Your Pipeline Is Slow or Stalled: Troubleshooting CPU, Memory, and I/O Bottlenecks on a Single Workstation</h1>

<h3 align="center">M. Asaduzzaman Prodhan<sup>*</sup> </h3>

<div align="center"><b> School of Biological Sciences, The University of Western Australia </b></div>

<div align="center"><b> 35 Stirling Highway, Perth, WA 6009, Australia. <sup>*</sup>Correspondence: prodhan82@gmail.com </b></div>

<br />

<p align="center">
  <a href="https://github.com/asadprodhan/Nextflow_Troubleshooting_Microcredential/tree/main#GPL-3.0-1-ov-file"><img src="https://img.shields.io/badge/License-GPL%203.0-yellow.svg" alt="License GPL 3.0" style="display: inline-block;"></a>
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

- Understand why bioinformatics pipelines become slow or stalled, and
- apply practical strategies to diagnose and resolve CPU, memory, and disk I/O bottlenecks on a single workstation

---

## Step 1 — Check pipeline status

```bash
ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head
```

**Scenario 1 — Active execution (healthy)**


| PID   | COMMAND           | %CPU | %MEM |
|------|------------------|------|------|
| 70562 | blast_formatter | 26.8 | 49.5 |
| 70591 | blast_formatter | 13.4 | 17.3 |
| 69366 | java            | 0.4  | 0.0  |


**Interpretation**
- blast_formatter → two `blast_formatter` processes are running → the workflow is running
- High CPU (26.8%, 13.4%) → `blast_formatter` is actively computing → tasks are actively progressing 
- Very high memory (49.5%, 17.3%) → `blast_formatter` is memory-intensive process
- java → Nextflow controller (low CPU, expected) 


**Scenario 2 — Idle or finished**


| PID   | COMMAND       | %CPU | %MEM |
|------|--------------|------|------|
| 69366 | java         | 0.3  | 0.1  |
| 2705  | gnome-shell  | 0.1  | 0.0  |


**Interpretation**

- only background/system processes
- no workflow-related processes consuming CPU
- pipeline is not running or may be idle, stuck, or finished

**Scenario 3 — Resource bottleneck**


| PID   | COMMAND | %CPU | %MEM |
|------|--------|------|------|
| 70562 | blastn | 0.5  | 10.2 |
| 70591 | blastn | 0.3  | 10.1 |


**Interpretation**

- workflow processes exist but CPU usage is very low
- tasks are waiting (likely disk I/O or memory issue)
- pipeline is slowed by resource bottlenecks

---

## Step 2 — Diagnose system resources

**System Load**

```
uptime | awk -F'load average: ' '{print "load average: " $2}'
```

**Output**

load average: 2.06, 2.11, 2.80


**Interpretation**

- `load average` shows system load over 1, 5, and 15 minutes
- Compare these values with the number of CPU cores (nproc)
- Load ≤ CPU cores → system is healthy
- Load > CPU cores → system is overloaded


**CPU Cores**

```bash
nproc
```

**Output**

20

**Interpretation**

- number of available CPU cores
- used as a reference to compare with load average from uptime


**Memory Usage**

```bash
free -h
```

**Output**

| Type | Total | Used  | Free  | Shared | Buff/Cache | Available |
|------|------|------|------|--------|-----------|-----------|
| Mem  | 125Gi | 1.8Gi | 68Gi | 9.1Mi  | 56Gi      | 123Gi     |
| Swap | 8.0Gi | 662Mi | 7.4Gi | —      | —         | —         |


**Interpretation**

- `available` → actual usable memory for new processes  
  - 123 Gi available out of 125 Gi total → almost all memory is available  
  - High available memory → system has sufficient RAM  

- `buff/cache` → filesystem cache used by the OS to speed up I/O  
  - 56 Gi used as cache → normal behaviour  
  - Compared with 123 Gi available → cache is not limiting memory  
  - High `buff/cache` is automatically reclaimed when needed  

- `Swap` → overflow memory used when RAM is insufficient  
  - 662 Mi used out of 8.0 Gi total (~8%)  
  - Low usage relative to total → no memory pressure


**System Activity (CPU, Memory, I/O)**

```bash
vmstat 2 5
```

| r | b | swpd  | free     | buff | cache    | si  | so  | bi    | bo    | in  | cs  | us | sy | id  | wa | st | gu |
|---|---|-------|----------|------|----------|-----|-----|-------|-------|-----|-----|----|----|-----|----|----|----|
| 1 | 0 | 678808 | 71702520 | 5284 | 59408296 | 775 | 1140 | 28318 | 10387 | 6616 | 1   | 13 | 0  | 82  | 6  | 0  | 0  |
| 0 | 0 | 678808 | 71702520 | 5284 | 59408356 | 0   | 0   | 0     | 0     | 128 | 131 | 0  | 0  | 100 | 0  | 0  | 0  |
| 0 | 0 | 678808 | 71702520 | 5284 | 59408356 | 0   | 0   | 0     | 0     | 143 | 140 | 0  | 0  | 100 | 0  | 0  | 0  |
| 0 | 0 | 678808 | 71702520 | 5284 | 59408356 | 0   | 0   | 0     | 0     | 118 | 119 | 0  | 0  | 100 | 0  | 0  | 0  |
| 0 | 0 | 678808 | 71702520 | 5284 | 59408356 | 0   | 0   | 0     | 0     | 139 | 151 | 0  | 0  | 100 | 0  | 0  | 0  |


**Interpretation**

- `r` (running processes) → 1 initially, then 0  
  - Very low CPU demand → no contention  

- `b` (blocked processes) → consistently 0  
  - No processes waiting on I/O → no disk bottleneck  

- `swpd` → 678,808 KB (~0.65 Gi)  
  - Very low swap usage → no memory pressure  

- `si/so` (swap in/out) → initially 775 / 1140, then 0  
  - No ongoing swapping → memory is stable  

- `bi/bo` (disk I/O) → initially 28,318 / 10,387, then 0  
  - Brief disk activity followed by idle → no sustained I/O load  

- `us` (CPU user) → initially 13%, then 0%  
- `id` (CPU idle) → 82–100%  
  - CPU mostly idle → system not under load  

- `wa` (I/O wait) → 6%, then 0%  
  - Very low → no I/O bottleneck  

- System is idle and healthy: no CPU contention, no memory pressure, and no disk I/O bottleneck

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
Run the above code over time. Say the outputs are follows:

> 15 19 21

**What this means**

- Each .exitcode file represents a completed task
- Increasing count → pipeline progressing. Even if the pipeline appears slow, an increasing .exitcode count is a reliable indicator of ongoing execution  
- Static count → tasks still running or waiting  

---

## Step 8 — Check task results

```bash
find work -name ".exitcode" -exec sh -c 'for f; do printf "%s\n" "$(cat "$f")"; done' sh {} \; | sort | uniq -c
```

**Example output**

10 0

9 130

2 143

**Interpretation**
- 11 tasks → exit code 0. These completed successfully. No issues with BLAST or pipeline logic   
- 8 tasks → exit code 130. Typically caused by SIGINT (e.g. Ctrl+C or workflow interruption)
- 2 tasks → exit code 143. Typically caused by SIGTERM (e.g. process killed or system shutdown)

**What this means overall**
- Total tasks: 21
- ~52% completed successfully
- ~48% interrupted externally

**This strongly indicates:**

- The pipeline itself is working correctly
- The failures are not due to errors in the pipeline/s
- Tasks were interrupted due to external events (e.g. killing process, resource overload, SIGINT)

---

## Step 9 — Check logs

An example for checking Nextflow pipeline:

```bash
tail -f .nextflow.log
```

**Interpretation of TaskHandler fields**

- `TaskHandler` → Internal Nextflow object representing a single task (process execution unit)  
- `id` → Unique identifier for the task within the workflow  
- `name` → Process name and input label (e.g. `blastn (sample1)`)  
- `status` → Current state (`NEW`, `RUNNING`, `COMPLETED`, etc.)  
- `exit` → Exit code after completion (`0` = success; non-zero = failure/interruption; `-` = empty- the task is currently pending or running)  
- `error` → Error message if the task failed (empty if none)  
- `workDir` → Directory containing task scripts, logs, and outputs  

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
