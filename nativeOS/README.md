**HA Proof-of-Concept for IBM MQ** 

* **Track A (recommended POC): Multi-Instance Queue Manager (MIQM)** on two lightweight Rocky VMs sharing an NFS volume (fast to set up; excellent learning value).
* **Track B (stretch goal): Native HA in Kubernetes** with a 3-node local cluster (closer to modern prod patterns; heavier lift).

I’ll lay out architecture, exact commands, health checks, and a failover runbook. No hand-waving.

---

# Track A — Multi-Instance Queue Manager (Two VMs + Shared NFS)

## 0) What you’ll get

* One queue manager `POCQM` with **one active** and **one warm standby**.
* Shared message/log/data on NFSv4 with proper file locking.
* Forced failover test with **zero data loss** (locks ensure single writer).
* Repeatable, idempotent scripts and checks.

> Note: This is “intra-laptop HA.” It won’t survive a full-laptop outage, but it’s perfect to learn HA mechanics you’ll later move to servers/cloud.

---

## 1) Topology

* **Host (your laptop)**: Rocky Linux, runs the NFS server (shared storage).
* **VM A** `mq-a` (Rocky, 2 vCPU, 2–4 GB RAM): MQ installed
* **VM B** `mq-b` (Rocky, 2 vCPU, 2–4 GB RAM): MQ installed
* **Network**: host-only / bridged; ensure all three can resolve each other

Suggested IPs:

```
Host:  192.168.56.1   (NFS server)
mq-a:  192.168.56.10
mq-b:  192.168.56.11
```

---

## 2) Host: NFS server setup (shared storage with locks)

```bash
# On the HOST (Rocky Linux)
sudo dnf install -y nfs-utils
sudo mkdir -p /srv/mqmshared/POCQM/{data,log}
sudo chown -R 1000:1000 /srv/mqmshared  # temp; will fix to mqm uid/gid later

# Export over NFSv4 (with locking)
echo '/srv/mqmshared 192.168.56.0/24(rw,sync,no_root_squash,fsid=0,crossmnt)' | sudo tee /etc/exports.d/mqm.exports

sudo systemctl enable --now nfs-server
sudo exportfs -rv
```

> **Locking**: NFSv4 provides integrated locking (no separate lockd config needed). Keep v4+ and avoid `nolock`.

---

## 3) Create two Rocky VMs and prep OS

On both `mq-a` and `mq-b`:

```bash
sudo dnf update -y
sudo dnf install -y nfs-utils chrony ksh tar unzip libaio
sudo systemctl enable --now chronyd     # clock sync matters for HA

# Hostname & /etc/hosts for clean name resolution (adjust IPs as used)
echo mq-a | sudo tee /etc/hostname   # on mq-a
# on mq-b do: echo mq-b | sudo tee /etc/hostname

# /etc/hosts on BOTH VMs:
sudo bash -lc 'cat >>/etc/hosts <<EOF
192.168.56.1   nfs-host
192.168.56.10  mq-a
192.168.56.11  mq-b
EOF'
```

---

## 4) Mount the shared filesystem on both VMs

```bash
# On BOTH mq-a and mq-b:
sudo mkdir -p /mnt/mqmshared/POCQM/{data,log}
echo 'nfs-host:/srv/mqmshared  /mnt/mqmshared  nfs4  _netdev,hard,intr,nocto  0 0' | sudo tee -a /etc/fstab
sudo mount -a
```

Validate:

```bash
touch /mnt/mqmshared/testfile
# On the other VM, confirm the file appears.
```

---

## 5) Install IBM MQ Advanced for Developers (both VMs)

> Use your IBM MQ Developer tarball (9.2+/9.3+). Below is the classic local install flow.

```bash
# Copy MQServer tar.gz to /tmp then:
cd /tmp
tar -xzf IBM_MQ_*.tar.gz
cd MQServer
sudo ./mqlicense.sh -accept

# Minimal packages for a server queue manager:
sudo dnf install -y *.rpm || sudo rpm -Uvh *.rpm

# Register install (if needed on your version)
sudo /opt/mqm/bin/setmqinst -i -p /opt/mqm

# Confirm
dspmqver
```

Create `mqm` if not created by RPMs:

```bash
# Usually created by MQ install. If not:
sudo groupadd mqm || true
sudo useradd -g mqm -m mqm || true
```

Give **mqm** ownership on the mounted NFS paths:

```bash
sudo chown -R mqm:mqm /mnt/mqmshared
```

---

## 6) Create the multi-instance queue manager

> Do this ONCE (on `mq-a`). The queue manager’s **data and logs** must be on the shared NFS path.

```bash
# As mqm user on mq-a
sudo -iu mqm
crtmqm -md /mnt/mqmshared/POCQM/data -ld /mnt/mqmshared/POCQM/log POCQM
strmqm POCQM
```

Harden basic MQSC config (listener, channels, queues). Still on `mq-a`:

```bash
runmqsc POCQM <<'EOF'
DEFINE LISTENER(L1) TRPTYPE(TCP) PORT(1414) CONTROL(QMGR) REPLACE
START LISTENER(L1)
DEFINE CHANNEL(DEV.APP.SVRCONN) CHLTYPE(SVRCONN) MCAUSER('mqm') REPLACE
DEFINE QLOCAL(Q1) DEFPSIST(YES) REPLACE
ALTER QMGR CHLAUTH(DISABLED) CONNAUTH(' ')  /* for POC only; NOT for prod */
REFRESH SECURITY TYPE(CONNAUTH)
EOF
```

**Stand up standby** on `mq-b`:

```bash
# As mqm user on mq-b
sudo -iu mqm
# No need to crtmqm; the data is shared. Just start as standby:
strmqm -x POCQM   # -x = standby
```

Check statuses:

```bash
# On both VMs
dspmq -x -o status
# Expect one "Running" (active) and one "Running as standby".
```

---

## 7) Prove it works (put/get + forced failover)

On **mq-a** (active), put 50 messages then start a consumer that runs indefinitely:

```bash
# Put:
for i in $(seq 1 50); do echo "msg-$i" | amqsput Q1 POCQM; done

# In a second terminal on mq-a, start get (it will block after queue drains):
amqsget Q1 POCQM
```

Now **force a failover**:

```bash
# On mq-a (active):
endmqm -i POCQM    # immediate stop (simulate crash without data loss)

# On mq-b (standby) within seconds becomes active. Verify:
dspmq -x -o status
```

Re-run a consumer on **mq-b**:

```bash
amqsget Q1 POCQM
# You should see remaining or new messages; the queue manager is now active on mq-b.
```

Optionally flip back by starting standby on `mq-a`:

```bash
# mq-a
strmqm -x POCQM
# mq-b
endmqm -i POCQM
# mq-a will take over as active.
```

---

## 8) Operational hardening (POC-level)

* **Health checks**:

  ```bash
  # Active/standby view
  watch -n 2 "dspmq -x -o status"
  ```
* **Lock sanity**: Never mount NFS with `nolock`. Keep `nfs4` and defaults above.
* **Permissions**: Entire shared path owned by `mqm:mqm`. No root-squash problems thanks to `no_root_squash` and running as `mqm`.
* **Time sync**: `chronyd` running everywhere.
* **Security (POC-only warnings)**: You disabled CHLAUTH and CONNAUTH above. For anything beyond a lab demo, re-enable and secure SVRCONN with TLS + CHLAUTH rules.

---

# Track B — Native HA in Kubernetes (three replicas)

If you want something closer to cloud-native production:

**High-level steps** (short version—you can ask me to expand and I’ll drop the manifests):

1. **Create a 3-node local cluster**

   * Use `kind` with a multi-node config (1 control plane + 3 workers), or `k3d` with 3 agents.
2. **Storage**

   * Use `local-path-provisioner` (dev) or a lightweight CSI that supports `ReadWriteOnce` volumes per replica.
3. **Deploy IBM MQ container(s)**

   * Use the **IBM MQ Advanced for Developers** image.
   * If you have access to the IBM MQ Operator + Native HA CRDs, deploy a **NativeHA** queue manager with `replicas: 3`.
4. **Service & Ingress**

   * Expose 1414 (MQ) and 9443 (web console) via NodePort/Ingress.
5. **Failover test**

   * Kill a pod and watch the raft leader re-elect; verify message continuity.

> This route more faithfully represents modern MQ HA in containers, but it requires container images and (optionally) the MQ Operator. It’s fantastic once you’re comfortable with Track A fundamentals.

---

## Extras you’ll probably want

### A. Minimal MQSC bundle (import any time)

```mqsc
* POC base
DEFINE LISTENER(L1) TRPTYPE(TCP) PORT(1414) CONTROL(QMGR) REPLACE
START LISTENER(L1)
DEFINE CHANNEL(DEV.APP.SVRCONN) CHLTYPE(SVRCONN) MCAUSER('mqm') REPLACE
DEFINE QLOCAL(Q1) DEFPSIST(YES) REPLACE
ALTER QMGR CHLAUTH(DISABLED) CONNAUTH(' ')
REFRESH SECURITY TYPE(CONNAUTH)
```

### B. Simple put/get smoke test (one-liner)

```bash
# Put 5 and read 5:
for i in {1..5}; do echo "hello-$i" | amqsput Q1 POCQM; done && amqsget Q1 POCQM
```

### C. Quick “observer” for failover timing

```bash
# On standby VM, log status every second:
while true; do date '+%T'; dspmq -x -o status; sleep 1; done
```

### D. Clean teardown

```bash
# Stop QMs (on both VMs)
endmqm -i POCQM || true

# Unmount (on both VMs)
sudo umount /mnt/mqmshared

# On host
sudo exportfs -u /srv/mqmshared
sudo systemctl stop nfs-server
```

---

## Why this is robust (even as a laptop POC)

* **Single-writer guarantee** via NFSv4 locks → no split-brain.
* **Warm standby** means nearly instant takeover (file handle transfer + restart).
* **Deterministic recovery**: MQ journal + persistent messages guarantee no loss when using default persistent settings.


If you want, I can package this into a **scripted installer** (creates VMs with `virt-install`, configures NFS, installs MQ, builds the QM, and runs the failover demo) or convert Track A into a **Vagrant** or **Ansible** playbook so you can rinse/repeat.
