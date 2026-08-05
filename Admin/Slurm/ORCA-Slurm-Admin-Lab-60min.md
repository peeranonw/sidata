# ORCA — SLURM Admin Hands-on Lab (60 นาที)

**คลัสเตอร์:** ORCA @ SiData+ · Slurm 25.05 บน BCM 11.0 · orca-headnode (10.10.10.254) + orca-h200 (8× H200)
**รูปแบบ:** ผู้สอน demo บนจอ → ทีมพิมพ์ตามเฉพาะข้อที่มีเครื่องหมาย ⌨️
**ขอบเขต:** **อ่านอย่างเดียว + submit job ของตัวเอง** — ไม่มีคำสั่งใดใน lab นี้ที่แก้ config, แก้ account, drain node หรือกระทบ job ของคนอื่น

| สัญลักษณ์ | ความหมาย |
|---|---|
| ⌨️ | ให้ทุกคนพิมพ์ตามพร้อมกัน |
| 👁️ | ผู้สอนทำให้ดูอย่างเดียว (ไม่ต้องพิมพ์ตาม) |
| ❓ | หยุดถาม รอคำตอบจากห้อง (เฉลยอยู่ในวงเล็บ) |
| ⚠️ | จุดที่พลาดบ่อยตอนทำงานจริง |

---

## Timing รวม 60 นาที

| เวลา | Lab | หัวข้อ | คำถามหลักที่ตอบได้หลังจบ |
|---|---|---|---|
| 0:00–0:05 | **Lab 0** | เข้าเครื่อง + ตรวจว่า Slurm ยังมีชีวิต | "Slurm ตายหรือยัง ตรวจยังไงใน 10 วินาที" |
| 0:05–0:15 | **Lab 1** | อ่านสถานะ node และ GPU | "ตอนนี้มี GPU ว่างกี่ใบ node อยู่ state อะไร" |
| 0:15–0:25 | **Lab 2** | อ่าน config / partition / QOS / limit | "ทำไม user คนนี้ขอ GPU 8 ใบไม่ได้" |
| 0:25–0:40 | **Lab 3** | Submit job จริง (srun + sbatch + GPU) | "job วิ่งบนเครื่องไหน เห็น GPU กี่ใบ ทำไม" |
| 0:40–0:50 | **Lab 4** | Job ค้าง PENDING — อ่าน reason ให้เป็น | "ไม่ใช่ระบบพัง แต่ติดที่อะไร" |
| 0:50–0:58 | **Lab 5** | หลังงานจบ: accounting + log + sdiag | "เมื่อคืน job ล่ม ไปดูที่ไหน" |
| 0:58–1:00 | **Wrap** | Cheat sheet + สิ่งที่ **ห้าม** ทำ | — |

---

## ⚙️ Prep checklist (ผู้สอนทำ **ก่อน** เริ่มคลาส)

ทำล่วงหน้าอย่างน้อย 1 วัน — ถ้าข้อไหนไม่ผ่าน lab จะสะดุดกลางคัน

- [ ] ทุกคนมี account บน orca-headnode และ ssh เข้าได้จากที่นั่ง (ทดสอบจริง ไม่ใช่ "น่าจะได้")
- [ ] account ทุกคนมี Slurm association แล้ว — ตรวจ: `sacctmgr show assoc user=<u> format=User,Account,Partition,QOS`
      ⚠️ user ที่มีใน Linux แต่ยังไม่มีใน Slurm DB จะ submit ไม่ได้เลย ขึ้น `Invalid account`
- [ ] ยืนยันชื่อ partition จริง — เอกสารนี้ใช้ `defq` (ค่า default ของ BCM) ตรวจด้วย `sinfo -s` แล้วแก้ทั้งไฟล์ถ้าไม่ตรง
- [ ] ยืนยันชื่อ GRES จริง — เอกสารนี้เขียน `--gres=gpu:1` แบบไม่ระบุ type ตรวจว่ามี type name (เช่น `gpu:h200:8`) ไหมด้วย `scontrol show node orca-h200 | grep Gres`
- [ ] orca-h200 อยู่ state `idle` หรือ `mix` ไม่ใช่ `drain`/`down` (ถ้า drain lab 3 จะค้างทั้งห้อง)
- [ ] เว้น GPU ว่างไว้อย่างน้อย 2 ใบตอนคลาส — ถ้ามี production job กิน 8 ใบอยู่ Lab 3 จะกลายเป็น Lab 4 โดยไม่ตั้งใจ
- [ ] เตรียมไฟล์ job script ไว้ล่วงหน้าที่ `/cm/shared/apps/lab/` แล้วให้ทุกคน copy — เร็วกว่าให้พิมพ์ heredoc เอง
- [ ] มี terminal ตัวใหญ่ font ≥ 16pt สำหรับฉายจอ

**Copy ไฟล์ lab (ทำในนามผู้สอน ก่อนคลาส):**
```bash
mkdir -p /cm/shared/apps/lab
# วางไฟล์ hello-gpu.sh (ดู Lab 3) ไว้ในนี้ ให้ทุกคนอ่านได้
chmod 644 /cm/shared/apps/lab/hello-gpu.sh
```

---

# Lab 0 — เข้าเครื่อง และตรวจว่า Slurm ยังมีชีวิต (5 นาที)

**เป้าหมาย:** ทุกคนอยู่ที่ shell เดียวกัน และรู้คำสั่ง "10 วินาทีแรก" ตอนมีคนโทรมาบอกว่า "คลัสเตอร์ใช้ไม่ได้"

⌨️ **0.1 — เข้า headnode และสร้างที่ทำงานของตัวเอง**
```bash
ssh <ชื่อผู้ใช้ของคุณ>@10.10.10.254
mkdir -p ~/lab-slurm && cd ~/lab-slurm
```

⌨️ **0.2 — คำถามแรกเสมอ: controller ยังตอบไหม**
```bash
scontrol ping
```
คาดว่าจะเห็น:
```
Slurmctld(primary) at orca-headnode is UP
```

👁️ **0.3 — ถ้า ping ไม่ตอบ ค่อยไปดู service (ผู้สอนทำให้ดู)**
```bash
systemctl status slurmctld --no-pager | head -12
```

> **ลำดับการวินิจฉัยที่ถูกต้อง** — อย่าเริ่มจาก `systemctl` เสมอไป
> `scontrol ping` ตอบ = ตัว controller ปกติ ปัญหาอยู่ที่ node หรือที่ job ไม่ใช่ที่ Slurm
> `scontrol ping` ไม่ตอบ = ค่อยไล่ไปที่ service → log → DB

❓ ถาม: ถ้า `scontrol ping` ตอบ UP แต่ user บอกว่า submit job ไม่ได้ ปัญหาน่าจะอยู่ตรงไหน
*(เฉลย: ไม่ได้อยู่ที่ controller — ให้ไปดู association/account ของ user ใน slurmdbd หรือดู error message ที่ user เจอจริง ๆ ก่อน)*

---

# Lab 1 — อ่านสถานะ node และ GPU (10 นาที)

**เป้าหมาย:** ดูรูปคลัสเตอร์ในหัวออกจาก text 3 บรรทัด และแยกให้ออกว่า node "ยุ่ง" กับ node "พัง" ต่างกันยังไง

⌨️ **1.1 — ภาพรวมแบบสรุป**
```bash
sinfo -s
```
```
PARTITION AVAIL  TIMELIMIT   NODES(A/I/O/T)  NODELIST
defq*        up   infinite          0/1/0/1  orca-h200
```
ตัวเลข `A/I/O/T` = **A**llocated / **I**dle / **O**ther / **T**otal

⌨️ **1.2 — ดูรายละเอียดต่อ node พร้อม GPU**
```bash
sinfo -N -o "%.12N %.9P %.6t %.10C %.8m %.20G %E"
```
| คอลัมน์ | อ่านว่า |
|---|---|
| `%t` | state — `idle` ว่าง · `mix` ใช้บางส่วน · `alloc` เต็ม · `drain` ห้ามรับงานใหม่ · `down` ตายแล้ว |
| `%C` | CPU แบบ `Allocated/Idle/Other/Total` |
| `%G` | GRES — GPU ที่ node นี้ประกาศไว้ |
| `%E` | เหตุผลที่ถูก drain/down (ว่าง = ปกติ) |

⌨️ **1.3 — เจาะ node เดียวให้ละเอียด**
```bash
scontrol show node orca-h200
```
บรรทัดที่ต้องอ่านให้เป็น:
```
CPUAlloc=0 CPUTot=224 ...
Gres=gpu:8                      ← ประกาศไว้ 8 ใบ
GresUsed=gpu:0                  ← ใช้อยู่ 0 ใบ
State=IDLE ThreadsPerCore=2 ...
RealMemory=2064000 AllocMem=0 FreeMem=...
Reason=...                      ← มีบรรทัดนี้เมื่อโดน drain/down เท่านั้น
```

> ⚠️ **จุดพลาดที่เจอบ่อยที่สุดในคลัสเตอร์ GPU**
> `Gres=gpu:8` คือสิ่งที่ **Slurm เชื่อ** ไม่ใช่สิ่งที่ **มีจริงบนเครื่อง**
> ถ้า GPU ใบหนึ่งหลุดหลังไดรเวอร์อัปเดต Slurm จะยังคิดว่ามี 8 ใบ แล้วส่ง job ไปตายซ้ำ ๆ
> เพราะฉะนั้นเวลาตรวจสุขภาพจริง ต้องเทียบ **สองฝั่ง** เสมอ

👁️ **1.4 — เทียบสองฝั่ง: Slurm เชื่ออะไร vs เครื่องมีอะไร**
```bash
scontrol show node orca-h200 | grep -o 'Gres=[^ ]*'      # ฝั่ง Slurm
srun --gres=gpu:1 nvidia-smi -L                          # ฝั่งเครื่องจริง (ผ่าน Slurm)
```

❓ ถาม: `sinfo` ขึ้น `drain` แปลว่า node พังหรือไม่
*(เฉลย: ไม่จำเป็น — `drain` แปลว่า "ห้ามรับ job ใหม่" ซึ่งอาจเป็นเพราะ admin สั่งเองเพื่อทำ maintenance ก็ได้ ต้องอ่านช่อง `Reason=` ถึงจะรู้ว่าคนสั่งหรือระบบสั่ง)*

---

# Lab 2 — อ่าน config, partition, QOS และ limit (10 นาที)

**เป้าหมาย:** ตอบคำถามยอดฮิต "ทำไม user คนนี้ขอแบบนี้ไม่ได้" โดยไม่ต้องเดา

⌨️ **2.1 — ค่า config ที่ใช้บ่อยที่สุด 5 ตัว**
```bash
scontrol show config | grep -E "ClusterName|SlurmctldHost|AccountingStorage(Type|Host)|SelectType|GresTypes|StateSaveLocation"
```
| ค่า | ทำไมต้องรู้ |
|---|---|
| `SelectType` | ตัวตัดสินว่า Slurm จัดสรรทั้ง node หรือแบ่งเป็น core/GPU ได้ — `cons_tres` = แบ่ง GPU ได้ |
| `GresTypes` | ต้องมี `gpu` ไม่งั้น `--gres=gpu:1` จะถูกปฏิเสธ |
| `AccountingStorageType` | มี `slurmdbd` = มี history ให้ย้อนดู, ไม่มี = `sacct` จะว่างเปล่า |
| `StateSaveLocation` | ที่เก็บ state ของคิว — ถ้าโฟลเดอร์นี้เต็มหรือหาย job ในคิวหายทั้งหมด |

> ⚠️ **ที่ ORCA อย่าแก้ `slurm.conf` ด้วยมือ** — config ถูกสร้างจาก BCM
> แก้ตรงไฟล์แล้วรอบหน้าที่ BCM regenerate ของที่แก้จะหายทั้งหมด (การแก้ config ผ่าน cmsh อยู่นอกขอบเขต lab นี้)

⌨️ **2.2 — partition คือกติกาของช่องทางเข้า**
```bash
scontrol show partition
```
อ่าน: `MaxTime` · `DefaultTime` · `MaxNodes` · `AllowGroups` · `State=UP`

⌨️ **2.3 — QOS คือเพดานที่บังคับจริง**
```bash
sacctmgr show qos format=Name%-15,Priority,MaxWall,MaxTRESPU%-25,GrpTRES%-25,MaxJobsPU
```
| ช่อง | ความหมาย |
|---|---|
| `MaxTRESPU` | เพดาน **ต่อผู้ใช้หนึ่งคน** เช่น `gres/gpu=4` = คนเดียวขอเกิน 4 ใบไม่ได้ |
| `GrpTRES` | เพดาน **รวมทั้ง QOS** ทุกคนบวกกัน |
| `MaxWall` | เวลาสูงสุดต่อ job |
| `MaxJobsPU` | จำนวน job ที่รันพร้อมกันได้ต่อคน |

⌨️ **2.4 — association: ใครผูกอยู่กับ account/QOS ไหน**
```bash
sacctmgr show assoc format=Account%-15,User%-12,Partition,QOS%-20,GrpTRES%-20,MaxJobs
```

👁️ **2.5 — เจาะรายคน (ใช้ตอนมีคนโทรมาถาม)**
```bash
sacctmgr show assoc user=<ชื่อผู้ใช้> format=User,Account,QOS,GrpTRES,MaxJobs
```

**เส้นทางการตัดสินใจของ Slurm ตอน user กด submit — ต้องผ่านครบทั้ง 4 ด่าน:**
```
user submit
   │
   ├─ 1. user มี association ไหม?      ไม่มี → "Invalid account or partition"
   ├─ 2. partition รับ request นี้ไหม?  เกิน MaxTime/MaxNodes → ปฏิเสธทันที
   ├─ 3. QOS limit ผ่านไหม?             เกิน → รับเข้าคิว แต่ค้าง PENDING
   └─ 4. มีทรัพยากรว่างไหม?             ไม่มี → PENDING (Resources)
                                        มี   → RUNNING
```

❓ ถาม: user บ่นว่า "submit แล้ว job ไม่วิ่งเลย" ต่างจาก "submit แล้วขึ้น error ทันที" อย่างไร
*(เฉลย: ขึ้น error ทันที = ตกด่าน 1–2 คือ account/partition ผิด แก้ที่ config ฝั่ง admin · ไม่วิ่งแต่ไม่ error = ผ่านเข้าคิวแล้ว ติดด่าน 3–4 ให้ไปอ่าน reason ใน `squeue` ซึ่งคือ Lab 4)*

---

# Lab 3 — Submit job จริง (15 นาที)

**เป้าหมาย:** admin ต้อง submit เป็น เพราะการ reproduce ปัญหาของ user คือวิธีที่เร็วที่สุดในการหาสาเหตุ

## 3A — srun: งานสั้น เห็นผลทันที (5 นาที)

⌨️ **3.1 — job แรก: เครื่องนี้ชื่ออะไร**
```bash
srun --partition=defq --ntasks=1 --time=00:02:00 hostname
```
ต้องได้ `orca-h200` — ถ้าได้ `orca-headnode` แปลว่าไม่ได้วิ่งผ่าน Slurm

⌨️ **3.2 — ขอ GPU 1 ใบ**
```bash
srun --partition=defq --gres=gpu:1 --time=00:02:00 nvidia-smi -L
```
```
GPU 0: NVIDIA H200 (UUID: GPU-xxxxxxxx-...)
```

⌨️ **3.3 — ขอ 2 ใบ แล้วดูสิ่งที่ job "มองเห็น"**
```bash
srun --partition=defq --gres=gpu:2 --time=00:02:00 bash -c 'echo "CUDA_VISIBLE_DEVICES=$CUDA_VISIBLE_DEVICES"; nvidia-smi -L'
```

> **นี่คือหัวใจของ GPU scheduling** — Slurm ไม่ได้ "ห้าม" ให้เห็น GPU ใบอื่นด้วยกฎบนกระดาษ
> แต่ set `CUDA_VISIBLE_DEVICES` ให้ process มองไม่เห็นตั้งแต่แรก (ผ่าน cgroup)
> ขอ 2 ใบ → เห็น 2 ใบ นับ 0,1 เสมอ ไม่ว่าจริง ๆ จะได้ใบที่เท่าไรบนเครื่อง

❓ ถาม: user เขียนโค้ดฮาร์ดโค้ด `cuda:5` ไว้ แล้วขอ `--gres=gpu:2` จะเกิดอะไรขึ้น
*(เฉลย: พังทันที เพราะใน job เห็นแค่ device 0 กับ 1 — ให้ user แก้โค้ดใช้ device ที่ Slurm ให้ ไม่ใช่ให้ admin ไปเพิ่ม GPU)*

## 3B — sbatch: งานจริงต้องเข้าคิว (10 นาที)

⌨️ **3.4 — copy job script**
```bash
cp /cm/shared/apps/lab/hello-gpu.sh ~/lab-slurm/
cat ~/lab-slurm/hello-gpu.sh
```

**เนื้อไฟล์ `hello-gpu.sh`** (ผู้สอนเตรียมไว้ก่อนคลาส):
```bash
#!/bin/bash
#SBATCH --job-name=lab-hello-gpu
#SBATCH --partition=defq
#SBATCH --gres=gpu:1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=00:05:00
#SBATCH --output=%x-%j.out          # ชื่องาน-เลข job .out
#SBATCH --error=%x-%j.err

echo "=== job $SLURM_JOB_ID on $(hostname) ==="
echo "user       : $SLURM_JOB_USER"
echo "partition  : $SLURM_JOB_PARTITION"
echo "cpus       : $SLURM_CPUS_ON_NODE"
echo "gpus seen  : $CUDA_VISIBLE_DEVICES"
nvidia-smi -L
echo "--- ทำงานหลอก ๆ 60 วินาที เพื่อให้ทันเห็นใน squeue ---"
sleep 60
echo "=== done ==="
```

⌨️ **3.5 — submit**
```bash
cd ~/lab-slurm
sbatch hello-gpu.sh
```
```
Submitted batch job 1234
```

⌨️ **3.6 — ดูคิว (พิมพ์ซ้ำ 2–3 ครั้ง ให้เห็นสถานะเปลี่ยน)**
```bash
squeue -o "%.8i %.14j %.10u %.10a %.9T %.10M %.6D %R"
```
| ช่อง | ความหมาย |
|---|---|
| `%T` | State — `PD` เข้าคิว · `R` กำลังรัน · `CG` กำลังเก็บของ |
| `%M` | เวลาที่รันไปแล้ว |
| `%R` | node ที่วิ่งอยู่ **หรือ** เหตุผลที่ยังไม่วิ่ง ← ช่องสำคัญที่สุด |

⌨️ **3.7 — เจาะ job เดียว**
```bash
scontrol show job <JOBID>
```
บรรทัดที่ต้องอ่าน: `JobState=` · `Reason=` · `NodeList=` · `TRES=` · `StdOut=` · `WorkDir=`

⌨️ **3.8 — อ่านผลลัพธ์**
```bash
cat lab-hello-gpu-<JOBID>.out
```

👁️ **3.9 — ยกเลิก job ของตัวเอง (ผู้สอนทำ 1 ครั้งให้ดู)**
```bash
scancel <JOBID ของตัวเอง>
```
> ⚠️ `scancel` ยกเลิกได้เฉพาะ job ตัวเอง — ยกเว้น root/operator ที่ยกเลิกของคนอื่นได้
> ในการทำงานจริง **ห้าม** `scancel -u <user>` หรือ `scancel --state=PENDING` โดยไม่ได้แจ้งเจ้าของก่อน
> งานที่รันมา 20 ชั่วโมงหายในหนึ่งวินาที และไม่มี undo

---

# Lab 4 — Job ค้าง PENDING: อ่าน reason ให้เป็น (10 นาที)

**เป้าหมาย:** แยกให้ออกระหว่าง "ระบบพัง" กับ "ระบบทำงานถูกต้องแล้วแต่ user ขอเกิน" — ซึ่งคือสายที่ admin รับบ่อยที่สุด

⌨️ **4.1 — จงใจขอเกินของที่มี**
```bash
sbatch --partition=defq --gres=gpu:64 --time=00:05:00 --job-name=lab-toobig --wrap="sleep 30"
```

⌨️ **4.2 — ดูว่า Slurm บอกอะไร**
```bash
squeue --me -o "%.8i %.14j %.9T %.40R"
scontrol show job <JOBID> | grep -E "JobState|Reason"
```

⌨️ **4.3 — เก็บกวาดทันที (อย่าทิ้งไว้ในคิว)**
```bash
scancel <JOBID>
```

## ตาราง Reason ที่ต้องจำ

| Reason | แปลว่า | ต้องทำอะไร |
|---|---|---|
| `Resources` | ของมีจริงแต่ตอนนี้ไม่ว่าง | **ไม่ต้องทำอะไร** รอ — นี่คือระบบทำงานปกติ |
| `Priority` | มี job อื่นคิวสูงกว่ารออยู่ | รอ หรือคุยเรื่อง priority/QOS |
| `PartitionNodeLimit` | ขอเกินที่ partition นี้มีให้ | job นี้จะไม่มีวันรัน — ให้ user แก้ request |
| `QOSMaxGRESPerUser` `QOSMaxJobsPerUser` `AssocMaxJobsLimit` | ชน limit ของ QOS/association | ตัดสินใจเชิงนโยบาย ไม่ใช่ bug |
| `ReqNodeNotAvail` | node ที่ต้องใช้ถูก drain/down/reserve | ไปดู `sinfo -R` ว่า node โดนอะไร |
| `Dependency` | รอ job อื่นที่ผูกไว้จบก่อน | ดู job ต้นทางว่ายังอยู่ไหม |
| `JobHeldUser` / `JobHeldAdmin` | ถูก hold ไว้ | ปลดด้วย `scontrol release` (นอกขอบเขต lab นี้) |
| `launch failed requeued held` | ยิงลง node แล้วเด้ง มักเป็นปัญหา node จริง | **นี่แหละของจริง** ไปดู log ที่ Lab 5 |

⌨️ **4.4 — node ไหนโดน drain และเพราะอะไร**
```bash
sinfo -R
```
ว่างเปล่า = ไม่มี node ไหนถูก drain ถือว่าดี

👁️ **4.5 — ลำดับความสำคัญของคิวคำนวณมาจากอะไร**
```bash
sprio -l
sshare -a
```

> **กฎที่ควรติดตัวกลับไป:** PENDING ที่ reason เป็น `Resources` หรือ `Priority` = **ระบบทำงานถูกต้อง**
> อย่าเพิ่ง restart อะไรทั้งนั้น 90% ของสายที่โทรเข้ามาว่า "Slurm พัง" คือสองบรรทัดนี้

---

# Lab 5 — หลังงานจบ: accounting, log, sdiag (8 นาที)

**เป้าหมาย:** ตอบคำถาม "เมื่อคืน job ของผมล่ม เกิดอะไรขึ้น" ในเช้าวันรุ่งขึ้น

⌨️ **5.1 — ประวัติ job ของตัวเองวันนี้**
```bash
sacct -X --starttime today --format=JobID,JobName%-18,Partition,State%-14,Elapsed,ExitCode,NodeList
```
`-X` = แสดงเฉพาะ job หลัก ไม่แตก job step ย่อย (อ่านง่ายกว่ามาก)

⌨️ **5.2 — เจาะ job เดียวแบบละเอียด รวมทรัพยากรที่ได้จริง**
```bash
sacct -j <JOBID> --format=JobID,JobName%-18,Account,AllocTRES%-45,Elapsed,MaxRSS,State,ExitCode
```

## อ่าน State ให้เป็น

| State | สาเหตุที่แท้จริงมักคือ |
|---|---|
| `COMPLETED` (ExitCode 0:0) | จบปกติ |
| `FAILED` (ExitCode n:0) | **โค้ดของ user** คืนค่าไม่เป็น 0 — ไม่ใช่ปัญหาคลัสเตอร์ |
| `TIMEOUT` | ชน `--time` ที่ขอไว้ ให้ user ขอเวลาเพิ่ม |
| `OUT_OF_MEMORY` | เกิน `--mem` — เทียบ `MaxRSS` กับที่ขอไว้ให้ user ดู |
| `NODE_FAIL` | **ปัญหาฝั่งเรา** ไปดู log ต่อทันที |
| `CANCELLED` | มีคนสั่งยกเลิก — `sacct` บอกได้ว่าใครสั่ง |

👁️ **5.3 — ประวัติของคนอื่น (admin ทำได้)**
```bash
sacct -X -a --starttime now-1day --format=JobID,User%-12,JobName%-18,State%-14,Elapsed,ExitCode
```

👁️ **5.4 — log อยู่ตรงไหน**
```bash
sudo tail -50 /var/log/slurmctld                    # ฝั่ง controller: รับ/ปฏิเสธ/จัดคิว
ssh orca-h200 'sudo tail -50 /var/log/slurmd'       # ฝั่ง node: ยิง job ลงเครื่องแล้วเกิดอะไร
```
> **หลักการเลือก log:** อาการอยู่ที่ "job ไม่เข้าคิว / ถูกปฏิเสธ" → `slurmctld`
> อาการอยู่ที่ "เข้าคิวแล้วแต่ตายตอนเริ่มรัน" → `slurmd` ของ node นั้น

👁️ **5.5 — ตัว scheduler เองสุขภาพดีไหม**
```bash
sdiag
```
ดู `Last cycle` / `Mean cycle` (ไมโครวินาที) และ `Backfill` — ถ้า mean cycle โตผิดปกติแปลว่า scheduler เริ่มทำงานไม่ทันคิว

❓ ถาม: user บอก "job ตายเพราะคลัสเตอร์มีปัญหา" แต่ `sacct` ขึ้น `FAILED ExitCode 1:0` จริง ๆ แล้วคืออะไร
*(เฉลย: โปรแกรมของ user เอง exit ด้วยรหัส 1 — คลัสเตอร์รัน job ให้สำเร็จแล้ว ให้ชี้ user ไปดูไฟล์ `.err` ของตัวเอง ถ้าเป็นปัญหาฝั่งเราจะขึ้น `NODE_FAIL` แทน)*

---

# Wrap-up (2 นาที)

## Cheat sheet — 12 คำสั่งที่ใช้จริงทุกวัน

| อยากรู้ว่า... | คำสั่ง |
|---|---|
| Slurm ยังมีชีวิตไหม | `scontrol ping` |
| ภาพรวมคลัสเตอร์ | `sinfo -s` |
| node ไหนโดน drain เพราะอะไร | `sinfo -R` |
| node นี้มี GPU กี่ใบ ใช้ไปเท่าไร | `scontrol show node orca-h200` |
| ตอนนี้ใครใช้อะไรอยู่ | `squeue -o "%.8i %.12u %.9T %.10M %R"` |
| ทำไม job นี้ไม่วิ่ง | `scontrol show job <ID>` → อ่าน `Reason=` |
| ทดสอบ GPU เร็ว ๆ | `srun --gres=gpu:1 nvidia-smi -L` |
| ส่ง job จริง | `sbatch script.sh` |
| ยกเลิก job ตัวเอง | `scancel <ID>` |
| เมื่อวานเกิดอะไรขึ้น | `sacct -X -a --starttime now-1day` |
| user คนนี้มีสิทธิ์อะไร | `sacctmgr show assoc user=<u>` |
| scheduler ทำงานทันไหม | `sdiag` |

## สิ่งที่ **ห้าม** ทำบน ORCA (อยู่นอก lab นี้ทั้งหมด)

| ห้าม | เพราะ |
|---|---|
| แก้ `/etc/slurm/slurm.conf` ด้วยมือ | BCM เป็นเจ้าของไฟล์ รอบหน้าที่ regenerate ของที่แก้หายหมด |
| `scancel -u <user>` / `scancel --state=PENDING` | ล้างงานคนอื่นทั้งชุด ไม่มี undo |
| `scontrol update NodeName=... State=DOWN` | node หลุดจากคิวทันที job ที่รันอยู่ตาย |
| `systemctl restart slurmctld` เพื่อ "ลองดู" | ไม่ได้แก้ปัญหา 90% ที่เจอ และรบกวนทั้งคลัสเตอร์ — อ่าน `Reason=` ก่อนเสมอ |
| `sacctmgr delete ...` | ลบประวัติ accounting ทิ้งถาวร |

## 3 ประโยคที่อยากให้จำกลับไป

1. **อ่าน `Reason=` ก่อนแตะอะไรทั้งนั้น** — Slurm บอกสาเหตุไว้แล้วเกือบทุกครั้ง
2. **`Gres=` คือสิ่งที่ Slurm เชื่อ ไม่ใช่สิ่งที่เครื่องมี** — ตรวจสองฝั่งเสมอเวลาสงสัย GPU
3. **PENDING ไม่ใช่ error** — `Resources`/`Priority` แปลว่าระบบทำงานถูกต้องแล้ว

---

## หมายเหตุสำหรับผู้สอน — ต้องยืนยันกับเครื่องจริงก่อนใช้สอน

| รายการ | ที่ใช้ในเอกสารนี้ | ตรวจด้วย |
|---|---|---|
| ชื่อ partition | `defq` | `sinfo -s` |
| ชื่อ/type ของ GRES | `gpu:8` (ไม่ระบุ type) | `scontrol show node orca-h200 \| grep Gres` |
| จำนวน CPU / RealMemory ใน Lab 1.3 | `CPUTot=224`, `RealMemory=2064000` | `scontrol show node orca-h200` |
| path ของ log | `/var/log/slurmctld`, `/var/log/slurmd` | `scontrol show config \| grep -i logfile` |
| ชื่อ/ค่า QOS ใน Lab 2.3 | ตารางเป็นตัวอย่าง | `sacctmgr show qos` |
| สิทธิ์ sudo ของผู้สอนบน orca-h200 | ใช้ใน Lab 5.4 | ทดสอบก่อนคลาส |

**ถ้าเวลาเหลือ** (บาง batch จะเร็วกว่าที่คิด): ให้แต่ละคนแก้ `hello-gpu.sh` ขอ `--gres=gpu:2` และ `--mem=8G` แล้ว submit ใหม่ ดูว่า `MaxRSS` ใน `sacct` เปลี่ยนไหม และ `CUDA_VISIBLE_DEVICES` กลายเป็นอะไร

**ถ้าเวลาไม่พอ**: ตัด Lab 4.5 (`sprio`/`sshare`) และ Lab 5.5 (`sdiag`) ออกก่อน — สองข้อนี้เป็นของเสริม ส่วน Lab 1, 3, 4 ห้ามตัด
