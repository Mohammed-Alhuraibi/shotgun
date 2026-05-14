# Running jianhong/shotgun on Kubernetes with a NAS-backed share

This directory contains everything needed to run **jianhong/shotgun** on a
Kubernetes cluster where the input data and pipeline working directory live
on a **CIFS/SMB NAS** rather than on a `StorageClass`. It is intended
for labs that have a shared file server and want their pods to read inputs
from / write outputs to that server directly — no MinIO, no S3 bucket, no
managed CSI driver beyond `smb.csi.k8s.io`.

The manifests target any SMB v3.0+ capable NAS (Synology, TrueNAS, ASUS,
generic Samba) and any Kubernetes cluster running the `smb.csi.k8s.io` CSI
driver. Only the `volumeAttributes.source` string in `nas-pv-pvc.yaml`
needs to change per environment.

---

## Architecture at a glance

```
   +--------------------+        +-----------------+        +------------------------+
   |  Your workstation  |        | Kubernetes      |        |  NAS server            |
   |                    |        | cluster         |        |  (SMB/CIFS)            |
   |  ~/nf-runs/<run>   |        |                 |        |                        |
   |   (launchDir,      |        | task pods       |        |  /<share>/             |
   |    .nextflow/,     |        |  mount the PVC  |        |    <subpath>/          |
   |    plugins)        |        |  at /mnt/nas    |        |     pipelines/         |
   |                    |        |        |        |        |       shotgun/         |
   |  /mnt/nas (CIFS)   |<------>|        v        |<------>|     projects/          |
   |  (same view as     |  SMB   |   nextflow-sa   |  SMB   |       <project>/       |
   |   the pods)        |        |   (RBAC)        |        |         inputs/        |
   |                    |        |                 |        |         results/       |
   |                    |        |                 |        |     work/              |
   +--------------------+        +-----------------+        +------------------------+
            ^                              ^
            |                              |
            +---- nextflow run             +---- creates/watches/deletes pods
                  (driver lives here)            in the `nextflow` namespace
```

Three rules make this layout work:

1. **`launchDir` lives on local disk**, not on CIFS. Nextflow extracts its
   plugins into `<launchDir>/.nextflow/` using symlinks and POSIX
   permission bits; CIFS supports both only when the mount has
   `mfsymlinks` and `noperm`. The launchDir holds a few megabytes of
   Nextflow state, so keeping it on local disk is faster, decouples runs
   from NAS availability, and avoids CIFS dialect compatibility issues.
   Without `mfsymlinks`, plugin extraction fails with `Operation not
   supported`.
2. **`projectDir`, `workDir`, `--input`, `--outdir`** all live on the NAS.
   They are reachable from both the workstation (via CIFS) and the task
   pods (via the PVC) at **the same absolute path**.
3. **Host and pod views of `/mnt/nas` must point at the same physical
   directory.** The host mount source and the PV's `volumeAttributes.source`
   must use the same `//<host>/<share>/<subpath>` so that
   `/mnt/nas/projects/foo/bar` resolves to the same file on both sides.

---

## Prerequisites

| Component | Notes |
|---|---|
| Kubernetes cluster | Single-node or multi-node, version 1.24+ recommended |
| `kubectl` | Configured with admin access to the cluster |
| `smb.csi.k8s.io` driver installed | See <https://github.com/kubernetes-csi/csi-driver-smb> |
| Nextflow on your workstation | Version `>=21.10.3` (this pipeline's minimum) |
| NAS server | Any SMB v3.0+ capable server, reachable from cluster nodes AND workstation |
| `cifs-utils` on your workstation | `sudo apt install cifs-utils` (Ubuntu/Debian) |

Verify the SMB CSI driver is installed and healthy:

```bash
kubectl get pods -n kube-system -l app=csi-smb-controller
kubectl get csidrivers smb.csi.k8s.io
```

---

## End-to-end setup

Steps 1–4 are one-time cluster-side setup performed by an administrator.
Steps 5–8 are one-time per workstation / per cluster. Steps 7, 9 and 10 are
repeated per project.

### Step 1 — Apply namespace, ServiceAccount and RBAC

```bash
kubectl apply -f k8s/ns-sa-rbac.yaml
```

This creates:

- Namespace `nextflow`
- ServiceAccount `nextflow-sa` (the identity Nextflow's k8s executor uses)
- Role `nextflow-pod-manager` granting just enough permissions to manage
  task pods, pod logs, PVCs and jobs inside the `nextflow` namespace

Verify:

```bash
kubectl -n nextflow get sa
kubectl -n nextflow get role,rolebinding
```

### Step 2 — Prepare the share on the NAS

On the NAS:

1. Create an SMB-exported share (e.g. `data`) and a top-level subdirectory
   inside it for this work (e.g. `nextflow`). The combination forms the path
   you'll plug into `volumeAttributes.source` later:
   `//<NAS-IP-OR-HOST>/<SHARE>/<SUBPATH>`.
2. Create or identify an SMB user with read/write access to that subpath.
3. **Disable the Recycle Bin for this share** if your NAS has that feature
   enabled. CIFS deletes are otherwise routed into a hidden `#Recycle/`
   directory that silently consumes quota until manually emptied — a common
   cause of `No space left on device` errors that look like permission
   failures.
4. Optional but recommended: pre-create the project skeleton on the NAS:
   ```
   <SHARE>/<SUBPATH>/
     pipelines/
     projects/
     work/
     results/
   ```

### Step 3 — Create the credentials secret

Edit `k8s/nas-secret.yaml` and replace `YOUR-NAS-USERNAME` /
`YOUR-NAS-PASSWORD` with the SMB account credentials from Step 2.

**Never commit real credentials.** Two ways to keep them out of git:

```bash
# Option A: keep the file local-only after filling it in
git update-index --assume-unchanged k8s/nas-secret.yaml

# Option B: skip the YAML entirely and create the secret imperatively
kubectl -n nextflow create secret generic nas-creds \
    --from-literal=username='<user>' \
    --from-literal=password='<pass>'
```

Apply (Option A only):

```bash
kubectl apply -f k8s/nas-secret.yaml
```

Verify:

```bash
kubectl -n nextflow get secret nas-creds
```

### Step 4 — Create the PersistentVolume and PersistentVolumeClaim

Edit `k8s/nas-pv-pvc.yaml` and replace the placeholder in
`volumeAttributes.source`:

```yaml
source: "//<NAS-IP-OR-HOST>/<SHARE-NAME>/<SUBPATH-OPTIONAL>"
```

with the actual path you prepared in Step 2, e.g.:

```yaml
source: "//192.168.1.10/data/nextflow"
```

Apply:

```bash
kubectl apply -f k8s/nas-pv-pvc.yaml
```

Verify the PVC binds:

```bash
kubectl get pv nextflow-nas-pv
kubectl -n nextflow get pvc nextflow-nas-pvc
```

Expected: PVC `STATUS` is `Bound` within a few seconds. If it stays
`Pending`, see [Troubleshooting](#troubleshooting).

### Step 5 — Mount the same share on your workstation

To make your workstation see the same files at the same path as the pods,
mount the NAS share locally at `/mnt/nas` using the same source string:

```bash
# /etc/samba/credentials.nas (root-readable only)
sudo tee /etc/samba/credentials.nas >/dev/null <<EOF
username=<your-nas-username>
password=<your-nas-password>
EOF
sudo chmod 600 /etc/samba/credentials.nas

# /etc/fstab line
echo '//<NAS-IP>/<SHARE>/<SUBPATH>  /mnt/nas  cifs  credentials=/etc/samba/credentials.nas,uid=1000,gid=1000,forceuid,forcegid,vers=3.0,file_mode=0755,dir_mode=0755,actimeo=1,mfsymlinks,_netdev,nofail  0  0' | sudo tee -a /etc/fstab

sudo mkdir -p /mnt/nas
sudo mount /mnt/nas
ls /mnt/nas      # should show the same content as the pod view
```

### Step 6 — Verify pod and host see the same files

Optional but recommended on first setup of a new NAS — it catches subpath
and dialect mismatches before they affect a real run.

```bash
cat <<'EOF' | kubectl -n nextflow apply -f -
apiVersion: v1
kind: Pod
metadata: { name: nas-check }
spec:
  serviceAccountName: nextflow-sa
  restartPolicy: Never
  containers:
  - name: c
    image: busybox:1.36
    command: ["sh","-c","ls /mnt/nas; sleep 30"]
    volumeMounts:
    - { name: vol, mountPath: /mnt/nas }
  volumes:
  - { name: vol, persistentVolumeClaim: { claimName: nextflow-nas-pvc } }
EOF

kubectl -n nextflow wait --for=condition=Ready pod/nas-check --timeout=60s
diff <(kubectl -n nextflow exec nas-check -- ls /mnt/nas) <(ls /mnt/nas)
kubectl -n nextflow delete pod nas-check
```

The `diff` must be empty. Any output means the host mount and the PV's
`source` point at different paths; revisit Steps 4 and 5 before continuing.

### Step 7 — Stage the pipeline and a project on the NAS

```bash
# Replicate the pipeline checkout onto the NAS so pods can read $projectDir/*
rsync -a \
    --exclude='work/' \
    --exclude='.nextflow/' \
    --exclude='.nextflow.log*' \
    --exclude='results/' \
    --exclude='k8s/' \
    <local-shotgun-checkout>/ /mnt/nas/pipelines/shotgun/

# Stage one project's inputs
mkdir -p /mnt/nas/projects/<your-project>/inputs/fastq
cp <local-fastq-dir>/*.fastq.gz /mnt/nas/projects/<your-project>/inputs/fastq/

# Write a samplesheet pointing at the NAS-side fastq paths
cat > /mnt/nas/projects/<your-project>/inputs/samplesheet.csv <<EOF
sample,fastq_1,fastq_2
sample1,/mnt/nas/projects/<your-project>/inputs/fastq/sample1_R1.fastq.gz,/mnt/nas/projects/<your-project>/inputs/fastq/sample1_R2.fastq.gz
EOF
```

### Step 8 — Stage host reference assets (one-time per cluster)

The shotgun pipeline's `KNEAD_DATA` step removes host reads using a Bowtie2
host index. The pipeline default points `params.bowtie2_index` at
`s3://ngi-igenomes/igenomes/.../Bowtie2Index/` — a ~4 GB directory in a
public S3 bucket in `eu-west-1`. Re-downloading that on every run is slow
and wasteful, so stage the index onto the NAS once and reuse it.

A ready-to-apply Job manifest is provided in
`k8s/k8s-jobs/download-bowtie2-hg38.yaml`. It runs
`aws s3 cp --no-sign-request --recursive` inside an `amazon/aws-cli`
container and writes to `/mnt/nas/pipeline_assets/bowtie2/hg38/`.

```bash
kubectl apply -f k8s/k8s-jobs/download-bowtie2-hg38.yaml
kubectl -n nextflow wait --for=condition=Complete \
    job/bowtie2-hg38-download --timeout=20m
kubectl -n nextflow logs job/bowtie2-hg38-download | tail -20
```

Verify:

```bash
ls -lh /mnt/nas/pipeline_assets/bowtie2/hg38/
# expected: genome.{1,2,3,4,rev.1,rev.2}.bt2 (~4 GB total) + genome.fa (~3 GB)
```

Layout convention for shared reference data:

```
/mnt/nas/pipeline_assets/
  bowtie2/
    hg38/       ← bowtie2 host indexes; one subdir per genome build
    grch38/
  trimmomatic/
    adapters/
  ...
```

Use `pipeline_assets/` for static reference data shared across pipelines
(genome indexes, adapters, FASTA files). Use `databases/` for tool-specific
classifier databases (kraken2, metaphlan, kaiju, humann, …) — see
[Skipping tools and supplying local databases](#skipping-tools-and-supplying-local-databases).

### Step 9 — Run the pipeline

The pipeline ships with a `k8s` profile in `nextflow.config` that wires the
executor, PVC and mount path. From your workstation (launching from local
disk, **not** from `/mnt/nas`):

```bash
mkdir -p ~/nf-runs/shotgun/<your-project>
cd ~/nf-runs/shotgun/<your-project>

nextflow run /mnt/nas/pipelines/shotgun/main.nf \
    -profile k8s \
    --input            /mnt/nas/projects/<your-project>/inputs/samplesheet.csv \
    --outdir           /mnt/nas/projects/<your-project>/results \
    -w                 /mnt/nas/work/<your-project> \
    --igenomes_ignore  true \
    --fasta            /mnt/nas/pipeline_assets/bowtie2/hg38/genome.fa \
    --bowtie2_index    /mnt/nas/pipeline_assets/bowtie2/hg38/
```

Key flags:

- `--igenomes_ignore true` — disables `conf/igenomes.config`, which would
  otherwise set `params.fasta` / `params.bowtie2_index` to
  `s3://ngi-igenomes/...` and reach into S3 even when you've provided a
  local override.
- `--fasta` / `--bowtie2_index` — point at the local copies you staged in
  Step 8.

What happens:

1. Nextflow's head process runs on your workstation; it extracts the k8s
   plugin into `~/nf-runs/shotgun/<your-project>/.nextflow/` (local disk —
   safe).
2. For each process, the driver writes `.command.sh` / `.command.run` into
   `/mnt/nas/work/<your-project>/<hash>/`. CIFS makes those files visible
   to the cluster.
3. The driver calls the k8s API to create a pod in the `nextflow` namespace,
   running as `nextflow-sa`, with the PVC mounted at `/mnt/nas`.
4. The pod `cd`s into its work dir on the NAS, runs `.command.run`, and
   writes its outputs back through the same mount.

In a second terminal you can watch:

```bash
kubectl -n nextflow get pods -w
```

### Step 10 — Verify a successful run

Look for the standard Nextflow "Pipeline completed successfully" banner,
and:

```bash
ls /mnt/nas/projects/<your-project>/results/
```

You should find MultiQC reports, FastQC outputs, and any per-tool
directories you didn't `--skip_<tool>`.

---

## Kubernetes-specific configuration in `nextflow.config`

The k8s profile in `nextflow.config` looks like this:

```groovy
k8s {
    process {
        executor    = 'k8s'
        stageInMode = 'copy'
    }

    k8s {
        namespace        = 'nextflow'
        serviceAccount   = 'nextflow-sa'
        storageClaimName = 'nextflow-nas-pvc'
        storageMountPath = '/mnt/nas'
    }
    params {
        max_memory = '16.GB'
        max_cpus   = 4
    }
}
```

| Field | Why it's there |
|---|---|
| `process.executor = 'k8s'` | Tells Nextflow to submit task pods to the cluster instead of running locally. |
| `process.stageInMode = 'copy'` | Copies inputs into each task's work directory instead of symlinking. CIFS can host symlinks when the mount has `mfsymlinks`, but `copy` is the safer default on shared filesystems where the host CIFS mount and the in-pod SMB CSI mount may use different options. |
| `k8s.namespace` / `serviceAccount` | Must match what Step 1 created. |
| `k8s.storageClaimName` / `storageMountPath` | The PVC and mount point inside each task pod. **Do not set `storageSubPath`** — keep host and pod paths identical by encoding any subpath in the PV's `source` instead. |
| `max_memory` / `max_cpus` | Caps per-task resource requests. Adjust to fit your cluster's node sizes. |

The profile deliberately omits `workDir`, `params.input`, `params.outdir`
and `params.tracedir`. These are project-specific and supplied on every
run via `-w`, `--input` and `--outdir` (see [Step 9](#step-9--run-the-pipeline)).
`tracedir` follows `--outdir` automatically via the global default
`tracedir = "${params.outdir}/pipeline_info"`.

> **Note:** `-w` must point at a NAS-side directory (e.g.
> `/mnt/nas/work/<your-project>`). Omitting `-w` falls back to Nextflow's
> default `./work` inside the launchDir, which is on local disk and not
> visible to the task pods — every task will fail to find its work
> directory.

### About the `aws { anonymous = true }` block

The top of `nextflow.config` declares:

```groovy
aws {
    client {
        anonymous = true
    }
}
```

**This is enabled by default in this fork** because:

- The pipeline's shipped `params.fasta` / `params.bowtie2_index` point at
  public S3 paths (`s3://ngi-igenomes/...`).
- Most users running shotgun on a private k8s cluster do not have AWS
  credentials configured on the launch host.
- Without anonymous mode, Nextflow's S3 client refuses to read even a
  fully public bucket — it tries to fetch caller identity first and fails.

It is safe to leave enabled: it only affects how Nextflow signs outgoing
S3 requests, not what S3 grants those requests. The recommended setup
combines it with `--igenomes_ignore true` plus NAS-local `--fasta` /
`--bowtie2_index` (see [Step 9](#step-9--run-the-pipeline)) so S3 is not
contacted on normal runs; anonymous mode remains available as a fallback
for occasional S3 lookups against the iGenomes defaults.

**Comment it out** if both of the following are true:

- You always set `--igenomes_ignore true` and point all inputs at NAS-local
  paths (no `s3://` URIs anywhere).
- You have real AWS credentials configured (env vars or `~/.aws/credentials`)
  for any S3 work you do.

**Replace it with explicit credentials** for private S3 / MinIO:

```groovy
aws {
    client {
        endpoint          = 'http://minio.minio.svc.cluster.local:9000'
        s3PathStyleAccess = true
    }
    accessKey = "${System.getenv('AWS_ACCESS_KEY_ID')}"
    secretKey = "${System.getenv('AWS_SECRET_ACCESS_KEY')}"
}
```

---

## Skipping tools and supplying local databases

The full shotgun pipeline runs MetaPhlAn, Kraken2, Kaiju, HUMAnN,
Centrifuge and mOTUs. Each requires a multi-gigabyte database. For an
initial smoke test on a new cluster, or while databases are still being
staged on the NAS, skip the heavy modules:

```bash
nextflow run /mnt/nas/pipelines/shotgun/main.nf \
    -profile k8s \
    --input  /mnt/nas/projects/<your-project>/inputs/samplesheet.csv \
    --outdir /mnt/nas/projects/<your-project>/results \
    -w       /mnt/nas/work/<your-project> \
    --skip_metaphlan  true \
    --skip_kraken2    true \
    --skip_kaiju      true \
    --skip_humann     true \
    --skip_centrifuge true \
    --skip_motus      true
```

This leaves `INPUT_CHECK`, `CAT_FASTQ`, `FASTQC`, `KNEAD_DATA`,
`CUSTOM_DUMPSOFTWAREVERSIONS` and `MULTIQC` enabled — sufficient to confirm
that pods schedule, inputs stage correctly, and the pipeline completes.

Once you've staged the databases on the NAS, point each tool at its local
copy by overriding the corresponding `params.<tool>_db`. The pipeline
detects a non-null DB path and skips the automatic install process for that
tool, e.g.:

```bash
--kraken2_db    /mnt/nas/databases/kraken2/k2_standard_08gb_2024-09-04 \
--metaphlan_db  /mnt/nas/databases/metaphlan/v3.0.14 \
--humann_dna_db /mnt/nas/databases/humann/v3/chocophlan \
--humann_pro_db /mnt/nas/databases/humann/v3/uniref \
--kaiju_db      /mnt/nas/databases/kaiju/<version> \
--centrifuge_db /mnt/nas/databases/centrifuge/<version> \
--motus_db      /mnt/nas/databases/motus/v3.0.1
```

Convention used here:

- **`pipeline_assets/`** for static reference data shared across pipelines
  (host genome indexes, adapter sequences, FASTA files). Step 8 stages the
  Bowtie2 hg38 index here.
- **`databases/`** for tool-specific classifier databases (kraken2, kaiju,
  metaphlan, humann, centrifuge, motus). Use one subdir per tool, then per
  version.

The `k8s/k8s-jobs/` directory in this repo is the right place to add
additional one-shot download manifests as you stage more databases — they
follow the same pattern as `download-bowtie2-hg38.yaml`.

---

## Troubleshooting

### `Operation not supported` during plugin extraction

```
ERROR ~ .nextflow/plr/<uuid>/nf-k8s-1.2.2: Operation not supported
```

You are launching `nextflow run` from a CIFS mount that is missing the
options Nextflow's plugin loader needs. Two fixes:

- **Recommended:** move to a local-disk launch dir
  (e.g. `~/nf-runs/<run>`) and re-run. The launch dir is small and
  local-disk is always going to be the most compatible target.
- **Alternative:** if you must launch from a CIFS path, make sure the
  mount line includes `mfsymlinks` (so symlink creation works via the
  Minshall+French encoding) and `noperm` (so POSIX-mode operations
  don't get rejected by CIFS' permission emulation). The fstab example
  in [Step 5](#step-5--mount-the-same-share-on-your-workstation) has
  `mfsymlinks` — add `noperm` if you want to try this path.

### Pod fails with `bash: .command.run: No such file or directory`

The pod and the host see different physical directories under `/mnt/nas`.
Almost always caused by:

- The PV's `source` and the host's CIFS mount source disagreeing about
  `<subpath>`. Fix by aligning Step 4 and Step 5 to use the same string.
- A pod-level `storageSubPath` that doesn't match what the host mount
  shows. We do not set `storageSubPath` in the shipped k8s profile for
  this reason — encode any subpath in the PV's `source` only.

Re-run [Step 6](#step-6--verify-pod-and-host-see-the-same-files) to verify
alignment.

### Process fails with `Unable to create temporary directory ... No space left on device`

The NAS share is full. Common causes:

- The Recycle Bin is silently retaining deleted files. Empty it via the NAS
  admin UI and ideally disable it for the Nextflow subpath.
- A previous failed run left a partial tarball in `work/`. Clean with
  `rm -rf /mnt/nas/work/<your-project>`.
- The share is genuinely full because it's a shared NAS; coordinate with
  whoever else uses it.

CIFS aggressively caches `statfs` — after freeing space, force a refresh
with:

```bash
sudo umount /mnt/nas && sudo mount /mnt/nas
df -h /mnt/nas
```

### `cp` returns "Permission denied" on `/mnt/nas/...` even though `ls -la` shows the file is yours

CIFS mode bits are display-only; the underlying ACL is what actually
controls access. `rsync` typically succeeds where `cp` fails because rsync
uses a write-then-rename strategy that the NAS ACL allows. Prefer `rsync`
on CIFS or fix the share-side ACLs.

### Pod was `OOMKilled` during a large download

The CIFS client's writeback page cache counts against the pod's cgroup
memory limit. For large streaming downloads, either:

- Raise the pod's `resources.limits.memory` (16 Gi is sufficient for
  multi-tens-of-GB downloads), or
- Run a periodic `sync` in the wrapper script to flush dirty pages:
  ```sh
  ( while true; do sleep 10; sync; done ) &
  curl ... | tar -xzf - -C /mnt/nas/...
  ```

### `nextflow-nas-pvc` stays `Pending`

```bash
kubectl -n nextflow describe pvc nextflow-nas-pvc
```

Common causes:

- SMB CSI driver not installed or not running in the cluster.
- The PV's `volumeHandle` clashes with another PV using the same string;
  change `volumeHandle` to something unique.
- The secret `nas-creds` doesn't exist or is in the wrong namespace.
- The NAS source path doesn't exist on the server (you point at a subpath
  that hasn't been created yet).

### Pipeline appears to stall for minutes before any pod is created

The default `params.fasta` and `params.bowtie2_index` point at public S3
buckets. The launcher pulls them over the workstation's internet link
before any task pod is scheduled, which can take several minutes on a slow
connection.

Fix: stage the Bowtie2 index onto the NAS using the Job in
[Step 8](#step-8--stage-host-reference-assets-one-time-per-cluster), then
pass `--igenomes_ignore true` together with the local paths via `--fasta`
and `--bowtie2_index` (see [Step 9](#step-9--run-the-pipeline)). With those
flags the launcher does not contact S3.

---

## Files in this directory

| File | Purpose |
|---|---|
| `ns-sa-rbac.yaml` | Namespace + ServiceAccount + Role + RoleBinding |
| `nas-secret.yaml` | SMB credentials secret (placeholder values — fill in before applying) |
| `nas-pv-pvc.yaml` | PV (with SMB CSI source) + PVC the k8s profile binds to |
| `k8s-jobs/` | One-shot Job manifests for staging assets and databases onto the NAS |
| `k8s-jobs/download-bowtie2-hg38.yaml` | Downloads the Bowtie2 hg38 host index from `s3://ngi-igenomes` to `/mnt/nas/pipeline_assets/bowtie2/hg38/` |
| `README.md` | This document |

Apply order: `ns-sa-rbac.yaml` → `nas-secret.yaml` → `nas-pv-pvc.yaml`,
then jobs from `k8s-jobs/` as needed for asset / database staging.

---

## Tested versions

- Kubernetes v1.30
- `smb.csi.k8s.io` v1.15
- SMB protocol v3.0
- Nextflow 25.10.2
