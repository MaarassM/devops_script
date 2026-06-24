# Intro to DevOps — Dopune + lista zadataka po outcomeu (LO3–LO6)

> Tvoj cheatsheet i skripta su već vrlo potpuni. Ovaj dokument ima dva dijela:
> **DIO 1 — dopune** (stvari iz practice questiona koje ne pišu eksplicitno ili su nepotpune).
> **DIO 2 — "task hunting" lista**: za svaki outcome popis tipova zadataka + gotove komande za kopiranje.
>
> Gdje god treba ime/prezime: zamijeni `Horvat` / `Ivan Horvat`.

---

# DIO 1 — DOPUNE (što fali ili je nepotpuno)

## 1. ⭐ NAJVAŽNIJE: generiranje YAML-a s `--dry-run=client -o yaml`

Tvoj cheatsheet ima samo `kubectl get ... -o yaml` za postojeće resurse. Ali najbrži način da dobiješ **kostur manifesta** za nešto što tek treba napraviti je `--dry-run=client -o yaml`. Ne radiš ništa na clusteru, samo ispiše YAML koji onda urediš. Ovo ti štedi minute na ispitu i izbjegava greške s indentacijom (LO5!).

```bash
# Deployment kostur
kubectl create deployment web --image=nginx:1.25 --replicas=3 \
  --dry-run=client -o yaml > web.yaml

# Pod kostur
kubectl run mypod --image=nginx --dry-run=client -o yaml > pod.yaml

# Service kostur
kubectl expose deployment web --port=80 --target-port=80 \
  --dry-run=client -o yaml > svc.yaml

# Job / CronJob kostur
kubectl create job myjob --image=busybox --dry-run=client -o yaml -- echo hello
kubectl create cronjob mycron --image=busybox --schedule="* * * * *" \
  --dry-run=client -o yaml -- date

# ConfigMap / Secret kostur
kubectl create configmap cm1 --from-literal=K=V --dry-run=client -o yaml
kubectl create secret generic s1 --from-literal=K=V --dry-run=client -o yaml
```

Radni postupak za bilo koji "napiši manifest" zadatak: **generiraj kostur → otvori u editoru → dodaj ono što fale (probe, volumeni, resources, nodeSelector...) → `kubectl apply -f`.**

> Napomena: `--dry-run=client` ne radi za SVE (npr. StatefulSet, PV nemaju imperativni create). Za njih kreni od primjera iz docsa.

## 2. StatefulSet, DaemonSet, initContainer — nema imperativne komande, treba YAML

Skripta ih objašnjava konceptualno, ali nema gotovih manifesta. Pošto se NE mogu napraviti `kubectl create`-om, evo gotovih predložaka.

**StatefulSet (redis:7, 3 replike) + headless Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis          # headless servis
spec:
  clusterIP: None       # <- headless: vraća A-zapis po podu
  selector:
    app: redis
  ports:
    - port: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis    # mora pokazivati na headless servis
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7
          ports:
            - containerPort: 6379
```
Dokaz: `kubectl get pods` → imena `redis-0, redis-1, redis-2` (stabilna, po redu).

**DaemonSet (busybox agent — jedan pod po čvoru):**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: agent
spec:
  selector:
    matchLabels:
      app: agent
  template:
    metadata:
      labels:
        app: agent
    spec:
      containers:
        - name: agent
          image: busybox
          command: ["sleep", "infinity"]
```

**initContainer + emptyDir (init puni podatke, glavni čita):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: initdemo
spec:
  volumes:
    - name: shared
      emptyDir: {}
  initContainers:
    - name: init
      image: busybox
      command: ["sh", "-c", "echo hello > /data/file.txt"]
      volumeMounts:
        - name: shared
          mountPath: /data
  containers:
    - name: main
      image: busybox
      command: ["sh", "-c", "cat /data/file.txt && sleep infinity"]
      volumeMounts:
        - name: shared
          mountPath: /data
```

## 3. Probe + resources u kontekstu cijelog kontejnera (kompletan isječak)

Cheatsheet ima fragmente proba, ali ne kako sjede u kontejneru zajedno s resources. Evo cijeli blok koji zalijepiš pod `containers:`:

```yaml
    - name: web
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "200m"
          memory: "256Mi"
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
      readinessProbe:
        tcpSocket:
          port: 80
        periodSeconds: 5
      startupProbe:
        httpGet:
          path: /
          port: 80
        failureThreshold: 24      # 24 × 5s = 120s budžeta za boot
        periodSeconds: 5
```

## 4. Volume mount tipovi u podu (ConfigMap / Secret / PVC / emptyDir) — sve na jednom mjestu

```yaml
spec:
  containers:
    - name: app
      image: nginx
      envFrom:                       # SVI ključevi secreta kao env
        - secretRef:
            name: db-cred
      env:                           # JEDAN ključ kao env
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-cred
              key: DB_PASSWORD
      volumeMounts:
        - name: cfg
          mountPath: /etc/config     # ConfigMap kao volume (ključ = datoteka)
        - name: cfg-one
          mountPath: /etc/app/only.conf
          subPath: only.conf         # samo JEDAN ključ na točan path
        - name: sec
          mountPath: /etc/secret
          readOnly: true
        - name: data
          mountPath: /var/data       # trajni podaci (PVC)
        - name: tmp
          mountPath: /scratch        # privremeno (emptyDir)
  volumes:
    - name: cfg
      configMap:
        name: app-config
    - name: cfg-one
      configMap:
        name: app-config
    - name: sec
      secret:
        secretName: db-cred
        defaultMode: 0400            # restriktivne dozvole
    - name: data
      persistentVolumeClaim:
        claimName: data-pvc
    - name: tmp
      emptyDir:
        sizeLimit: 100Mi
```

PVC kostur (treba ga prije nego ga pod montira):
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
```

## 5. imagePullSecrets — kako se DockerHub login secret STVARNO koristi (LO4 #2)

Skripta kaže "stvori docker-registry Secret" ali ne pokaže gdje ide. Evo:
```bash
kubectl create secret docker-registry dockerhub \
  --docker-server=docker.io \
  --docker-username=<TVOJ_USER> \
  --docker-password=<TVOJ_PASS>
```
Pa u podu/deploymentu pod `spec.template.spec` (ili `spec` poda):
```yaml
      imagePullSecrets:
        - name: dockerhub
```

## 6. Multi-stage Containerfile (skripta opisuje, evo gotov primjer)

```dockerfile
# faza 1: build
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN go build -o app .

# faza 2: minimalna finalna slika
FROM alpine:3.20
COPY --from=builder /src/app /app
CMD ["/app"]
```
Poanta: `--from=builder` kopira SAMO gotov binarni file; toolchain ostaje u prvoj fazi i ne ide u finalnu sliku → slika padne s ~1 GB na par MB.

## 7. Sitnice koje nedostaju

- **`podman generate systemd` / Quadlet** (LO1 practice #22, može se pojaviti): `podman generate systemd --new --name <c>` ili Quadlet `.container` datoteka u `~/.config/containers/systemd/` → kontejner se diže na boot. Rijetko, ali spomenuto.
- **`podman healthcheck run <c>`** — ručno pokreni healthcheck; status vidiš u `podman ps` (kolona STATUS → healthy/unhealthy).
- **`.containerignore`** — kao `.gitignore`, izbacuje datoteke iz build konteksta (ekvivalent `.dockerignore`).
- **`podman commit <c> <slika>`** — već u cheatsheetu; zapamti da je to "ružni" način (ne reproducibilno) — na ispitu OK za demonstraciju.
- **Healthcheck u Containerfileu** puni oblik:
  `HEALTHCHECK --interval=5s --retries=3 CMD curl -f http://localhost/ || exit 1`
- **`kubectl create job --from=cronjob/<ime>`** — ručno okini job iz cronjoba (već u cheatsheetu, samo potvrda).

---

# DIO 2 — LISTA ZADATAKA PO OUTCOMEU + KOMANDE

> Za svaki outcome: tipovi zadataka koji se mogu pojaviti (iz practice questiona) + komanda/manifest koji ih rješava. Prepoznaj tip → primijeni obrazac.

## LO3 — Isporuka aplikacija, mreže, sigurnost (podman)

| Tip zadatka | Rješenje (komanda/obrazac) |
|---|---|
| **App + baza komuniciraju po imenu** | `podman network create net` → oba kontejnera s `--network net` → app koristi ime kontejnera baze kao host |
| **Pod s 2 kontejnera (localhost)** | `podman pod create --name p -p 8080:80` → `podman run -d --pod p ...` ×2 |
| **Two-tier stack (Ghost/Gitea/Redmine + DB)** | mreža + baza (env: user/pass/db) + app → first-run setup u browseru |
| **Perzistencija baze (named volume)** | `-v dbdata:/var/lib/postgresql/data` → `rm` + ponovo s istim volumenom → podaci tu |
| **Bind mount za perzistenciju/config** | `-v /host/dir:/cont/path:Z` (`:Z` za SELinux!) |
| **Secret umjesto --env** | `printf 'pw' \| podman secret create dbpass -` → `--secret dbpass` → datoteka u `/run/secrets/dbpass` |
| **Compose stack** | `compose.yaml` → `podman compose up -d` → `podman compose ps` |
| **Baza skrivena (interna mreža)** | u compose: `networks: backend: internal: true`, baza bez `ports:` |
| **Healthcheck + depends_on** | compose: `healthcheck:` + `depends_on: db: condition: service_healthy` |
| **env_file** | compose: `env_file: .env` |
| **Skaliranje servisa** | `podman compose up --scale app=3` |
| **Reverse proxy (nginx ispred app)** | nginx + app na istoj mreži; nginx prosljeđuje na `app:80` |
| **Backup volumena u tar** | `podman run --rm -v vol:/data -v $(pwd):/backup alpine tar czf /backup/b.tar.gz -C /data .` |
| **Most prema k8s** | `podman generate kube pod > pod.yaml` → `podman kube play pod.yaml` |
| **Segmentacija (2 mreže)** | `podman network connect <net2> <kontejner>` (kontejner na frontend+backend) |
| **Dokaz DNS-a** | `podman exec app getent hosts baza` (ime → IP) |

**Gotov LO3 stack (kopiraj-zalijepi):**
```bash
podman network create appnet
podman volume create dbdata

podman run -d --name db --network appnet \
  -e POSTGRES_PASSWORD=secret -e POSTGRES_USER=appuser -e POSTGRES_DB=appdb \
  -v dbdata:/var/lib/postgresql/data \
  docker.io/library/postgres:16

podman run -d --name app --network appnet -p 8080:80 \
  docker.io/library/<APP_SLIKA>

podman ps
podman exec app getent hosts db        # dokaz: DNS po imenu radi
```

## LO4 — Kubernetes (kubectl + minikube)

| Tip zadatka | Rješenje (komanda/obrazac) |
|---|---|
| **Deployment + izvoz YAML** | `kubectl create deployment web --image=nginx:1.25 --replicas=3` → `... -o yaml > web.yaml` |
| **Skaliranje (2 načina)** | `kubectl scale deployment web --replicas=5` / uredi `replicas:` + `apply` |
| **Rolling update** | `kubectl set image deployment/web nginx=nginx:1.27` → `kubectl rollout status deployment/web` |
| **Rollout history + rollback** | `kubectl rollout history deployment/web` → `kubectl rollout undo deployment/web` |
| **CHANGE-CAUSE** | `kubectl annotate deployment/web kubernetes.io/change-cause="v1.27"` |
| **Strategija RollingUpdate/Recreate** | u YAML: `spec.strategy.type` + `maxSurge`/`maxUnavailable` |
| **requests/limits** | u kontejneru `resources:` (vidi DIO 1, t.3) |
| **Label selector / list podova** | `kubectl get pods -l app=web` |
| **Expose (Service)** | `kubectl expose deployment web --port=80 --target-port=80` |
| **NodePort + URL** | `... --type=NodePort --name=web-svc` → `minikube service web-svc --url` |
| **ClusterIP DNS test** | debug pod → `wget -qO- web-svc` / `nslookup web-svc` |
| **ConfigMap/Secret create** | `kubectl create configmap/secret generic ... --from-literal=K=V` |
| **Secret kao env (envFrom)** | u podu `envFrom: - secretRef: name: ...` |
| **ConfigMap/Secret kao volume** | `volumeMounts` + `volumes:` (vidi DIO 1, t.4) |
| **StatefulSet + headless svc** | YAML iz DIO 1, t.2 |
| **DaemonSet / Job / CronJob** | `kubectl create job/cronjob ...` ili YAML iz DIO 1 |
| **Probe (liveness/readiness/startup)** | blok iz DIO 1, t.3 |
| **PV/PVC/StorageClass list** | `kubectl get pv,pvc,storageclass` |
| **DockerHub pull limit** | `kubectl create secret docker-registry ...` + `imagePullSecrets` (DIO 1, t.5) |
| **emptyDir / initContainer** | YAML iz DIO 1, t.2 i t.4 |

**Brzi LO4 niz (deployment → scale → update → expose → dokaz):**
```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl scale deployment web --replicas=5
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
kubectl get rs -o wide                       # aktivna RS = ne-nul replike, image 1.27
kubectl expose deployment web --name=web-svc --type=NodePort --port=80
minikube service web-svc --url               # URL za browser/screenshot
```

## LO5 — Troubleshooting (find → fix → explain)

> Uvijek struktura odgovora: **(1) greška, (2) ispravak, (3) zašto.** Tablice imaš u skripti; ovdje su DIJAGNOSTIČKE komande koje moraš pokrenuti da dođeš do uzroka.

| Simptom | Prva komanda za dijagnozu | Tipičan fix |
|---|---|---|
| Pod Pending | `kubectl describe pod <p>` (Events) | smanji requests / popravi nodeSelector / stvori PVC |
| ImagePullBackOff | `kubectl describe pod <p>` | ispravi tag / dodaj imagePullSecret |
| CrashLoopBackOff | `kubectl logs <p> --previous` | popravi uzrok pada |
| OOMKilled | `kubectl get pod <p> -o yaml` | digni memory limit |
| Completed (ubuntu/busybox) | `kubectl describe pod <p>` | dodaj `command: ["sleep","infinity"]` |
| CreateContainerConfigError | `kubectl describe pod <p>` | dodaj ključ u CM/Secret |
| nikad Ready | `kubectl describe pod <p>` | popravi readiness probe |
| Service prazan | `kubectl get endpoints <svc>` | uskladi selektor i labele |
| ProgressDeadlineExceeded | `kubectl describe deployment <d>` | riješi ImagePullBackOff/probe |
| Terminating zaglavljen | `kubectl describe pod <p>` | `--grace-period=0 --force` (rizik) |
| YAML typo/indentacija | `kubectl apply --dry-run=server --validate=true -f f.yaml` | ispravi polje/uvlaku |
| no-shell kontejner | `kubectl debug -it <p> --image=busybox --target=<c>` | debug iznutra |
| DNS unutar poda | `kubectl exec -it <p> -- nslookup <svc>` | provjeri /etc/resolv.conf, FQDN |

**Podman troubleshooting cheat (najčešći):**
```
-p obrnut             -> HOST:CONTAINER  (npr. 8081:80)
-e KEY (bez =val)     -> daj vrijednost
--network host + -p   -> makni jedno (-p se ignorira)
busybox Exited(0)     -> dodaj: sleep infinity
--rm -d ... echo      -> makni --rm (inače nema logova)
--memory 8m za bazu   -> digni limit (512m+)
default mreža, ping   -> napravi user-defined mrežu
-v ./dir:/path        -> dodaj :Z  (SELinux na Red Hat)
```

**Containerfile troubleshooting cheat:**
```
apt-get install bez update   -> RUN apt-get update && apt-get install -y X
CMD python app.py            -> CMD ["python","app.py"]  (exec forma, hvata signale)
COPY . pa RUN npm install    -> prvo COPY package.json, install, pa COPY .  (caching)
ENV PATH=/app/bin            -> ENV PATH=/app/bin:$PATH  (ne briši stari PATH)
EXPOSE ne publisha           -> uskladi port s -p / app portom
velika slika                 -> ... && rm -rf /var/lib/apt/lists/*  (isti RUN)
USER prije COPY              -> kopiraj/instaliraj kao root, USER na kraj (ili COPY --chown)
golang ~1GB                  -> multi-stage build (DIO 1, t.6)
```

## LO6 — Teorija (nema komandi, samo argumentacija)

> Tvoja skripta ovo POTPUNO pokriva (točke 1–23 + Swarm/k3s dodatak). Ovdje samo **struktura odgovora** i mini-šalabahter argumenata.

**Formula za svaki odgovor:** (1) jasan stav → (2) 2–3 konkretna argumenta ZA → (3) 1 protuargument/kompromis.

Mini-šalabahter (ključne riječi po temi):
- **Docker vs Podman:** daemon (root, SPOF) vs daemonless + rootless → Podman sigurniji.
- **k8s vs Compose (mali tim):** Compose = jednostavno/jeftino/jedan host; k8s tek kad treba HA+skaliranje.
- **OpenShift vs vanilla k8s:** OS = sigurnost po defaultu (SCC), S2I, integrirani alati, podrška → regulirane industrije; minus = lock-in, cijena.
- **k3s vs full k8s:** k3s lagan (jedan binary, SQLite) → edge/IoT/mali on-prem; full k8s overkill za malo.
- **Swarm:** Docker-native, najjednostavniji, ali u opadanju (potisnut k8s).
- **Storage k8s:** PVC + StorageClass + CSI → fleksibilno za stateful; Podman vezan uz host.
- **Vendor lock-in:** k8s open source/CNCF → manji lock-in; OpenShift Red Hat ekosustav.
- **Operator pattern:** CRD + controller → automatizirane day-2 operacije, ali trud za izradu.
- **CI/CD u k8s:** autoscaling + izolacija, rizik noisy-neighbor (riješi limitima).
- **Spiky promet (streaming):** k8s na cloudu + HPA + cluster autoscaler + više regija/CDN.

---

# CHECKLIST PRIJE PREDAJE SVAKOG ZADATKA
1. Naredba/YAML priložen u datoteku, označen `Lo<N>_<broj>`.
2. **Screenshot dokaza** (`ps`/`get`/`describe`/UI u browseru).
3. Za `_D` zadatak: **rečenica-dvije objašnjenja** (ne samo naredba).
4. Točni detalji iz teksta zadatka (slika, port, ime, env varijabla, tvoje ime/prezime).
5. Ako DockerHub limit → `podman login docker.io` / docker-registry Secret PRIJE rješavanja.
```
