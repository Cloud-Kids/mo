# MediOps — AWS Infrastructure Incident Recovery

## Incident Postmortem & Technical Bug Fix Report

Dokumen ini mencatat temuan kesalahan konfigurasi infrastruktur AWS, root cause, tindakan perbaikan, dan hasil validasi setelah proses recovery MediOps.

### 1. Incident Overview

MediOps mengalami sejumlah kesalahan konfigurasi pada infrastruktur AWS yang berdampak pada konektivitas jaringan, keamanan resource, konfigurasi container, database, messaging, dan monitoring.

Recovery dilakukan dengan pendekatan:

- Satu kali provisioning awal menggunakan AWS CloudFormation.
- Build dan push container image ke Amazon ECR satu kali.
- Perbaikan insiden dilakukan secara **in-place** menggunakan AWS CLI.
- **Tidak** melakukan CloudFormation stack update selama proses incident remediation.
- **Tidak** melakukan rebuild image untuk setiap perbaikan konfigurasi infrastruktur.

### Deployment environment

| Property | Value |
|---|---|
| Cloud provider | AWS |
| Region | us-east-1 |
| Container registry | Amazon ECR |
| Container orchestration | Amazon ECS (Fargate) |
| Load balancer | Application Load Balancer |
| Database | Amazon RDS (PostgreSQL 15) |
| Object storage | Amazon S3 |
| Messaging | Amazon SQS |
| Notifications | Amazon SNS |
| Secrets management | AWS Secrets Manager |
| Parameter management | AWS Systems Manager Parameter Store |
| Monitoring | Amazon CloudWatch |
| Infrastructure provisioning | AWS CloudFormation |

### 2. Recovery Summary

| Component | Final state |
|---|---|
| CloudFormation stack `mediops-incident` | CREATE_COMPLETE (tidak di-update) |
| ECS desired tasks | 2 |
| ECS running tasks | 2 |
| ECS task definition | Revision 2 |
| ALB target health | 2 healthy targets |
| Application container port | 3000 |
| Health check | `/health` |
| Database connectivity | CONNECTED |
| Storage connectivity | CONNECTED |
| Queue connectivity | CONNECTED |
| SQS redrive configuration | Configured (DLQ + maxReceiveCount=3) |
| End-to-end workflow | QUEUED → COMPLETED |
| S3 document storage | Private object storage |
| CloudWatch dashboard | `mediops-dashboard` created |
| CloudWatch alarm | `mediops-ecs-cpu-high` created |
| Auto Scaling Group | Min=2, Max=4, Desired=2, TargetTrackingScaling |

Application endpoint:

```
http://mediops-alb-1093140444.us-east-1.elb.amazonaws.com
```

> The endpoint is associated with the recovery environment and may no longer be available after the AWS Learner Lab session expires.

### 3. Bug Findings & Root Cause Analysis

Sebanyak **13 temuan** konfigurasi diidentifikasi selama audit infrastruktur.

---

#### I-01 — Incorrect Private Route Table

**Severity: Critical**

| | |
|---|---|
| Finding | Private route table (`mediops-private-rt`) menggunakan `0.0.0.0/0 → Internet Gateway` |
| Root cause | Default route pada private route table mengarah ke gateway yang seharusnya hanya untuk public subnet. NAT Gateway `nat-055e3edfbf1e77930` sudah tersedia tetapi tidak digunakan |
| Impact | Private subnet tidak memiliki outbound internet yang valid; Fargate tasks dengan `AssignPublicIp: DISABLED` tidak dapat menjangkau ECR, Secrets Manager, S3, SQS, SNS, atau CloudWatch |
| Remediation | `aws ec2 delete-route` (0.0.0.0/0 via IGW) lalu `aws ec2 create-route` `0.0.0.0/0 → nat-055e3edfbf1e77930` pada route table privat |
| Verification | `describe-route-tables` menunjukkan `0.0.0.0/0 → NatGatewayId: nat-055e3edfbf1e77930`, state `active`; public route table tetap via IGW |

#### I-02 — Publicly Exposed Application Security Group

**Severity: Critical**

| | |
|---|---|
| Finding | `mediops-app-sg` mengizinkan `TCP 22` dan `TCP 3000` dari `0.0.0.0/0` |
| Root cause | Inbound rules memberikan akses langsung dari internet ke application tier, melewati ALB |
| Impact | Port aplikasi dan SSH terbuka untuk seluruh internet; melanggar security boundary |
| Remediation | Revoke kedua rule `0.0.0.0/0`; tambahkan `TCP 3000` source `mediops-alb-sg` (`sg-0d3e3cea34fdde364`) |
| Verification | `describe-security-groups`: hanya tersisa ingress TCP 3000 dari ALB-SG; tidak ada `0.0.0.0/0` pada app/db SG |

#### I-03 — Publicly Exposed Database Security Group

**Severity: Critical**

| | |
|---|---|
| Finding | `mediops-db-sg` mengizinkan `TCP 5432` dari `0.0.0.0/0` |
| Root cause | Database inbound access tidak dibatasi berdasarkan sumber workload aplikasi |
| Impact | PostgreSQL port terekspos ke internet (ditambah RDS `PubliclyAccessible: false`, tetapi rule tetap salah) |
| Remediation | Revoke `0.0.0.0/0`; tambahkan `TCP 5432` source `mediops-app-sg` (`sg-0fe8bc476752956cf`) |
| Verification | Hanya `App-SG → DB-SG : TCP 5432`; koneksi RDS dari ECS berhasil (readiness `database("SELECT 1")` PASS) |

#### I-04 — Incorrect ECS Task Definition & Service

**Severity: Critical**

| | |
|---|---|
| Finding | Task definition revision 1: `containerPort=80`, `APP_PORT=""`, `DB_HOST=""`, `DB_SECRET_ARN=""`; service `DesiredCount=0` |
| Root cause | Konfigurasi container dan service tidak sesuai runtime contract aplikasi (`APP_PORT=3000`, butuh `DB_HOST`, `DB_SECRET_ARN`, `DOCS_BUCKET`) |
| Impact | Tidak ada task berjalan; aplikasi tidak akan pernah siap |
| Remediation | Register task definition **revision 2**: `containerPort=3000`, `APP_PORT=3000`, `DB_HOST=<RDS endpoint>`, `DB_SECRET_ARN=<Secrets Manager ARN>`, `DOCS_BUCKET` tetap; update service: `desired-count=2`, load balancer `containerPort=3000` |
| Verification | Service `desired=2, running=2`; kedua target ALB healthy; `/health` merespons |

#### I-05 — Incorrect ALB Target Group Health Check

**Severity: High**

| | |
|---|---|
| Finding | Target group `mediops-tg` health check path `/` (bukan `/health`) |
| Root cause | Load balancer menggunakan path yang bukan application health endpoint; port target group sudah benar (3000) |
| Remediation | `aws elbv2 modify-target-group --health-check-path /health` |
| Verification | `describe-target-groups`: `HealthCheckPath=/health`; 2 target `healthy` |

#### I-06 — Insecure S3 Bucket Configuration

**Severity: High**

| | |
|---|---|
| Finding | Bucket `mediops-docs-devin01-684232006783`: Public Access Block semua `false`, versioning belum aktif (encryption SSE-S3 sudah default-enabled oleh AWS) |
| Root cause | Bucket belum menerapkan baseline perlindungan akses publik dan object versioning |
| Impact | Readiness gate gagal; risiko eksposur objek publik |
| Remediation | `put-public-access-block` (keempat flag `true`) + `put-bucket-versioning Status=Enabled` |
| Verification | `get-public-access-block` semua `true`; `get-bucket-versioning` = `Enabled`; upload dokumen berhasil via `/api/documents` |

#### I-07 — Database Password Stored as Plaintext

**Severity: High**

| | |
|---|---|
| Finding | `/mediops/db_password` bertipe `String` (plaintext) di Parameter Store |
| Root cause | Kredensial database disimpan tanpa enkripsi |
| Impact | Password dapat dibaca siapa pun dengan `ssm:GetParameter`; readiness mensyaratkan parameter ini tidak dapat diakses tanpa dekripsi |
| Remediation | Nilai password terlebih dahulu diamankan ke Secrets Manager `mediops-db-credentials`, kemudian parameter `/mediops/db_password` **dihapus** (`delete-parameter`). Catatan: mengubah ke `SecureString` saja tidak cukup — parameter tetap muncul pada `get_parameters` (dengan ciphertext) dan gate tetap gagal |
| Verification | `get-parameters` mengembalikan parameter pada `InvalidParameters`; tidak ada password plaintext yang tersisa |

#### I-08 — Missing Secrets Manager Database Credentials

**Severity: Critical**

| | |
|---|---|
| Finding | Tidak ada secret untuk kredensial database; `DB_SECRET_ARN` kosong |
| Root cause | Aplikasi membaca kredensial via `secretsmanager.get_secret_value(DB_SECRET_ARN)` dengan struktur JSON `{username, password, dbname}` |
| Impact | Koneksi PostgreSQL mustahil tanpa secret ini |
| Remediation | `create-secret --name mediops-db-credentials` dengan JSON credentials yang cocok dengan RDS (`mediops_admin` / `mediops` / password master); ARN dimasukkan ke `DB_SECRET_ARN` task definition rev 2 |
| Verification | `describe-task-definition` rev 2 memuat ARN secret; readiness `secret.username == MasterUsername` dan `dbname == DBName` PASS |

#### I-09 — Missing SQS Dead Letter Queue

**Severity: Medium**

| | |
|---|---|
| Finding | `mediops-jobs-queue` tidak memiliki RedrivePolicy maupun DLQ |
| Root cause | Failure handling untuk message yang gagal diproses belum dikonfigurasi |
| Remediation | Buat `mediops-jobs-dlq` (retensi 14 hari), set `RedrivePolicy` pada queue utama: `deadLetterTargetArn=<DLQ ARN>`, `maxReceiveCount=3` |
| Verification | `get-queue-attributes` menunjukkan RedrivePolicy valid; dashboard API `redrive_configured: true` |

#### I-10 — Missing SNS Subscription

**Severity: Medium**

| | |
|---|---|
| Finding | Topic `mediops-alerts` tidak memiliki subscription |
| Root cause | Notification topic belum terhubung ke consumer |
| Remediation | Buat queue `mediops-alerts-queue` + queue policy mengizinkan `sns.amazonaws.com` `sqs:SendMessage` dari topic tersebut, lalu `sns subscribe` protocol `sqs` |
| Verification | `list-subscriptions-by-topic` menampilkan subscription SQS aktif |

#### I-11 — Missing CloudWatch Monitoring

**Severity: Medium**

| | |
|---|---|
| Finding | Tidak ada dashboard `mediops-dashboard` maupun alarm ber-prefix `mediops-` yang mengarah ke SNS topic |
| Root cause | Monitoring resources belum dikonfigurasi |
| Remediation | `put-dashboard mediops-dashboard` (ECS CPU, ALB HealthyHostCount, RDS CPU) + `put-metric-alarm mediops-ecs-cpu-high` (AWS/EC2 CPUUtilization per ASG, threshold 80%, actions → SNS topic) |
| Verification | `list-dashboards` dan `describe-alarms --alarm-name-prefix mediops-` mengembalikan resource dengan `AlarmActions` ke `mediops-alerts` |

#### I-12 — Insufficient Auto Scaling Configuration

**Severity: Medium**

| | |
|---|---|
| Finding | ASG `mediops-asg`: Min=1, Max=1, Desired=1, tanpa scaling policy |
| Root cause | Kapasitas minimum dan mekanisme scaling berdasarkan beban belum tersedia |
| Remediation | `update-auto-scaling-group` Min=2, Max=4, Desired=2; `put-scaling-policy` tipe `TargetTrackingScaling` (`ASGAverageCPUUtilization`, target 50%) |
| Verification | `describe-auto-scaling-groups` dan `describe-policies` sesuai ekspektasi. **Catatan:** ASG adalah mekanisme scaling EC2 yang terpisah dari ECS service `desiredCount` — keduanya tidak boleh disamakan |

#### I-13 — Incorrect Database Host Parameter

**Severity: Low**

| | |
|---|---|
| Finding | `/mediops/db_host` = `db.internal` (placeholder tidak valid) |
| Root cause | Hostname belum mengarah ke RDS endpoint aktual |
| Remediation | `put-parameter` nilai `mediops-db.ckmbuqshmmp7.us-east-1.rds.amazonaws.com` (harus sama dengan `DB_HOST` env dan `rds.Endpoint.Address`) |
| Verification | Parameter berisi endpoint RDS; task dapat me-resolve dan connect ke PostgreSQL |

---

### 4. Recovery Execution Strategy

- **Phase 1 — Pre-deployment:** Docker pada WSL Ubuntu (linux/amd64), verifikasi IAM LabRole & ECR, AMI AL2023 terbaru.
- **Phase 2 — Container build:** `docker build` dari `app/`, push ke ECR `mediops-app:latest` — satu kali.
- **Phase 3 — Initial provisioning:** `cloudformation deploy` stack `mediops-incident` (satu kali; stack tidak di-update selama remediation).
- **Phase 4 — In-place remediation:** seluruh perbaikan di atas via AWS CLI pada resource terkait.
- **Phase 5 — Application validation:** tunggu service stabil → target health → health endpoint → connectivity DB/storage/queue → uji E2E.

### 5. Final Validation Results

| Test | Expected | Result |
|---|---|---|
| CloudFormation stack | CREATE_COMPLETE | PASS |
| ECS desired / running | 2 / 2 | PASS |
| Task definition | Revision 2 | PASS |
| ALB target health | 2 healthy | PASS |
| Application port / health path | 3000 / `/health` | PASS |
| Database / Storage / Queue connectivity | CONNECTED | PASS |
| SQS redrive | Configured | PASS |
| E2E queue workflow | QUEUED → COMPLETED | PASS (`completed:1`) |
| S3 document upload | Stored privately | PASS |
| Application restoration key | Diperoleh via `/api/restoration/check` | PASS |

### 6. CloudFormation Drift Notes

Semua perbaikan dilakukan **di luar** CloudFormation. Stack `mediops-incident` tetap `CREATE_COMPLETE` dan tidak pernah di-update, tetapi resource yang diubah in-place sekarang menyimpang dari template awal. Jika stack di-refresh atau diperbarui di kemudian hari, CloudFormation dapat mendeteksi drift atau mengembalikan nilai template pada operasi update berikutnya.

Resource yang saat ini **drift** dari template:

| Resource | Perubahan in-place |
|---|---|
| Private route table `mediops-private-rt` | Default route IGW → NAT Gateway |
| `mediops-app-sg` | Ingress 22/3000 `0.0.0.0/0` dicabut; ditambah ALB-SG→3000 |
| `mediops-db-sg` | Ingress 5432 `0.0.0.0/0` dicabut; ditambah App-SG→5432 |
| ECS service `mediops-service` | `DesiredCount` 0→2; task definition → rev 2; LB containerPort 80→3000 |
| ECS task definition | Revision baru (bukan revisi CFN): port 3000, env vars terisi |
| Target group `mediops-tg` | HealthCheckPath `/` → `/health` |
| S3 bucket `mediops-docs-*` | Public Access Block ON; versioning Enabled |
| SSM `/mediops/db_password` | **Dihapus** (parameter CFN tidak lagi ada) |
| SSM `/mediops/db_host` | `db.internal` → RDS endpoint |
| Secrets Manager | `mediops-db-credentials` dibuat (di luar template) |
| SQS | `mediops-jobs-dlq` + `mediops-alerts-queue` dibuat; RedrivePolicy + queue policy ditambahkan |
| SNS `mediops-alerts` | Subscription SQS ditambahkan |
| CloudWatch | `mediops-dashboard` + `mediops-ecs-cpu-high` dibuat |
| ASG `mediops-asg` | Min/Max/Desired 1→2/4/2; TargetTrackingScaling policy ditambahkan |
| ECR `mediops-app` | Repository dibuat di luar template |

Implikasi praktis:

- **Jangan jalankan `update-stack`** pada template asli — akan konflik dengan perbaikan in-place (mis. mencoba membuat ulang `/mediops/db_password` atau menimpa task definition).
- **Jangan `delete-stack`** jika ingin mempertahankan hasil recovery — deletion policy RDS adalah `Delete`, sehingga database akan ikut terhapus.
- Jika environment harus di-reprovision dari nol, perbaikan ini harus diterapkan ulang secara manual atau template harus diperbarui terlebih dahulu.

### 7. Key Technical Lessons

1. **Network segmentation must be explicit** — private subnet tidak boleh memakai IGW sebagai default route; gunakan NAT Gateway.
2. **Security groups should follow application traffic flow** — Internet → ALB → App-SG → DB-SG, hanya port yang diperlukan per hop.
3. **Container port configuration must be consistent** — Dockerfile `EXPOSE`, ECS `containerPort`, service LB mapping, TG port, dan health check path harus selaras.
4. **Secrets must not be stored as plaintext** — gunakan Secrets Manager; hindari password di environment variable, image, atau Parameter Store String.
5. **Messaging requires failure handling** — SQS memerlukan DLQ + redrive policy; SNS→SQS memerlukan queue policy.
6. **Infrastructure recovery is not the same as application validation** — deploy sukses ≠ aplikasi bekerja; validasi harus end-to-end (container health, target health, query DB, operasi S3, pengiriman message, failure handling).

### 8. Security Considerations

Sebelum environment digunakan di luar lab temporer:

- Rotate application key dan kredensial temporer.
- Jangan pernah commit application key, AWS credentials, database password, atau secret payload.
- Batasi ingress ALB ke sumber yang disetujui bila memungkinkan.
- Pertahankan RDS dan application tetap private.
- Review bucket policy dan public access settings S3.
- Review izin IAM role (`LabRole`).
- Konfirmasi queue policy SNS→SQS tetap restrict-to-topic.
- Validasi tujuan notifikasi alarm CloudWatch.
- Hapus resource lab secara aman setelah latihan.

> Credential handling: application keys are intentionally excluded from this report. Never publish a live or temporary key in a public repository.

### 9. Conclusion

Recovery MediOps menggabungkan provisioning awal satu kali via CloudFormation dengan remediasi in-place via CLI. Keadaan akhir yang tercapai:

- Dua ECS task berjalan, dua ALB target healthy.
- Konektivitas database, storage, dan queue berhasil.
- Pemrosesan SQS end-to-end berhasil (QUEUED → COMPLETED).
- Dokumen tersimpan di S3 privat.
- DLQ, monitoring, dan auto scaling terkonfigurasi.

Pelajaran utama: infrastruktur dianggap recovered bukan ketika resource ada, melainkan ketika aplikasi dapat melayani request, mengakses dependensinya, memproses message, dan mempertahankan security boundary yang dimaksud.

---

*Document type: AWS Incident Postmortem & Bug Fix Report — MediOps recovery lab, us-east-1. Approach: initial CloudFormation provisioning + in-place CLI remediation.*
