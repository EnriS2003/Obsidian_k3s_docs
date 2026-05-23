# EXP3_MainNodeFailure – Procedure Used for All Runs

This document summarizes the procedure followed for the `EXP3_MainNodeFailure` experiment, also referred to as the **main node kill** or **control-plane node failure** experiment.

The goal of this document is to make the experiment reproducible in a future chat or future work session.

---

## 1. Experiment Purpose

The goal of `EXP3_MainNodeFailure` is to evaluate what happens when the main k3s node is shut down.

In the current prototype, VM1 has two important roles:

| VM | Hostname | Role | IP |
|---|---|---|---|
| VM1 | `alpinelinux` | k3s server / control-plane / main entrypoint | `192.168.100.10` |
| VM2 | `alpine-node-2` | k3s worker node | `192.168.100.30` |
| VM3 | `rancher.lab.local` | Rancher management server | `192.168.100.20` |

This experiment is different from the worker-node failure experiment because VM1 hosts the Kubernetes control-plane. When VM1 is powered off, the Kubernetes API server is unavailable.

Therefore, during the failure window:

```text
kubectl does not work
SSH to VM1 does not work
scp from VM1 does not work
Kubernetes-side after-failure snapshots cannot be collected
```

This is not a mistake in the experiment. It is one of the main findings: the current prototype does not have a highly available control-plane.

---

## 2. Experiment Naming and Folder Structure

The experiment name used is:

```text
EXP3_MainNodeFailure
```

The runs performed were structured as:

```text
run-01_2026-05-18
run-02_2026-05-18
run-03_2026-05-18
```

### Windows folder structure

For each run, the Windows path was:

```powershell
C:\Users\i0012934\OneDrive - HTI\Documenti\K8s project\3.2-PrototypeExperiments&baseline\2_OfficialExperiments\EXP3_MainNodeFailure\<RUN_NAME>
```

Each run folder contains:

```text
run-notes.md
http-monitor-vm1.ps1
http-monitor-vm2.ps1
downtime-results-vm1.csv
downtime-results-vm2.csv
fileFromVM/
```

The `fileFromVM` folder is used to keep files copied from the VM separate from local Windows files.

### VM folder structure

On VM1, the path used was:

```bash
/root/experiments/EXP3_MainNodeFailure/<RUN_NAME>
```

Important: the VM path does **not** contain an `official` directory.

---

## 3. Why Two HTTP Monitors Were Used

For this experiment, two HTTP monitors were used:

| Monitor | Endpoint | Purpose |
|---|---|---|
| `http-monitor-vm1.ps1` | `http://192.168.100.10/api/data` | Tests the main node / normal entrypoint |
| `http-monitor-vm2.ps1` | `http://192.168.100.30/api/data` | Tests whether the worker node can still serve the application during VM1 failure |

This distinction is important.

If VM1 goes down, `192.168.100.10` is expected to become unreachable. Monitoring `192.168.100.30` helps determine whether any application-level runtime continuity remains on the worker node.

---

## 4. Procedure Template for Each Run

The same procedure was followed for each run, changing only the run name.

Set the run variable as needed:

```powershell
$run = "run-01_2026-05-18"
```

or:

```powershell
$run = "run-02_2026-05-18"
```

or:

```powershell
$run = "run-03_2026-05-18"
```

---

## Step 1 – Create the Windows Run Folder

PowerShell:

```powershell
$root = "C:\Users\i0012934\OneDrive - HTI\Documenti\K8s project\3.2-PrototypeExperiments&baseline"
$exp = "EXP3_MainNodeFailure"
$run = "run-01_2026-05-18"

New-Item -ItemType Directory -Force -Path "$root\2_OfficialExperiments\$exp\$run"
New-Item -ItemType Directory -Force -Path "$root\2_OfficialExperiments\$exp\$run\fileFromVM"

Copy-Item "$root\1_Docs\TemplateForExperiments_FINAL.md" "$root\2_OfficialExperiments\$exp\$run\run-notes.md" -Force

cd "$root\2_OfficialExperiments\$exp\$run"
```

For run 02 or run 03, only change the `$run` value.

---

## Step 2 – Create the VM1 HTTP Monitor

This monitor checks the main node endpoint:

```text
http://192.168.100.10/api/data
```

PowerShell, inside the run folder:

```powershell
@'
# HTTP availability monitoring script - VM1 endpoint

$Url = "http://192.168.100.10/api/data"
$OutputFile = "downtime-results-vm1.csv"
$IntervalMilliseconds = 500
$TimeoutSeconds = 1

"timestamp,url,status_code,success,latency_seconds,error" | Out-File -FilePath $OutputFile -Encoding utf8

Write-Host "Monitoring started"
Write-Host "URL: $Url"
Write-Host "Output: $OutputFile"
Write-Host "Press CTRL+C to stop"

while ($true) {
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss.fff"
    $stopwatch = [System.Diagnostics.Stopwatch]::StartNew()

    try {
        $response = Invoke-WebRequest -Uri $Url -TimeoutSec $TimeoutSeconds -UseBasicParsing
        $stopwatch.Stop()

        $statusCode = [int]$response.StatusCode
        $latency = [Math]::Round($stopwatch.Elapsed.TotalSeconds, 6)

        if ($statusCode -ge 200 -and $statusCode -lt 400) {
            $success = "true"
            $errorMessage = ""
        } else {
            $success = "false"
            $errorMessage = "HTTP_$statusCode"
        }
    }
    catch {
        $stopwatch.Stop()

        $statusCode = "000"
        $success = "false"
        $latency = [Math]::Round($stopwatch.Elapsed.TotalSeconds, 6)
        $errorMessage = $_.Exception.Message.Replace(",", ";")
    }

    "$timestamp,$Url,$statusCode,$success,$latency,""$errorMessage""" | Out-File -FilePath $OutputFile -Append -Encoding utf8
    Write-Host "$timestamp -> $statusCode | success=$success | latency=${latency}s"

    Start-Sleep -Milliseconds $IntervalMilliseconds
}
'@ | Set-Content .\http-monitor-vm1.ps1
```

---

## Step 3 – Create the VM2 HTTP Monitor

This monitor checks the worker node endpoint:

```text
http://192.168.100.30/api/data
```

PowerShell, inside the run folder:

```powershell
@'
# HTTP availability monitoring script - VM2 endpoint

$Url = "http://192.168.100.30/api/data"
$OutputFile = "downtime-results-vm2.csv"
$IntervalMilliseconds = 500
$TimeoutSeconds = 1

"timestamp,url,status_code,success,latency_seconds,error" | Out-File -FilePath $OutputFile -Encoding utf8

Write-Host "Monitoring started"
Write-Host "URL: $Url"
Write-Host "Output: $OutputFile"
Write-Host "Press CTRL+C to stop"

while ($true) {
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss.fff"
    $stopwatch = [System.Diagnostics.Stopwatch]::StartNew()

    try {
        $response = Invoke-WebRequest -Uri $Url -TimeoutSec $TimeoutSeconds -UseBasicParsing
        $stopwatch.Stop()

        $statusCode = [int]$response.StatusCode
        $latency = [Math]::Round($stopwatch.Elapsed.TotalSeconds, 6)

        if ($statusCode -ge 200 -and $statusCode -lt 400) {
            $success = "true"
            $errorMessage = ""
        } else {
            $success = "false"
            $errorMessage = "HTTP_$statusCode"
        }
    }
    catch {
        $stopwatch.Stop()

        $statusCode = "000"
        $success = "false"
        $latency = [Math]::Round($stopwatch.Elapsed.TotalSeconds, 6)
        $errorMessage = $_.Exception.Message.Replace(",", ";")
    }

    "$timestamp,$Url,$statusCode,$success,$latency,""$errorMessage""" | Out-File -FilePath $OutputFile -Append -Encoding utf8
    Write-Host "$timestamp -> $statusCode | success=$success | latency=${latency}s"

    Start-Sleep -Milliseconds $IntervalMilliseconds
}
'@ | Set-Content .\http-monitor-vm2.ps1
```

---

## Step 4 – Create the Run Folder on VM1

On VM1:

```bash
mkdir -p /root/experiments/EXP3_MainNodeFailure/run-01_2026-05-18
cd /root/experiments/EXP3_MainNodeFailure/run-01_2026-05-18
```

For run 02:

```bash
mkdir -p /root/experiments/EXP3_MainNodeFailure/run-02_2026-05-18
cd /root/experiments/EXP3_MainNodeFailure/run-02_2026-05-18
```

For run 03:

```bash
mkdir -p /root/experiments/EXP3_MainNodeFailure/run-03_2026-05-18
cd /root/experiments/EXP3_MainNodeFailure/run-03_2026-05-18
```

---

## Step 5 – Collect BEFORE Snapshots

On VM1, inside the run folder:

```bash
kubectl get nodes -o wide > nodes-before.txt
kubectl get deployments -n prototype > deployments-before.txt
kubectl get pods -n prototype -o wide > pods-before.txt
kubectl get services -n prototype > services-before.txt
kubectl get ingress -n prototype > ingress-before.txt
kubectl get events -n prototype --sort-by=.lastTimestamp > events-before.txt
kubectl get events -A --sort-by=.lastTimestamp > events-all-before.txt
```

Also check the baseline state on screen:

```bash
kubectl get nodes -o wide
kubectl get deployments -n prototype
kubectl get pods -n prototype -o wide
```

Expected general state before the failure:

```text
VM1 / alpinelinux      Ready
VM2 / alpine-node-2    Ready

web-ui            2/2
api-backend       2/2
logic-simulator   1/1
data-generator    1/1
```

---

## Step 6 – Start Both HTTP Monitors

Open two separate PowerShell terminals.

In both terminals, go to the run folder:

```powershell
cd "C:\Users\i0012934\OneDrive - HTI\Documenti\K8s project\3.2-PrototypeExperiments&baseline\2_OfficialExperiments\EXP3_MainNodeFailure\run-01_2026-05-18"
```

For run 02 or run 03, change the run folder accordingly.

In the first PowerShell:

```powershell
.\http-monitor-vm1.ps1
```

In the second PowerShell:

```powershell
.\http-monitor-vm2.ps1
```

Wait around 20–30 seconds before injecting the fault.

Expected observation:

- VM1 monitor should normally return HTTP `200`.
- VM2 monitor may or may not return HTTP `200`, depending on whether the application can be reached directly through the worker node.

If VM2 does not respond before the fault, this should be recorded in `run-notes.md`.

---

## Step 7 – Inject the Fault: Power Off VM1

Open a third PowerShell terminal.

Check the VM name:

```powershell
Get-VM
```

Then power off VM1:

```powershell
Get-Date -Format "yyyy-MM-dd HH:mm:ss.fff"
Stop-VM -Name "alpinelinux" -TurnOff
```

The timestamp printed by `Get-Date` should be recorded as:

```text
Fault Injection Time
```

In practice, the first HTTP `000` in the monitor can also be used as the operational client-side fault timestamp, if it coincides with the manual shutdown.

---

## Step 8 – During VM1 Failure

During VM1 failure, do not try to collect Kubernetes-side after-failure snapshots.

The reason is:

```text
VM1 hosts the k3s control-plane and API server.
When VM1 is powered off, kubectl cannot communicate with the cluster.
```

During this phase:

```text
kubectl does not work
SSH to VM1 does not work
scp from VM1 does not work
Kubernetes after-failure snapshots cannot be collected
```

Data available during the failure window:

```text
downtime-results-vm1.csv
downtime-results-vm2.csv
manual observation
```

Let both HTTP monitors run for a few minutes.

---

## Step 9 – Power VM1 Back On

From PowerShell:

```powershell
Start-VM -Name "alpinelinux"
```

Check when SSH becomes available again:

```powershell
Test-NetConnection 192.168.100.10 -Port 22
```

Wait until:

```text
TcpTestSucceeded : True
```

Then SSH back into VM1.

---

## Step 10 – Collect AFTER-RETURN Snapshots

On VM1, go back into the run folder:

```bash
cd /root/experiments/EXP3_MainNodeFailure/run-01_2026-05-18
```

For run 02 or run 03, use the corresponding folder.

Wait until `kubectl` works again:

```bash
kubectl get nodes -o wide
```

Then collect after-return snapshots:

```bash
kubectl get nodes -o wide > nodes-after-node-return.txt
kubectl get deployments -n prototype > deployments-after-node-return.txt
kubectl get pods -n prototype -o wide > pods-after-node-return.txt
kubectl get services -n prototype > services-after-node-return.txt
kubectl get ingress -n prototype > ingress-after-node-return.txt
kubectl get events -n prototype --sort-by=.lastTimestamp > events-prototype-after-node-return.txt
kubectl get events -A --sort-by=.lastTimestamp > events-all-after-node-return.txt
```

---

## Step 11 – Stop Both HTTP Monitors

Return to the two PowerShell windows running the monitors and stop each one with:

```text
CTRL+C
```

The Windows run folder should now contain:

```text
downtime-results-vm1.csv
downtime-results-vm2.csv
```

---

## Step 12 – Copy VM Files to Windows

PowerShell:

```powershell
$root = "C:\Users\i0012934\OneDrive - HTI\Documenti\K8s project\3.2-PrototypeExperiments&baseline"
$exp = "EXP3_MainNodeFailure"
$run = "run-01_2026-05-18"
$dest = "$root\2_OfficialExperiments\$exp\$run\fileFromVM"

scp -r root@192.168.100.10:/root/experiments/EXP3_MainNodeFailure/run-01_2026-05-18/. "$dest"
```

For run 02:

```powershell
$run = "run-02_2026-05-18"
$dest = "$root\2_OfficialExperiments\$exp\$run\fileFromVM"

scp -r root@192.168.100.10:/root/experiments/EXP3_MainNodeFailure/run-02_2026-05-18/. "$dest"
```

For run 03:

```powershell
$run = "run-03_2026-05-18"
$dest = "$root\2_OfficialExperiments\$exp\$run\fileFromVM"

scp -r root@192.168.100.10:/root/experiments/EXP3_MainNodeFailure/run-03_2026-05-18/. "$dest"
```

Check copied files:

```powershell
Get-ChildItem "$dest"
```

Expected files copied from VM:

```text
nodes-before.txt
deployments-before.txt
pods-before.txt
services-before.txt
ingress-before.txt
events-before.txt
events-all-before.txt
nodes-after-node-return.txt
deployments-after-node-return.txt
pods-after-node-return.txt
services-after-node-return.txt
ingress-after-node-return.txt
events-prototype-after-node-return.txt
events-all-after-node-return.txt
```

---

## 5. Quick CSV Analysis Commands

### Analyze VM1 monitor

PowerShell, inside the run folder:

```powershell
$csv1 = Import-Csv .\downtime-results-vm1.csv
$total1 = $csv1.Count
$success1 = ($csv1 | Where-Object { $_.success -eq "true" }).Count
$failed1 = ($csv1 | Where-Object { $_.success -eq "false" }).Count

"VM1 total: $total1"
"VM1 successful: $success1"
"VM1 failed: $failed1"

$failures1 = $csv1 | Where-Object { $_.success -eq "false" }

"VM1 first failure:"
$failures1 | Select-Object -First 1

"VM1 last failure:"
$failures1 | Select-Object -Last 1
```

### Analyze VM2 monitor

```powershell
$csv2 = Import-Csv .\downtime-results-vm2.csv
$total2 = $csv2.Count
$success2 = ($csv2 | Where-Object { $_.success -eq "true" }).Count
$failed2 = ($csv2 | Where-Object { $_.success -eq "false" }).Count

"VM2 total: $total2"
"VM2 successful: $success2"
"VM2 failed: $failed2"

$failures2 = $csv2 | Where-Object { $_.success -eq "false" }

"VM2 first failure:"
$failures2 | Select-Object -First 1

"VM2 last failure:"
$failures2 | Select-Object -Last 1
```

---

## 6. Notes to Add in Each Run

Each run note should include this explanation:

```text
During VM1 failure, Kubernetes-side after-failure data could not be collected because VM1 hosts the k3s control-plane and API server. Therefore, only external HTTP monitoring was available during the failure window. Kubernetes snapshots were collected before the fault and after VM1 returned.
```

Also include:

```text
This experiment is not only an application availability test. It also tests the availability of the current k3s control-plane architecture. Since the cluster has only one server/control-plane node, VM1 is a single point of failure for Kubernetes API availability and cluster management.
```

---

## 7. Expected Interpretation

The expected classification for this experiment is stronger than for the worker-node failure experiment.

For worker-node failure, k3s can still be managed because the control-plane remains alive.

For main-node failure:

```text
VM1 down means the control-plane is down.
```

Expected classification:

```text
Not fully handled by current architecture
```

or:

```text
Not handled by k3s alone in the current single-server setup
```

Reason:

```text
The current prototype has no highly available control-plane.
```

The correct architectural solution would require:

```text
- multiple k3s server/control-plane nodes;
- embedded etcd with at least three server nodes, or an external highly available datastore;
- a stable API endpoint / virtual IP / load balancer;
- a stable application entrypoint independent from a single node IP.
```

---

## 8. Main Finding Template

A useful finding for the final analysis:

```text
The main-node failure experiment shows that the current prototype has a single point of failure at VM1. When VM1 is powered off, the Kubernetes API server becomes unavailable, so Kubernetes-side after-failure data cannot be collected and the cluster cannot be managed until VM1 returns. The primary application entrypoint on 192.168.100.10 also becomes unavailable. This failure mode cannot be fully solved at workload level; it requires a high-availability control-plane and a stable application/API endpoint.
```

---

## 9. Difference from EXP1 Worker Node Failure

| Aspect | EXP1 Worker Node Failure | EXP3 Main Node Failure |
|---|---|---|
| Failed node | VM2 / worker node | VM1 / main/control-plane node |
| Kubernetes API during failure | Available | Unavailable |
| `kubectl` during failure | Works | Does not work |
| After-failure Kubernetes snapshot | Available | Not available |
| Application entrypoint | Usually still VM1 | VM1 itself goes down |
| Main issue | Placement, endpoint update, partial degradation | Control-plane and entrypoint single point of failure |
| Solution direction | Placement rules, replicas, ingress behaviour | HA control-plane, VIP/load balancer, stable endpoint |

---

## 10. Procedure Summary

For each EXP3 run:

```text
1. Create Windows run folder.
2. Create VM run folder.
3. Create two HTTP monitors: VM1 endpoint and VM2 endpoint.
4. Save Kubernetes BEFORE snapshots.
5. Start both HTTP monitors.
6. Power off VM1.
7. Observe only via external HTTP monitoring while VM1 is down.
8. Power VM1 back on.
9. Wait until SSH and kubectl work again.
10. Save Kubernetes AFTER-RETURN snapshots.
11. Stop HTTP monitors.
12. Copy VM files into Windows fileFromVM.
13. Analyze downtime-results-vm1.csv and downtime-results-vm2.csv.
```
