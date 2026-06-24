# Intro to DevOps — Riješeni ispiti (LO3–LO6)

> Dva kompletna ogledna ispita, oba potpuno riješena, po uzoru na format pravog kolokvija.
> **Struktura:** svaki LO = 12 bodova / 30 min. Zadaci `_M` (osnovni: napravi i dokaži) i `_D` (teži: napravi **i objasni**).
> Gdje god treba ime/prezime: zamijeni `Horvat` / `Ivan Horvat` svojim podacima.
> Okruženje: LO3 = **podman**; LO4/LO5 = **kubectl + minikube**; LO6 = teorija.

---
---

# ISPIT A — group A (riješen)

## Learning outcome 3 — Isporuka aplikacija kontejnerima · 12 pts / 30 min

### Zadatak 1 · [LO3_M / 8 points] — Drupal + PostgreSQL

> Postavi Drupal i PostgreSQL kontejner pomoću Podmana ili podman-compose. Osiguraj da Drupal može komunicirati s PostgreSQL kontejnerom. Koristi env varijable: `POSTGRES_PASSWORD=my-secret-pw`, `POSTGRES_DB=drupal`, `POSTGRES_USER=drupal_user`, `PGPASSWORD=drupal_password`.

**Rješenje**
```bash
podman network create drupal-net

podman run -d --name postgres --network drupal-net \
  -e POSTGRES_PASSWORD=my-secret-pw \
  -e POSTGRES_DB=drupal \
  -e POSTGRES_USER=drupal_user \
  -e PGPASSWORD=drupal_password \
  docker.io/library/postgres:16

podman run -d --name drupal --network drupal-net -p 8080:80 \
  docker.io/library/drupal:latest
```

Završi Drupal instalaciju na `http://localhost:8080`. Kod koraka baze upiši:
- Database type: **PostgreSQL**, Name: `drupal`, User: `drupal_user`, Password: `my-secret-pw`
- Advanced → Host: **`postgres`** (ime kontejnera, NE localhost)

**Dokaz**
```bash
podman ps                                # oba kontejnera "Up"
podman exec drupal getent hosts postgres # ime "postgres" -> IP (DNS radi)
```
+ screenshot uspješne Drupal instalacije u browseru.

**Objašnjenje**
Kontejneri su na istoj user-defined mreži, pa ih ugrađeni podman DNS razrješava po imenu. Drupal kao host baze koristi `postgres` (ime kontejnera), a ne `localhost` — `localhost` bi značio sam Drupal kontejner. `POSTGRES_USER`+`POSTGRES_PASSWORD` kreiraju superuser nalog, `POSTGRES_DB` praznu bazu.

---

### Zadatak 2 · [LO3_D / 4 points] — Bind mount za perzistenciju

> Napravi direktorij `/drupal_data` na hostu. Bind mountom perzistiraj podatke Drupal kontejnera. Kontejner mora moći pisati u njega; demonstriraj.

**Rješenje**
```bash
sudo mkdir -p /drupal_data
sudo chmod 777 /drupal_data

podman rm -f drupal
podman run -d --name drupal --network drupal-net -p 8080:80 \
  -v /drupal_data:/opt/drupal/web/sites:Z \
  docker.io/library/drupal:latest
```

**Dokaz**
```bash
podman exec drupal touch /opt/drupal/web/sites/dokaz.txt
ls -l /drupal_data        # dokaz.txt vidljiv NA HOSTU -> pisanje radi
```

**Objašnjenje**
`-v /drupal_data:/opt/drupal/web/sites` mapira host direktorij u path gdje Drupal drži podatke; sve zapisano tamo živi na hostu i preživljava brisanje kontejnera. `:Z` prilagođava SELinux oznaku (na Red Hatu bez toga "Permission denied"). Caveat: mount preko `sites` ga na početku zasjeni praznim sadržajem — za dokaz pisanja to je u redu.

---

## Learning outcome 4 — Ubrzana isporuka (Kubernetes) · 12 pts / 30 min

### Zadatak 1 · [LO4_M / 4 points] — Deployment, scale, rolling update, ReplicaSets

> Kreiraj Deployment `web` (image `nginx:1.25`, 3 replike) jednom imperativnom naredbom. (a) skaliraj na 5; (b) rolling update na `nginx:1.27` + status; (c) izlistaj ReplicaSet-e i identificiraj aktivnu.

**Rješenje**
```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl scale deployment web --replicas=5                      # (a)
kubectl set image deployment/web nginx=nginx:1.27             # (b)
kubectl rollout status deployment/web
kubectl get rs -o wide                                        # (c)
```

**Dokaz**
```
NAME             DESIRED  CURRENT  READY  IMAGES
web-xxxxxxxxx    5        5        5      nginx:1.27   <- AKTIVNA
web-yyyyyyyyy    0        0        0      nginx:1.25   <- stara (0 replika)
```

**Objašnjenje**
Aktivna je ReplicaSet s ne-nul brojem replika (nova, `nginx:1.27`). Staru (`nginx:1.25`) rolling update je ispraznio na 0, ali je čuva za rollback. Veza: Deployment → ReplicaSet → Pod (preko selektora labela).

---

### Zadatak 2 · [LO4_M / 4 points] — Secret + ConfigMap u podu

> Kreiraj Secret `db-cred` (`DB_USER=appuser`, `DB_PASSWORD=<prezime>`) i ConfigMap `app-config` (`OWNER=<ime i prezime>`). Pokreni pod iz `nginx:1.25` koji učita sve ključeve Secreta kao env (`envFrom`) i montira ConfigMap kao volume na `/etc/config`. Dokaži oboje.

**Rješenje**
```bash
kubectl create secret generic db-cred \
  --from-literal=DB_USER=appuser --from-literal=DB_PASSWORD=Maras

kubectl create configmap app-config --from-literal=OWNER="Mihael Maras"

```bash
cat > app-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: nginx:1.25
      envFrom:
        - secretRef:
            name: db-cred
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: app-config
EOF
```


```bash
kubectl apply -f app-pod.yaml
```

**Dokaz**
```bash
kubectl exec app -- env | grep DB_          # DB_USER=appuser, DB_PASSWORD=Horvat
kubectl exec app -- cat /etc/config/OWNER   # Ivan Horvat
```

**Objašnjenje**
`envFrom + secretRef` ubacuje svaki ključ Secreta kao env varijablu istog imena. ConfigMap kao volume pretvara svaki ključ (`OWNER`) u datoteku u `/etc/config`.

---

### Zadatak 3 · [LO4_D / 4 points] — NodePort Service + provjera povezivosti

> Izloži `web` NodePort servisom `web-svc` (port 80), dohvati ga preko minikube URL-a. Iz throwaway poda provjeri povezivost preko DNS imena servisa. Objasni razliku port / targetPort / nodePort.

**Rješenje**
```bash
kubectl expose deployment web --name=web-svc --type=NodePort --port=80
minikube service web-svc --url
```
```bash
kubectl run tmp --rm -it --image=busybox -- sh
  wget -qO- web-svc:80        # nginx stranica -> radi
  nslookup web-svc            # DNS razrješava
  exit
```

**Objašnjenje**
- **port** — port na kojem Service sluša unutar clustera.
- **targetPort** — port na podu/kontejneru kamo Service prosljeđuje (80, nginx).
- **nodePort** — port otvoren na svakom čvoru (30000–32767) za pristup izvana.

Tok: `host:nodePort → Service:port → Pod:targetPort`.

---

## Learning outcome 5 — Troubleshooting · 12 pts / 30 min
> Za svaki: (1) greška, (2) ispravak, (3) zašto.

### Zadatak 1 · [LO5_M / 4 points] — Dvije pokvarene podman naredbe

**a) Obrnut port mapping**
```bash
# POKVARENO:  podman run -d --name web -p 80:8080 docker.io/library/nginx
# ISPRAVAK:
podman run -d --name web -p 8080:80 docker.io/library/nginx
```
Zašto: `-p` je `HOST:KONTEJNER`. nginx sluša na 80; kontejnerski port mora biti 80, ne 8080.

**b) busybox odmah izađe (Exited 0)**
```bash
# POKVARENO:  podman run -d --name c1 docker.io/library/busybox
# ISPRAVAK:
podman run -d --name c1 docker.io/library/busybox sleep 1d
```
Zašto: busybox nema dugotrajan proces; kontejner živi koliko i PID 1, pa odmah završi. `sleep 1d` ga drži živim.

---

### Zadatak 2 · [LO5_M / 4 points] — Containerfile koji ne builda

```dockerfile
# POKVARENO:
FROM debian:12
RUN apt-get install -y nginx
```
**Ispravak (uz čišćenje keša u istom layeru):**
```dockerfile
FROM debian:12
RUN apt-get update && \
    apt-get install -y nginx && \
    rm -rf /var/lib/apt/lists/*
```
Zašto: bez `apt-get update` lista paketa je prazna pa install padne. Čišćenje keša mora biti u **istom** `RUN` jer svaki RUN stvara novi layer — brisanje u kasnijem layeru ne smanjuje veličinu.

---

### Zadatak 3 · [LO5_D / 4 points] — Pod u CrashLoopBackOff

```yaml
# POKVARENO:
apiVersion: v1
kind: Pod
metadata:
  name: tool
spec:
  containers:
    - name: tool
      image: docker.io/library/ubuntu:24.04
```
**Dijagnoza**
```bash
kubectl describe pod tool       # State Terminated/Completed, restarti rastu
kubectl logs tool --previous    # nema greške aplikacije — samo izađe
```
**Ispravak**
```yaml
    - name: tool
      image: docker.io/library/ubuntu:24.04
      command: ["sleep", "infinity"]
```
**Objašnjenje**
ubuntu nema dugotrajan proces → izađe (exit 0) → default `restartPolicy: Always` ga stalno diže → CrashLoopBackOff. `sleep infinity` daje proces koji ne završava.

---

## Learning outcome 6 — Evaluacija orkestracije (teorija) · 12 pts / 30 min

### Zadatak 1 · [LO6_M / 4 points] — Docker vs Podman

**Arhitektura:** Docker ima centralni **daemon** (`dockerd`) koji radi kao root — single point of failure i veća napadna površina. Podman je **daemonless**: svaki poziv pokreće kontejner izravno, bez stalnog procesa.
**Rootless:** Podman radi bez root ovlasti (user namespace), pa proboj iz kontejnera ne daje root na hostu. Docker tradicionalno traži privilegirani daemon.
**Zaključak:** sigurnosno svjestan tim bira Podman — nema privilegiranog demona, rootless smanjuje posljedice kompromitacije; uz to je CLI-kompatibilan s Dockerom i integrira se sa systemd-om.

### Zadatak 2 · [LO6_M / 4 points] — Startup: Compose vs Kubernetes

**Stav:** za startup s dva inženjera → **Compose na jednom hostu**.
**Argumenti:** niska cijena (jedan VM), gotovo nikakav operativni overhead, kratka krivulja učenja, brz time-to-market; cijela app u jednom `compose.yaml`.
**Protuargument/kompromis:** Compose nema autoscaling, multi-node self-healing ni HA; jedan host je SPOF. Na Kubernetes prijeći tek kad se pojave stvarne potrebe za skaliranjem/dostupnošću — prerano uvođenje znači velik teret bez koristi.

### Zadatak 3 · [LO6_D / 4 points] — OpenShift vs vanilla k8s (regulirana industrija)

**Sigurnost:** OpenShift ima strože defaulte — SCC blokira root kontejnere, integrirani registry sa skeniranjem, ugrađene mrežne politike. Vanilla k8s sve to traži ručno složiti.
**Alati:** OpenShift donosi web konzolu, S2I build, integrirani CI/CD, monitoring/logging "out of the box"; na vanilla k8s to su odvojeni projekti.
**Protuargument:** vendor lock-in (Routes, SCC, S2I, `oc`) + trošak licence.
**Preporuka:** za reguliranu industriju koja želi integriranu, podržanu platformu OpenShift se isplati — ugrađena sigurnost, compliance i službena podrška vrijede više od rizika lock-ina. Za tim s jakom k8s ekspertizom i potrebom za fleksibilnošću ostaje opravdan vanilla k8s.

---
---

# ISPIT B — group B (novi zadaci, riješen)

## Learning outcome 3 — Isporuka aplikacija kontejnerima · 12 pts / 30 min

### Zadatak 1 · [LO3_M / 8 points] — Gitea + PostgreSQL (compose, segmentacija mreže)

> Pomoću podman compose postavi Gitea + PostgreSQL. Osiguraj da komuniciraju. Bazu drži na **internoj mreži bez objavljenog porta**, a izloži **samo** Gitea. Dokaži da baza NIJE dostupna s hosta.

**Rješenje** — `compose.yaml`:
```yaml
services:
  db:
    image: docker.io/library/postgres:16
    environment:
      POSTGRES_USER: gitea
      POSTGRES_PASSWORD: gitea
      POSTGRES_DB: gitea
    volumes:
      - giteadb:/var/lib/postgresql/data
    networks:
      - backend                       # samo interna mreža
  gitea:
    image: docker.io/gitea/gitea:latest
    environment:
      GITEA__database__DB_TYPE: postgres
      GITEA__database__HOST: db:5432  # baza po imenu servisa
      GITEA__database__NAME: gitea
      GITEA__database__USER: gitea
      GITEA__database__PASSWD: gitea
    ports:
      - "3000:3000"                   # samo app izložena
    networks:
      - backend                       # da doseže bazu
      - frontend                      # za izlaz/izloženost
    depends_on:
      - db
networks:
  backend:
    internal: true                    # nema pristupa s/na host
  frontend:
volumes:
  giteadb:
```
```bash
podman compose up -d
podman compose ps
```
Otvori `http://localhost:3000`, prođi Gitea first-run setup, napravi korisnika.

**Dokaz**
```bash
podman compose ps                          # db i gitea "Up", db bez host porta
podman exec gitea getent hosts db          # ime "db" -> IP (komunikacija radi)
curl -s --max-time 3 localhost:5432; echo "<- baza NIJE dostupna s hosta"
```
+ screenshot Gitea sučelja u browseru.

**Objašnjenje**
`db` je samo na `backend` mreži koja je `internal: true` i nema `ports:`, pa joj se ne može pristupiti s hosta — to je segmentacija (izloži app, sakrij bazu). Gitea je na obje mreže: `backend` da doseže bazu po imenu `db`, `frontend` za izloženost/izlaz. DNS po imenu servisa radi jer su na istoj user-defined mreži.

---

### Zadatak 2 · [LO3_D / 4 points] — Perzistencija u named volumenu

> Perzistiraj podatke baze u named volumenu tako da prežive brisanje i ponovno stvaranje kontejnera baze. Dokaži da su podaci i dalje tu.

**Rješenje**
Volumen `giteadb` već postoji iz Zadatka 1. Dokaz perzistencije:
```bash
# 1. Upiši marker u bazu
podman exec db psql -U gitea -d gitea -c "CREATE TABLE dokaz(id int); INSERT INTO dokaz VALUES (42);"

# 2. Obriši i ponovo stvori kontejner baze (volumen ostaje)
podman compose rm -sf db
podman compose up -d db

# 3. Provjeri da je podatak preživio
podman exec db psql -U gitea -d gitea -c "SELECT * FROM dokaz;"   # vraća 42
```

**Dokaz**
Zadnja naredba ispiše redak `42` iz tablice `dokaz` iako je kontejner baze u međuvremenu obrisan i ponovo stvoren.

**Objašnjenje**
Podaci baze žive u named volumenu `giteadb` (montiran na `/var/lib/postgresql/data`), izvan životnog ciklusa kontejnera. Brisanje kontejnera ne dira volumen, pa novi kontejner s istim volumenom vidi stare podatke. (Bez volumena podaci bi nestali s kontejnerom.)

---

## Learning outcome 4 — Ubrzana isporuka (Kubernetes) · 12 pts / 30 min

### Zadatak 1 · [LO4_M / 4 points] — Job + CronJob

> (a) Kreiraj Job (busybox) koji nešto izračuna jednom i završi; pokaži COMPLETIONS i pročitaj rezultat iz logova. (b) Kreiraj CronJob koji ispisuje datum svake minute; suspendiraj ga i pokaži Jobove koje stvara.

**Rješenje**
`job.yaml`:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: izracun
spec:
  template:
    spec:
      containers:
        - name: izracun
          image: busybox
          command: ["sh", "-c", "echo Rezultat: $((6 * 7))"]
      restartPolicy: Never
  backoffLimit: 2
```
```bash
kubectl apply -f job.yaml                                  # (a)

kubectl create cronjob datum --image=busybox \
  --schedule="* * * * *" -- date                           # (b)
kubectl patch cronjob datum -p '{"spec":{"suspend":true}}' # suspend
```

**Dokaz**
```bash
kubectl get job izracun           # COMPLETIONS 1/1
kubectl logs job/izracun          # "Rezultat: 42"

kubectl get cronjob datum         # SUSPEND True
kubectl get jobs                  # Jobovi koje je cron stvorio (datum-xx…)
```

**Objašnjenje**
Job pokreće pod **jednom do uspješnog završetka** (`COMPLETIONS 1/1`); `restartPolicy: Never` jer ne želimo restart nakon završetka. CronJob stvara Job po rasporedu (cron sintaksa `* * * * *` = svake minute); `suspend: true` ga pauzira bez brisanja.

---

### Zadatak 2 · [LO4_M / 4 points] — ClusterIP Service + DNS

> Kreiraj Deployment `api` i izloži ga ClusterIP servisom. Iz drugog (throwaway) poda razriješi servis preko DNS-a i dohvati ga.

**Rješenje**
```bash
kubectl create deployment api --image=nginx --replicas=2
kubectl expose deployment api --port=80 --target-port=80      # ClusterIP (default)
```
```bash
kubectl run tmp --rm -it --image=busybox -- sh
  nslookup api                 # razrješava na ClusterIP
  wget -qO- api                # vraća nginx stranicu
  exit
```

**Dokaz**
`nslookup api` vrati ClusterIP servisa; `wget -qO- api` ispiše nginx HTML. (Opcionalno `kubectl get endpoints api` → IP-evi 2 podova.)

**Objašnjenje**
ClusterIP servis daje stabilnu unutarklastersku adresu i DNS ime `api` (puni FQDN `api.default.svc.cluster.local`). Drugi pod ga doseže po imenu jer cluster DNS razrješava servis na njegov ClusterIP, koji load-balansira preko podova iz selektora.

---

### Zadatak 3 · [LO4_D / 4 points] — Liveness + readiness probe

> Dodaj HTTP livenessProbe i readinessProbe (port 80, putanja `/`) nginx Deploymentu i provjeri ih preko `kubectl describe`. Objasni zadano ponašanje kad proba nema.

**Rješenje** — `web-probe.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webp
  template:
    metadata:
      labels:
        app: webp
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 5
```
```bash
kubectl apply -f web-probe.yaml
```

**Dokaz**
```bash
kubectl describe pod -l app=webp     # sekcija Liveness/Readiness pokazuje http-get / na :80
```

**Objašnjenje**
- **livenessProbe** — provjerava je li kontejner živ; ako padne → k8s ga **restarta**.
- **readinessProbe** — je li spreman primati promet; ako ne → **izbaci ga iz Servicea** (ne restarta).
- **Bez proba (default):** k8s smatra Pod zdravim čim proces krene — ne provjerava odgovara li aplikacija. Zato app može "raditi" u statusu Running iako zapravo ne poslužuje promet.

---

## Learning outcome 5 — Troubleshooting · 12 pts / 30 min
> Za svaki: (1) greška, (2) ispravak, (3) zašto.

### Zadatak 1 · [LO5_M / 4 points] — Dvije pokvarene podman naredbe

**a) `--network host` ignorira `-p`**
```bash
# POKVARENO:  podman run -d --name app --network host -p 8080:80 docker.io/library/nginx
# ISPRAVAK (jedna od dvije opcije):
podman run -d --name app -p 8080:80 docker.io/library/nginx    # bez host mreže
# ILI: podman run -d --name app --network host docker.io/library/nginx  # app na host:80
```
Zašto: u `--network host` kontejner dijeli mrežu hosta direktno, pa `-p` nema učinka (port je već "na hostu"). Koristiš ili `-p` (bridge) ili `--network host`, ne oboje.

**b) `--rm` + detached → logovi nestanu**
```bash
# POKVARENO:  podman run --rm -d --name job docker.io/library/alpine echo hello
#             (zatim `podman logs job` ne radi)
# ISPRAVAK:
podman run -d --name job docker.io/library/alpine echo hello
podman logs job          # "hello"
```
Zašto: `echo` odmah završi, a `--rm` automatski obriše kontejner čim završi — pa kad zatražiš logove, kontejnera više nema. Makni `--rm` ako trebaš logove poslije.

---

### Zadatak 2 · [LO5_M / 4 points] — Dvije pogreške u Containerfileu

**a) Loš redoslijed za layer caching**
```dockerfile
# POKVARENO:
FROM node:20
COPY . /app
WORKDIR /app
RUN npm install
```
```dockerfile
# ISPRAVAK:
FROM node:20
WORKDIR /app
COPY package*.json ./        # prvo samo manifest ovisnosti
RUN npm install              # ovaj layer se kešira
COPY . .                     # tek onda ostatak koda
```
Zašto: kad `COPY . /app` dođe prije `npm install`, svaka promjena bilo kojeg fajla poništi cache i `npm install` se vrti iznova. Kopiranjem prvo `package.json` ovisnosti se keširaju i ne reinstaliraju dok se ne promijene.

**b) Prepisivanje PATH-a**
```dockerfile
# POKVARENO:   ENV PATH=/app/bin          (curl/sh se više ne nalaze)
# ISPRAVAK:
ENV PATH=/app/bin:$PATH
```
Zašto: postavljanjem samo `/app/bin` izbrisao si originalni PATH (`/usr/bin`, `/bin`...), pa alati nestanu. Dopiši, ne zamijeni.

---

### Zadatak 3 · [LO5_D / 4 points] — Pod u ImagePullBackOff

> Pod je u ImagePullBackOff. Dijagnosticiraj uzrok, ispravi i objasni.

```yaml
# POKVARENO (typo u tagu):
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  containers:
    - name: web
      image: nginx:1.25-alpinee     # <- nepostojeći tag
```
**Dijagnoza**
```bash
kubectl describe pod web      # Events: Failed to pull image ... not found / manifest unknown
```
**Ispravak**
```yaml
      image: nginx:1.25-alpine    # ispravan tag
```
```bash
kubectl delete pod web
kubectl apply -f web.yaml
kubectl get pod web           # Running
```
**Objašnjenje**
ImagePullBackOff znači da k8s ne može povući sliku — ovdje zbog **krivog taga** (`alpinee`). Drugi mogući uzroci: typo u imenu, privatni registry bez `imagePullSecrets`, ili DockerHub rate-limit (riješi `docker-registry` Secretom). `kubectl describe` u Events pokazuje točan razlog.

---

## Learning outcome 6 — Evaluacija orkestracije (teorija) · 12 pts / 30 min

### Zadatak 1 · [LO6_M / 4 points] — Mrežni model: Kubernetes vs Docker

**Docker/Podman mreža** — jednostavna, na jednom hostu: bridge mreže, DNS po imenu kontejnera, port publishing (`-p`). Dobra za malo kontejnera na jednom računalu.
**Kubernetes mreža** — distribuirana preko više čvorova: **svaki Pod ima vlastitu IP** (flat network, svi podovi se vide bez NAT-a), **Servisi** daju stabilne adrese i load balancing, **CNI** dodaci (Calico, Flannel) implementiraju mrežu, **NetworkPolicy** za segmentaciju.
**Zaključak:** Dockerova mreža je jednostavnija ali ograničena na host; Kubernetesova je složenija ali skalabilna i radi konzistentno preko cijelog clustera — nužno za distribuirane aplikacije.

### Zadatak 2 · [LO6_M / 4 points] — Full Kubernetes vs k3s (mali on-prem)

**k3s** — lagana, **certificirana** distribucija k8s (jedan binarni file, mali memorijski otisak, po defaultu SQLite umjesto etcd-a). Isti `kubectl`/API. Idealna za edge/IoT/male on-prem/Raspberry Pi.
**Full k8s** — veći resursni otisak (etcd, odvojene komponente control planea), složeniji za postaviti i održavati; opravdan tek uz puno čvorova i složene potrebe.
**Stav:** za malu on-prem instalaciju full Kubernetes je **overkill** — k3s daje istu kompatibilnost uz djelić resursa i truda. Full k8s biraš kad rasteš do velikog, višečvorišnog sustava s naprednim zahtjevima.

### Zadatak 3 · [LO6_D / 4 points] — Bi li odabrao Kubernetes za mikroservise?

**Stav: DA**, Kubernetes je dobar izbor za mikroservisnu arhitekturu.
**Argumenti ZA:**
- **Orkestracija** velikog broja servisa: deklarativno upravljanje, automatsko raspoređivanje, skaliranje po servisu (HPA).
- **Otpornost (resilience):** self-healing (restart palih podova), rolling update/rollback bez prekida, readiness/liveness provjere.
- **Ekosustav:** service discovery, Helm (paketi), Istio (service mesh), Prometheus (monitoring) — sve prilagođeno mikroservisima.
**Protuargument/kompromis:** velika **složenost** i day-2 operativni teret; za mali broj servisa ili mali tim može biti pretežak — tada Compose/k3s bolje balansiraju cijenu i korist.
**Zaključak:** za pravu mikroservisnu arhitekturu s mnogo servisa i potrebom za skaliranjem i pouzdanošću, prednosti Kubernetesa nadmašuju njegovu složenost.

---
---

# BRZI PODSJETNIK ZA OBA ISPITA
- **Dokaz uz svaki zadatak:** `podman ps` / `kubectl get` / `describe` / UI screenshot.
- **`_D` = uvijek dopiši objašnjenje** (1–2 rečenice), ne samo naredbu.
- **Točni detalji iz teksta:** slika, port, ime, env varijabla, tvoje ime/prezime.
- **DockerHub limit** (`toomanyrequests`) → `podman login docker.io` / docker-registry Secret PRIJE rješavanja.
- **Generiraj YAML brzo:** `kubectl create <...> --dry-run=client -o yaml > file.yaml`, pa uredi.
- **Prije apply provjeri:** `kubectl apply --dry-run=server --validate=true -f file.yaml`.
