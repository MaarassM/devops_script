# Intro to DevOps — UDŽBENIK (LO3–LO6)

> Svaka practice question je naslov (pitanje na koje naredba odgovara). Ispod: **Naredba → Objašnjenje → Dokaz**.
> Zamijeni `Horvat` / `Ivan Horvat` svojim podacima. `podman` = `docker`, `Containerfile` = `Dockerfile`.
> Prije k8s zadataka uvijek: `minikube status` (ako `i/o timeout` → `minikube start`).

---
---

# LO3 — Isporuka aplikacija, mreže, sigurnost (podman)

## 3.1 — Napravi pod (s objavljenim portom), dodaj app i bazu, pokaži da se vide preko localhosta

**Naredba**
```bash
podman pod create --name p -p 8080:80
podman run -d --pod p --name db -e POSTGRES_PASSWORD=pw docker.io/library/postgres:16
podman run -d --pod p --name app docker.io/library/httpd
```
**Objašnjenje:** Kontejneri u istom podu **dijele mrežni namespace**, pa se vide preko `localhost` (kao da su na istom računalu). Port se objavljuje na razini poda, ne pojedinog kontejnera.
**Dokaz**
```bash
podman exec app curl -s localhost:5432 || echo "baza odgovara na localhost"
podman pod ps
```

## 3.2 — User-defined mreža: app + postgres, dokaži razlučivanje po imenu

**Naredba**
```bash
podman network create net
podman run -d --network net --name db -e POSTGRES_PASSWORD=pw docker.io/library/postgres:16
podman run -d --network net --name app docker.io/library/httpd
```
**Objašnjenje:** Na user-defined mreži radi ugrađeni podman **DNS**, pa kontejner doseže drugi po **imenu kontejnera** (`db`), a ne po IP-u. Default mreža (`podman0`) nema DNS, pa ime tamo ne radi.
**Dokaz**
```bash
podman exec app getent hosts db    # ime "db" -> IP adresa
```

## 3.3 — Two-tier stack (NE Drupal) + first-run setup, npr. Gitea + PostgreSQL

**Naredba**
```bash
podman network create gitea-net
podman volume create giteadb
podman run -d --name db --network gitea-net -v giteadb:/var/lib/postgresql/data \
  -e POSTGRES_USER=gitea -e POSTGRES_PASSWORD=gitea -e POSTGRES_DB=gitea \
  docker.io/library/postgres:16
podman run -d --name gitea --network gitea-net -p 3000:3000 \
  -e GITEA__database__DB_TYPE=postgres -e GITEA__database__HOST=db:5432 \
  -e GITEA__database__NAME=gitea -e GITEA__database__USER=gitea -e GITEA__database__PASSWD=gitea \
  docker.io/gitea/gitea:latest
```
**Objašnjenje:** Dvoslojna aplikacija = app (Gitea) + baza (Postgres) na istoj mreži, app dohvaća bazu po imenu `db`. Env varijable konfiguriraju Gitea da odmah koristi Postgres.
**Dokaz:** otvori `http://localhost:3000`, prođi instalaciju, napravi korisnika → screenshot sučelja.

## 3.4 — Perzistencija baze u named volumenu (preživi rm + recreate)

**Naredba**
```bash
podman volume create dbdata
podman run -d --name db -v dbdata:/var/lib/postgresql/data -e POSTGRES_PASSWORD=pw docker.io/library/postgres:16
podman exec db psql -U postgres -c "CREATE TABLE t(x int); INSERT INTO t VALUES (42);"
podman rm -f db
podman run -d --name db -v dbdata:/var/lib/postgresql/data -e POSTGRES_PASSWORD=pw docker.io/library/postgres:16
```
**Objašnjenje:** Named volume živi **izvan** kontejnera, pa brisanje kontejnera ne dira podatke. Novi kontejner s istim volumenom vidi stare podatke; bez volumena podaci bi nestali s kontejnerom.
**Dokaz**
```bash
podman exec db psql -U postgres -c "SELECT * FROM t;"    # vraća 42
```

## 3.5 — Podman secret umjesto --env za lozinku baze

**Naredba**
```bash
printf 'tajna123' | podman secret create db_pass -
podman run -d --name db --secret db_pass \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/db_pass docker.io/library/postgres:16
```
**Objašnjenje:** Secret se montira kao datoteka u `/run/secrets/`, pa se lozinka **ne vidi** u `podman inspect` ni u listi procesa (za razliku od `--env`). Sigurnosna korist: tajna ne curi u metapodatke kontejnera.
**Dokaz**
```bash
podman exec db cat /run/secrets/db_pass     # tajna123
podman inspect db | grep -i password        # nema lozinke u inspectu
```

## 3.6 — Compose datoteka za app + bazu, podigni s up -d

**Naredba** — `compose.yaml`:
```yaml
services:
  db:
    image: docker.io/library/postgres:16
    environment:
      POSTGRES_PASSWORD: pw
  app:
    image: docker.io/library/httpd
    ports:
      - "8080:80"
```
```bash
podman compose up -d
```
**Objašnjenje:** Compose opisuje cijeli stack deklarativno u jednoj datoteci; `up -d` ga digne u pozadini i automatski stvori zajedničku mrežu na kojoj se servisi vide po imenu.
**Dokaz**
```bash
podman compose ps     # db i app oba "Up"
```

## 3.7 — Baza na internoj mreži (skrivena), izloži samo app

**Naredba** — `compose.yaml`:
```yaml
services:
  db:
    image: docker.io/library/postgres:16
    environment: {POSTGRES_PASSWORD: pw}
    networks: [backend]
  app:
    image: docker.io/library/httpd
    ports: ["8080:80"]
    networks: [backend, frontend]
networks:
  backend: {internal: true}
  frontend: {}
```
**Objašnjenje:** `internal: true` mreža nema vezu s hostom, a baza nema `ports:`, pa joj se ne može pristupiti izvana. App je na obje mreže — `backend` da doseže bazu, `frontend` za izloženost. To je **mrežna segmentacija**.
**Dokaz**
```bash
curl -s --max-time 3 localhost:5432; echo "<- baza nedostupna s hosta"
podman exec <app_kontejner> getent hosts db    # app i dalje vidi bazu
```

## 3.8 — Healthcheck + depends_on: service_healthy (app čeka bazu)

**Naredba** — `compose.yaml` (dio):
```yaml
services:
  db:
    image: docker.io/library/postgres:16
    environment: {POSTGRES_PASSWORD: pw}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5
  app:
    image: docker.io/library/httpd
    depends_on:
      db:
        condition: service_healthy
```
**Objašnjenje:** Healthcheck periodički provjerava je li baza spremna; `depends_on: service_healthy` drži app zaustavljenim dok baza ne postane "healthy". Time se izbjegava da app krene prije nego baza prihvaća konekcije.
**Dokaz**
```bash
podman compose up -d
podman compose ps     # db: healthy, app krene tek nakon toga
```

## 3.9 — env_file u compose umjesto inline vrijednosti

**Naredba** — `.env`:
```
POSTGRES_PASSWORD=pw
POSTGRES_DB=appdb
```
`compose.yaml` (dio):
```yaml
  db:
    image: docker.io/library/postgres:16
    env_file: .env
```
**Objašnjenje:** `env_file` izdvaja kredencijale iz compose datoteke u zasebnu `.env` datoteku, pa ne stoje "inline" u verzioniranom kodu. Lakše ih je mijenjati i držati izvan repozitorija.
**Dokaz**
```bash
podman compose up -d
podman exec <db> env | grep POSTGRES    # varijable učitane iz .env
```

## 3.10 — Skaliranje servisa na više replika

**Naredba**
```bash
podman compose up -d --scale app=3
```
**Objašnjenje:** `--scale` pokrene više instanci istog servisa. Zahtjevi se raspodjeljuju među replikama (ako je ispred reverse proxy ili compose-ova interna mreža s DNS round-robinom).
**Dokaz**
```bash
podman compose ps     # tri app instance
```

## 3.11 — Bind mount (config) + named volume (data) u istom kontejneru

**Naredba**
```bash
mkdir -p ./conf && echo "config" > ./conf/app.conf
podman run -d --name mix -v ./conf:/etc/app:Z -v appdata:/var/lib/app docker.io/library/httpd
```
**Objašnjenje:** **Bind mount** veže konkretnu host mapu (dobro za konfiguraciju koju uređuješ izvana); **named volume** je spremište kojim upravlja podman (dobro za podatke koji trebaju trajati i biti prenosivi). Svaki za svoju svrhu.
**Dokaz**
```bash
podman exec mix cat /etc/app/app.conf       # config s hosta vidljiv
podman volume inspect appdata               # volumen postoji
```

## 3.12 — Backup named volumena u tar + restore u novi

**Naredba**
```bash
podman run --rm -v appdata:/data -v $(pwd):/backup docker.io/library/alpine \
  tar czf /backup/backup.tar.gz -C /data .
podman volume create appdata2
podman run --rm -v appdata2:/data -v $(pwd):/backup docker.io/library/alpine \
  tar xzf /backup/backup.tar.gz -C /data
```
**Objašnjenje:** Pomoćni (alpine) kontejner montira volumen i host mapu pa tar-uje sadržaj na host. Za restore se isti tar raspakira u svjež volumen. Sam volumen se ne može direktno kopirati, pa ide preko helper kontejnera.
**Dokaz**
```bash
ls -lh backup.tar.gz                                  # arhiva nastala
podman run --rm -v appdata2:/data docker.io/library/alpine ls /data   # podaci vraćeni
```

## 3.13 — nginx reverse proxy ispred app

**Naredba** — `nginx.conf`:
```
events {}
http {
  server {
    listen 80;
    location / { proxy_pass http://app:80; }
  }
}
```
```bash
podman network create net
podman run -d --network net --name app docker.io/library/httpd
podman run -d --network net --name proxy -p 80:80 -v ./nginx.conf:/etc/nginx/nginx.conf:Z docker.io/library/nginx
```
**Objašnjenje:** Reverse proxy je jedinstvena ulazna točka — prima promet s hosta i prosljeđuje ga internoj aplikaciji po imenu (`app`). Korist: jedan izložen port, lakši TLS, app ostaje skrivena.
**Dokaz**
```bash
curl http://localhost:80     # vraća httpd stranicu preko proxyja
```

## 3.14 — DB-admin kontejner (adminer) za pregled baze

**Naredba**
```bash
podman run -d --network net --name adminer -p 8081:8080 docker.io/library/adminer
```
**Objašnjenje:** Adminer je web sučelje za baze; staviš ga na istu mrežu kao baza pa se preko preglednika spojiš (server = ime kontejnera baze). Korisno za provjeru da je podatak stvarno spremljen.
**Dokaz:** otvori `http://localhost:8081`, spoji se (System: PostgreSQL, Server: `db`, user/pass), pregledaj tablice → screenshot.

## 3.15 — Generiraj Kubernetes YAML iz poda

**Naredba**
```bash
podman generate kube p > pod.yaml
```
**Objašnjenje:** Iz pokrenutog poda generira ekvivalentni Kubernetes manifest. Time se **povezuje LO3 i LO4** — lokalni podman pod prevedeš u nešto što k8s razumije.
**Dokaz**
```bash
cat pod.yaml     # vidiš apiVersion/kind: Pod, kontejnere...
```

## 3.16 — Pokreni pod iz k8s YAML-a (kube play) i sruši ga

**Naredba**
```bash
podman kube play pod.yaml
podman kube play --down pod.yaml
```
**Objašnjenje:** `kube play` čita k8s manifest i podigne odgovarajući pod u podmanu (bez pravog Kubernetesa). `--down` ga uredno sruši.
**Dokaz**
```bash
podman pod ps     # pod se pojavi nakon play, nestane nakon --down
```

## 3.17 — CPU/memory limiti po servisu u compose + provjera

**Naredba** — `compose.yaml` (dio):
```yaml
  app:
    image: docker.io/library/httpd
    mem_limit: 256m
    cpus: 0.5
```
**Objašnjenje:** Ograničavaš resurse po servisu da jedan kontejner ne pojede sav RAM/CPU hosta. Provjeriš stvarnu potrošnju uživo s `podman stats`.
**Dokaz**
```bash
podman compose up -d
podman stats --no-stream     # LIMIT kolone pokazuju 256m / 50%
```

## 3.18 — Pod inspect: koje namespace dijele kontejneri

**Naredba**
```bash
podman pod inspect p
```
**Objašnjenje:** Kontejneri u podu **dijele network i IPC** namespace (zato se vide preko localhosta), ali **NE dijele PID** po defaultu (svaki vidi samo svoje procese). Inspect pokazuje konfiguraciju tih namespace-a.
**Dokaz:** u izlazu tražiš `SharedNamespaces` (sadrži `net`, `ipc`, ali ne `pid`).

## 3.19 — Razlučivanje po imenu pada na različitim mrežama → spoji na zajedničku

**Naredba**
```bash
podman network create net1 && podman network create net2
podman run -d --name a --network net1 docker.io/library/alpine sleep 1d
podman run -d --name b --network net2 docker.io/library/alpine sleep 1d
podman exec a ping -c1 b          # NE radi (različite mreže)
podman network connect net1 b     # popravak: spoji b i na net1
podman exec a ping -c1 b          # sada radi
```
**Objašnjenje:** DNS razlučivanje po imenu radi samo **unutar iste** user-defined mreže. Kontejneri na različitim mrežama se ne vide; spajanjem na zajedničku mrežu razlučivanje proradi.
**Dokaz:** drugi `ping` uspijeva, prvi ne.

## 3.20 — Jedan kontejner na dvije mreže (frontend + backend), segmentacija

**Naredba**
```bash
podman network create frontend && podman network create backend
podman run -d --name app --network backend docker.io/library/httpd
podman network connect frontend app
```
**Objašnjenje:** App je na obje mreže — frontend (proxy) i backend (baza) — pa komunicira s oba sloja, ali frontend nikad ne vidi backend direktno. To je **segmentacija**: smanjuje napadnu površinu.
**Dokaz**
```bash
podman inspect -f '{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}' app   # frontend backend
```

## 3.21 — compose logs + ps za troubleshooting

**Naredba**
```bash
podman compose logs
podman compose ps
```
**Objašnjenje:** `logs` skupi izlaz svih servisa na jednom mjestu (vidiš tko puca), `ps` pokazuje status svakog servisa. Prvi alati kad multi-service stack ne radi.
**Dokaz:** izlaz pokazuje logove + statuse svih servisa.

## 3.22 — Dijeljeni named volume kroz dvije replike + rizik

**Naredba**
```bash
podman volume create shared
podman run -d --name r1 -v shared:/data docker.io/library/httpd
podman run -d --name r2 -v shared:/data docker.io/library/httpd
```
**Objašnjenje:** Isti volumen montiran u više replika daje im istu konfiguraciju/podatke. **Rizik:** ako obje istovremeno pišu u iste datoteke (read-write), može doći do **korupcije podataka** (nema zaključavanja).
**Dokaz**
```bash
podman exec r1 sh -c "echo test > /data/f"
podman exec r2 cat /data/f     # r2 vidi isto -> dijeljeno
```

## 3.23 — compose down i sudbina volumena (--volumes)

**Naredba**
```bash
podman compose down              # briše kontejnere+mrežu, ČUVA named volumene
podman compose down --volumes    # briše i named volumene (podaci nestaju)
```
**Objašnjenje:** Obični `down` zaustavi i obriše kontejnere i mrežu, ali **named volumeni ostaju** (da se podaci ne izgube). `--volumes` ih izričito briše — koristi oprezno.
**Dokaz**
```bash
podman volume ls     # nakon obični down volumen postoji; nakon --volumes nestaje
```

---
---

# LO4 — Kubernetes (53 pitanja)

## 4.1 — Deployment web (nginx:1.25, 3 replike) jednom naredbom + izvoz YAML

**Naredba**
```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl get deployment web -o yaml > web.yaml
```
**Objašnjenje:** `create deployment` imperativno napravi Deployment; `-o yaml > file` izveze generirani manifest da ga možeš verzionirati/urediti. Deployment automatski stvara ReplicaSet koji drži 3 Poda.
**Dokaz**
```bash
kubectl get deployment web     # READY 3/3
```

## 4.2 — Omogući autentificirano povlačenje s DockerHuba (koji resurs?)

**Naredba**
```bash
kubectl create secret docker-registry dockerhub \
  --docker-server=docker.io --docker-username=<USER> --docker-password=<PASS>
```
**Objašnjenje:** Resurs koji trebaš je **docker-registry Secret** (tip `kubernetes.io/dockerconfigjson`). Referenciraš ga preko `imagePullSecrets` u Podu/Deploymentu da se k8s autentificira pri pull-u i izbjegne anonimni rate-limit.
**Dokaz**
```bash
kubectl get secret dockerhub     # TYPE kubernetes.io/dockerconfigjson
```

## 4.3 — Skaliraj 3→5 na dva načina + pokaži ReplicaSet

**Naredba**
```bash
kubectl scale deployment web --replicas=5          # način 1 (imperativno)
kubectl edit deployment web                        # način 2: promijeni replicas: 5
kubectl get rs
```
**Objašnjenje:** `scale` mijenja broj replika izravno; `edit` (ili uređivanje manifesta + `apply`) je deklarativni način. Oba mijenjaju isti ReplicaSet koji onda kreira dodatne Podove.
**Dokaz**
```bash
kubectl get rs     # DESIRED/CURRENT 5
```

## 4.4 — Rolling update nginx:1.25 → 1.27 + status

**Naredba**
```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
```
**Objašnjenje:** `set image` pokrene rolling update — k8s postupno diže nove Podove i gasi stare bez prekida. `rollout status` čeka i prati napredak dok update ne završi.
**Dokaz**
```bash
kubectl rollout status deployment/web     # "successfully rolled out"
```

## 4.5 — Rollout history + rollback + CHANGE-CAUSE

**Naredba**
```bash
kubectl annotate deployment/web kubernetes.io/change-cause="update na 1.27"
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
```
**Objašnjenje:** `history` pokazuje revizije; `undo` vraća na prethodnu. **CHANGE-CAUSE** kolona je prazna ako je ne popuniš anotacijom `kubernetes.io/change-cause` — služi da znaš što je svaka revizija mijenjala.
**Dokaz**
```bash
kubectl rollout history deployment/web     # vidiš revizije + change-cause
```

## 4.6 — Strategija RollingUpdate (maxSurge:1, maxUnavailable:0)

**Naredba** — u manifestu pod `spec:`:
```yaml
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```
**Objašnjenje:** `maxUnavailable: 0` znači da nijedan Pod ne smije nedostajati tijekom update-a; `maxSurge: 1` dopušta jedan višak. Rezultat: **nula prekida** dostupnosti jer uvijek imaš puni broj zdravih Podova.
**Dokaz**
```bash
kubectl apply -f web.yaml
kubectl get deployment web -o yaml | grep -A3 strategy
```

## 4.7 — Strategija Recreate + scenarij

**Naredba** — pod `spec:`:
```yaml
  strategy:
    type: Recreate
```
**Objašnjenje:** Recreate prvo ugasi **sve** stare Podove pa onda digne nove (uz kratak prekid). Treba kad stara i nova verzija **ne smiju raditi istovremeno** — npr. ekskluzivni lock na bazi ili nekompatibilna shema podataka.
**Dokaz**
```bash
kubectl rollout status deployment/web     # svi stari padnu prije novih
```

## 4.8 — CPU/memory requests i limits + provjera u pod specu

**Naredba**
```bash
kubectl set resources deployment web \
  --requests=cpu=100m,memory=128Mi --limits=cpu=200m,memory=256Mi
```
**Objašnjenje:** **requests** = koliko Pod garantirano dobije (koristi se za raspoređivanje), **limits** = gornja granica (prekoračenje memorije → OOMKilled). Definiraju resursni "ugovor" Poda.
**Dokaz**
```bash
kubectl get pod -l app=web -o jsonpath='{.items[0].spec.containers[0].resources}'
```

## 4.9 — revisionHistoryLimit: 3

**Naredba** — pod `spec:`:
```yaml
  revisionHistoryLimit: 3
```
**Objašnjenje:** Određuje koliko se **starih ReplicaSet-a** čuva za rollback. Manji broj štedi resurse, ali ograničava koliko se revizija unatrag možeš vratiti.
**Dokaz**
```bash
kubectl get rs     # najviše 3 stare (0-replika) RS uz aktivnu
```

## 4.10 — Nepostojeći tag → rollout zaglavi, stari Podovi i dalje rade

**Naredba**
```bash
kubectl set image deployment/web nginx=nginx:9.99
kubectl rollout status deployment/web      # zaglavi
kubectl get pods                           # novi ImagePullBackOff, stari Running
```
**Objašnjenje:** Novi Podovi ne mogu povući nepostojeću sliku (ImagePullBackOff), pa rolling update **ne ruši stare** dok novi ne postanu spremni. To je sigurnosna značajka — aplikacija ostaje dostupna unatoč lošem deployu.
**Dokaz**
```bash
kubectl get pods     # stari web podovi i dalje Running i serviraju
```

## 4.11 — Label selector: izlistaj Podove jednog Deploymenta

**Naredba**
```bash
kubectl get pods -l app=web
```
**Objašnjenje:** Deployment "hvata" svoje Podove preko labela (`selector.matchLabels`). Veza ide **Deployment → ReplicaSet → Pod**, svi povezani istim labelama.
**Dokaz:** ispisani samo Podovi s labelom `app=web`.

## 4.12 — Expose Deployment (koji objekt nastaje, kako selektor)

**Naredba**
```bash
kubectl expose deployment web --port=80 --target-port=80
```
**Objašnjenje:** Nastaje **Service** (default tip ClusterIP). Selektor Servicea se automatski izvede iz labela Deploymenta, pa Service zna koje Podove load-balansira.
**Dokaz**
```bash
kubectl get svc web
kubectl describe svc web     # Selector: app=web, Endpoints: IP-evi podova
```

## 4.13 — Sidecar (drugi kontejner u istom Podu)

**Naredba** — u pod templateu:
```yaml
      containers:
        - name: web
          image: nginx
        - name: sidecar
          image: busybox
          command: ["sh","-c","while true; do date; sleep 5; done"]
```
**Objašnjenje:** Sidecar je drugi kontejner u istom Podu koji pomaže glavnom (logiranje, proxy). Dijele **mrežu (localhost)** i mogu dijeliti **volumene**, pa usko surađuju.
**Dokaz**
```bash
kubectl get pod <p> -o jsonpath='{.spec.containers[*].name}'   # web sidecar
```

## 4.14 — Deployment httpd:2.4, 2 replike, named port, nodeSelector (Pending)

**Naredba** — pod template:
```yaml
    spec:
      nodeSelector:
        disk: ssd
      containers:
        - name: httpd
          image: httpd:2.4
          ports:
            - name: web
              containerPort: 80
```
**Objašnjenje:** `nodeSelector` traži čvor s labelom `disk=ssd`. Ako takvog čvora nema, Pod ostaje **Pending** jer ga scheduler ne može nigdje smjestiti.
**Dokaz**
```bash
kubectl get pods     # Pending; describe -> "node(s) didn't match nodeSelector"
```

## 4.15 — Bare Pod (busybox sleep), delete vs Deployment-managed

**Naredba**
```bash
kubectl run solo --image=busybox --restart=Never -- sleep 1d
```
**Objašnjenje:** Bare Pod nema kontroler iznad sebe — kad ga obrišeš, **nestaje zauvijek**. Pod kojim upravlja Deployment se nakon brisanja **automatski ponovo digne** (ReplicaSet ga vrati).
**Dokaz**
```bash
kubectl delete pod solo     # nestane i ne vraća se
```

## 4.16 — StatefulSet redis:7 (3 replike) + headless Service

**Naredba**
```yaml
apiVersion: v1
kind: Service
metadata: {name: redis}
spec:
  clusterIP: None
  selector: {app: redis}
  ports: [{port: 6379}]
---
apiVersion: apps/v1
kind: StatefulSet
metadata: {name: redis}
spec:
  serviceName: redis
  replicas: 3
  selector: {matchLabels: {app: redis}}
  template:
    metadata: {labels: {app: redis}}
    spec:
      containers:
        - name: redis
          image: redis:7
          ports: [{containerPort: 6379}]
```
**Objašnjenje:** StatefulSet daje **stabilna imena** (`redis-0/1/2`) i stabilan identitet po replici. Headless Service (`clusterIP: None`) ne daje jednu VIP nego **A-zapis po podu**, pa se obratiš točno određenoj replici.
**Dokaz**
```bash
kubectl get pods     # redis-0, redis-1, redis-2 (redom)
```

## 4.17 — DaemonSet (jedan Pod po čvoru)

**Naredba**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata: {name: agent}
spec:
  selector: {matchLabels: {app: agent}}
  template:
    metadata: {labels: {app: agent}}
    spec:
      containers: [{name: agent, image: busybox, command: ["sleep","infinity"]}]
```
**Objašnjenje:** DaemonSet osigurava **točno jedan Pod po čvoru** (za agente logova/monitoringa). Doda li se novi čvor, automatski dobije svoju kopiju Poda.
**Dokaz**
```bash
kubectl get pods -o wide     # jedan agent po svakom NODE
```

## 4.18 — Job (izračuna jednom, COMPLETIONS, čitaj logove)

**Naredba**
```bash
kubectl create job izracun --image=busybox -- sh -c "echo $((6*7))"
```
**Objašnjenje:** Job pokreće Pod **do uspješnog završetka** (ne drži ga stalno kao Deployment). Kad zadatak završi, Job je `Completed`, a rezultat pročitaš iz logova.
**Dokaz**
```bash
kubectl get job izracun      # COMPLETIONS 1/1
kubectl logs job/izracun     # 42
```

## 4.19 — CronJob (datum svake minute), suspend, popis Jobova

**Naredba**
```bash
kubectl create cronjob datum --image=busybox --schedule="* * * * *" -- date
kubectl patch cronjob datum -p '{"spec":{"suspend":true}}'
kubectl get jobs
```
**Objašnjenje:** CronJob stvara Job po rasporedu (cron sintaksa `* * * * *` = svake minute). `suspend: true` ga pauzira bez brisanja; svaki okidaj stvori novi Job.
**Dokaz**
```bash
kubectl get cronjob datum    # SUSPEND True
kubectl get jobs             # datum-xxxxx Jobovi
```

## 4.20 — Ručno okini Job iz postojećeg CronJoba

**Naredba**
```bash
kubectl create job rucno --from=cronjob/datum
```
**Objašnjenje:** `--from=cronjob/...` napravi Job na zahtjev koristeći predložak CronJoba, bez čekanja rasporeda. Korisno za testiranje cron posla odmah.
**Dokaz**
```bash
kubectl get job rucno     # COMPLETIONS 1/1
```

## 4.21 — StatefulSet stvara/gasi Podove redom (vs Deployment)

**Naredba**
```bash
kubectl scale statefulset redis --replicas=5
kubectl get pods -w
```
**Objašnjenje:** StatefulSet stvara Podove **redom** (`redis-0`, pa `-1`...) i gasi obrnutim redom. Deployment to radi paralelno jer su mu Podovi zamjenjivi; StatefulSet poštuje redoslijed zbog stabilnog identiteta.
**Dokaz:** `-w` (watch) pokazuje da nastaju jedan po jedan.

## 4.22 — Multi-container Pod dijeli emptyDir (jedan piše, drugi čita)

**Naredba**
```yaml
apiVersion: v1
kind: Pod
metadata: {name: share}
spec:
  volumes: [{name: shared, emptyDir: {}}]
  containers:
    - name: writer
      image: busybox
      command: ["sh","-c","echo hello > /data/f; sleep 1d"]
      volumeMounts: [{name: shared, mountPath: /data}]
    - name: reader
      image: busybox
      command: ["sh","-c","sleep 5; cat /data/f; sleep 1d"]
      volumeMounts: [{name: shared, mountPath: /data}]
```
**Objašnjenje:** `emptyDir` je privremeni volumen koji žive dok žive Pod; oba kontejnera ga montiraju, pa jedan zapiše datoteku, a drugi je pročita. Dokazuje dijeljenu pohranu unutar Poda.
**Dokaz**
```bash
kubectl logs share -c reader     # "hello"
```

## 4.23 — Popis StorageClass, PV, PVC

**Naredba**
```bash
kubectl get storageclass,pv,pvc
```
**Objašnjenje:** **StorageClass** je recept za dinamičko stvaranje pohrane, **PV** je stvarni komad pohrane, **PVC** je zahtjev Poda za pohranom. Jedna naredba ti pokaže sve tri razine.
**Dokaz:** ispis tri tipa resursa.

## 4.24 — ConfigMap kao volume (svaki ključ = datoteka)

**Naredba**
```bash
kubectl create configmap app-config --from-literal=OWNER="Ivan Horvat"
```
```yaml
      volumeMounts: [{name: cfg, mountPath: /etc/config}]
  volumes: [{name: cfg, configMap: {name: app-config}}]
```
**Objašnjenje:** Kad ConfigMap montiraš kao volume, **svaki ključ postaje datoteka** u mount pathu (ključ `OWNER` → `/etc/config/OWNER`). Tako app čita konfiguraciju kao obične datoteke.
**Dokaz**
```bash
kubectl exec <p> -- cat /etc/config/OWNER     # Ivan Horvat
```

## 4.25 — Jedan ključ ConfigMapa na točan path (subPath)

**Naredba**
```yaml
      volumeMounts:
        - {name: cfg, mountPath: /etc/app/only.conf, subPath: OWNER}
```
**Objašnjenje:** `subPath` montira **samo jedan ključ** na točnu datoteku, bez da zasjeni cijeli direktorij. Caveat: takav mount se **ne osvježava automatski** kad se ConfigMap promijeni.
**Dokaz**
```bash
kubectl exec <p> -- cat /etc/app/only.conf
```

## 4.26 — Secret kao volume (dekodirane vrijednosti, restriktivne dozvole)

**Naredba**
```yaml
      volumeMounts: [{name: sec, mountPath: /etc/secret, readOnly: true}]
  volumes:
    - name: sec
      secret: {secretName: db-cred, defaultMode: 0400}
```
**Objašnjenje:** Secret montiran kao volume daje **dekodirane** vrijednosti u datotekama (ne base64). `defaultMode: 0400` postavlja restriktivne dozvole (samo vlasnik čita), da tajna ne curi.
**Dokaz**
```bash
kubectl exec <p> -- cat /etc/secret/DB_PASSWORD     # čista vrijednost
kubectl exec <p> -- ls -l /etc/secret               # dozvole 0400
```

## 4.27 — Scaling StatefulSeta: PVC po replici (sudbina pri brisanju)

**Naredba**
```bash
kubectl get pvc
```
**Objašnjenje:** Svaka replika StatefulSeta dobije **vlastiti PVC** (preko `volumeClaimTemplates`). Pri brisanju StatefulSeta PVC-ovi **ostaju** (da se podaci ne izgube) — moraš ih obrisati ručno.
**Dokaz:** broj PVC-a = broj replika; ostaju i nakon `delete statefulset`.

## 4.28 — initContainer puni dijeljeni volumen prije glavnog

**Naredba**
```yaml
  initContainers:
    - name: init
      image: busybox
      command: ["sh","-c","echo data > /shared/f"]
      volumeMounts: [{name: shared, mountPath: /shared}]
  containers:
    - name: main
      image: busybox
      command: ["sh","-c","cat /shared/f; sleep 1d"]
      volumeMounts: [{name: shared, mountPath: /shared}]
  volumes: [{name: shared, emptyDir: {}}]
```
**Objašnjenje:** initContainer se izvrši **prije** glavnog i pripremi podatke u dijeljeni volumen. Glavni kontejner kreće tek kad init uspješno završi, pa zatekne pripremljene podatke.
**Dokaz**
```bash
kubectl logs <p> -c main     # "data"
```

## 4.29 — readOnly: true na mountu (pisanje odbijeno)

**Naredba**
```yaml
      volumeMounts: [{name: cfg, mountPath: /etc/config, readOnly: true}]
```
**Objašnjenje:** `readOnly: true` montira volumen samo za čitanje, pa svaki pokušaj pisanja vrati grešku. Korisno za konfiguraciju koju app ne smije mijenjati.
**Dokaz**
```bash
kubectl exec <p> -- sh -c "echo x > /etc/config/y"     # Read-only file system
```

## 4.30 — sizeLimit na emptyDir

**Naredba**
```yaml
  volumes: [{name: tmp, emptyDir: {sizeLimit: 100Mi}}]
```
**Objašnjenje:** `sizeLimit` ograničava koliko emptyDir smije narasti. Ako kontejner premaši limit, k8s **izbaci Pod** (eviction) da zaštiti čvor.
**Dokaz**
```bash
kubectl describe pod <p>     # Volume tmp SizeLimit: 100Mi
```

## 4.31 — Generic Secret iz literala (base64, NIJE šifriran)

**Naredba**
```bash
kubectl create secret generic db-cred \
  --from-literal=DB_USER=appuser --from-literal=DB_PASSWORD=Horvat
kubectl get secret db-cred -o jsonpath='{.data.DB_USER}' | base64 -d
```
**Objašnjenje:** Secret pohranjuje vrijednosti **base64-kodirane, ne šifrirane** — svatko s pristupom ih dekodira. Base64 je samo kodiranje (čitljivost), ne sigurnost.
**Dokaz:** `base64 -d` vraća `appuser` → dokaz da je samo kodirano.

## 4.32 — Secret iz datoteka (cert + key)

**Naredba**
```bash
kubectl create secret generic certs --from-file=tls.crt --from-file=tls.key
```
**Objašnjenje:** `--from-file` napravi Secret gdje **ime datoteke postaje ključ** (`tls.crt`, `tls.key`), a sadržaj datoteke vrijednost. Korisno za certifikate koje ne želiš tipkati ručno.
**Dokaz**
```bash
kubectl get secret certs -o jsonpath='{.data}'     # ključevi tls.crt, tls.key
```

## 4.33 — TLS-tipizirani Secret iz cert/key para

**Naredba**
```bash
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
```
**Objašnjenje:** Tip `kubernetes.io/tls` je poseban Secret za TLS; konzumiraju ga npr. Ingress ili HTTPS server. Ima fiksne ključeve `tls.crt` i `tls.key`.
**Dokaz**
```bash
kubectl get secret tls-secret     # TYPE kubernetes.io/tls
```

## 4.34 — Secret kao volume vs env (sigurnosni trade-off)

**Naredba**
```yaml
      volumeMounts: [{name: sec, mountPath: /etc/secret, readOnly: true}]
```
**Objašnjenje:** Kao **volume** tajne su u datotekama i mogu se osvježiti bez restarta; kao **env** lakše procure (u logove, child procese, `env` ispis). Volume je sigurniji za osjetljive podatke.
**Dokaz:** usporedi `kubectl exec <p> -- env` (vidi tajnu) vs datoteka s `0400` dozvolama.

## 4.35 — envFrom secretRef: učitaj sve ključeve Secreta kao env

**Naredba**
```yaml
      envFrom:
        - secretRef: {name: db-cred}
```
**Objašnjenje:** `envFrom + secretRef` ubaci **svaki ključ** Secreta kao env varijablu istog imena odjednom, bez nabrajanja pojedinačno. Brže kad trebaš sve vrijednosti.
**Dokaz**
```bash
kubectl exec <p> -- env | grep DB_     # DB_USER, DB_PASSWORD
```

## 4.36 — docker-registry Secret + imagePullSecrets

**Naredba**
```bash
kubectl create secret docker-registry reg \
  --docker-server=docker.io --docker-username=<U> --docker-password=<P>
```
```yaml
      imagePullSecrets:
        - name: reg
```
**Objašnjenje:** Za povlačenje iz **privatnog** registra (ili za zaobilaženje DockerHub limita) k8s treba kredencijale; daješ mu ih preko `imagePullSecrets` koji pokazuje na docker-registry Secret.
**Dokaz**
```bash
kubectl describe pod <p>     # Successfully pulled (nema ImagePullBackOff)
```

## 4.37 — Secret s više ključeva, montiraj samo jedan

**Naredba**
```yaml
  volumes:
    - name: sec
      secret:
        secretName: multi
        items:
          - {key: only, path: only.txt}
```
**Objašnjenje:** Preko `items` biraš **točno koje ključeve** montiraš, ostali se ignoriraju. Tako Pod dobije samo tajnu koja mu treba, ne sve.
**Dokaz**
```bash
kubectl exec <p> -- ls /etc/secret     # samo only.txt
```

## 4.38 — ClusterIP Service + razlučivanje DNS-om iz drugog Poda

**Naredba**
```bash
kubectl expose deployment web --name=web-svc --port=80 --target-port=80
kubectl run tmp --rm -it --image=busybox -- sh
#  nslookup web-svc
#  wget -qO- web-svc
```
**Objašnjenje:** ClusterIP daje stabilnu internu adresu i DNS ime; drugi Pod ga doseže po imenu jer cluster DNS razrješava `web-svc` na njegov ClusterIP. FQDN je `web-svc.default.svc.cluster.local`.
**Dokaz:** `nslookup` vrati ClusterIP, `wget` ispiše nginx stranicu.

## 4.39 — NodePort Service + pristup preko minikube

**Naredba**
```bash
kubectl expose deployment web --name=np --type=NodePort --port=80
minikube service np --url
```
**Objašnjenje:** NodePort otvori port (30000–32767) na svakom čvoru, pa servisu pristupiš izvana preko `IP_čvora:nodePort`. `minikube service --url` ti da gotov URL.
**Dokaz**
```bash
curl $(minikube service np --url)     # nginx stranica
```

## 4.40 — LoadBalancer Service (pending na minikube → tunnel)

**Naredba**
```bash
kubectl expose deployment web --name=lb --type=LoadBalancer --port=80
minikube tunnel     # u drugom terminalu
```
**Objašnjenje:** LoadBalancer traži vanjski LB od cloud providera; minikube ga nema pa EXTERNAL-IP ostaje `<pending>`. `minikube tunnel` simulira LB i dodijeli pristupačnu adresu.
**Dokaz**
```bash
kubectl get svc lb     # nakon tunnela EXTERNAL-IP dobiven
```

## 4.41 — Headless Service za StatefulSet (A-zapis po Podu)

**Naredba** — `clusterIP: None` (vidi 4.16).
**Objašnjenje:** Headless Service ne daje jednu virtualnu IP nego **DNS A-zapis za svaki Pod** (`redis-0.redis...`). Tako se obratiš točno određenoj replici, što StatefulSet treba.
**Dokaz**
```bash
kubectl run tmp --rm -it --image=busybox -- nslookup redis     # više IP-eva, po podu
```

## 4.42 — Endpoints/EndpointSlice (puni se iz selektora)

**Naredba**
```bash
kubectl get endpoints web-svc
```
**Objašnjenje:** Endpoints su lista IP-eva Podova koje Service "pokriva"; k8s ih automatski puni iz **selektora** Servicea. Ako je prazno, selektor ne hvata nijedan Pod (čest uzrok "Service ne radi").
**Dokaz:** ispis pokazuje IP:port svih ciljanih Podova.

## 4.43 — Cross-namespace pristup (FQDN radi, kratko ime ne)

**Naredba**
```bash
kubectl run tmp -n drugi --rm -it --image=busybox -- \
  wget -qO- web-svc.default.svc.cluster.local
```
**Objašnjenje:** Iz drugog namespacea kratko ime (`web-svc`) se ne razrješava jer DNS pretpostavlja lokalni namespace; treba **puni FQDN** s `<ns>.svc.cluster.local`. Pokazuje da su imena namespace-scoped.
**Dokaz:** FQDN dohvati stranicu, kratko ime padne na DNS-u.

## 4.44 — port-forward na Service vs na Pod

**Naredba**
```bash
kubectl port-forward svc/web-svc 8080:80     # na Service (preko load balancinga)
kubectl port-forward pod/<ime> 8080:80       # na konkretni Pod
```
**Objašnjenje:** port-forward privremeno preusmjeri lokalni port u cluster (za testiranje, bez izlaganja vani). Na Service ide kroz balansiranje, na Pod direktno na taj jedan Pod.
**Dokaz**
```bash
curl localhost:8080     # dohvaća app lokalno
```

## 4.45 — Razlika port / targetPort / nodePort

**Objašnjenje:**
- **port** — port na kojem Service sluša unutar clustera.
- **targetPort** — port na podu/kontejneru kamo Service prosljeđuje.
- **nodePort** — vanjski port na čvoru (samo NodePort tip).

Tok: `host:nodePort → Service:port → Pod:targetPort`.
**Dokaz**
```bash
kubectl get svc <svc> -o yaml | grep -E "port|targetPort|nodePort"
```

## 4.46 — Provjeri ClusterIP iz throwaway debug poda

**Naredba**
```bash
kubectl run tmp --rm -it --image=busybox -- sh -c "wget -qO- web-svc; nc -zv web-svc 80"
```
**Objašnjenje:** Privremeni Pod testira povezanost **iznutra iz clustera** — onako kako bi je vidio drugi servis. `wget`/`nc` potvrde da Service odgovara na očekivanom portu.
**Dokaz:** `wget` vraća sadržaj, `nc -zv` javlja "open".

## 4.47 — Dva Deploymenta, jedan Service (zajednička labela → balansira oba)

**Naredba**
```bash
kubectl create deployment a --image=nginx && kubectl label deployment a tier=shared --overwrite
kubectl create deployment b --image=nginx && kubectl label deployment b tier=shared --overwrite
# Service selector: tier: shared
```
**Objašnjenje:** Service balansira preko **svih** Podova koje hvata selektor; ako oba Deploymenta dijele labelu `tier=shared`, jedan Service ih oba pokriva. Zahtjevi padaju na oba.
**Dokaz**
```bash
kubectl get endpoints <svc>     # IP-evi iz oba deploymenta
```

## 4.48 — Dijagnoza zašto Pod ne doseže Service (redoslijed)

**Naredba**
```bash
kubectl get endpoints <svc>      # 1. ima li endpointa?
kubectl describe svc <svc>       # 2. selektor odgovara labelama?
kubectl get pods --show-labels   # 3. readiness OK?
```
**Objašnjenje:** Provjeravaš redom: **DNS → endpoints → selektor/labele → readiness → port**. Prazni endpoints = selektor ne hvata Podove; Pod not-ready = izbačen iz Servicea.
**Dokaz:** lociraš slomljenu kariku u lancu.

## 4.49 — Expose NodePort i potvrdi da radi

**Naredba**
```bash
kubectl expose deployment web --type=NodePort --port=80
curl $(minikube service web --url)
```
**Objašnjenje:** NodePort izloži servis izvana; `curl` na minikube URL potvrdi da promet stiže od hosta do Poda.
**Dokaz:** `curl` vraća odgovor aplikacije.

## 4.50 — HTTP livenessProbe na nginx + provjera

**Naredba**
```yaml
      livenessProbe:
        httpGet: {path: /, port: 80}
        initialDelaySeconds: 5
        periodSeconds: 10
```
**Objašnjenje:** liveness provjerava je li kontejner **živ**; ako proba padne, k8s ga **restarta**. Sprječava "zombi" stanje gdje proces radi ali ne odgovara.
**Dokaz**
```bash
kubectl describe pod <p>     # Liveness: http-get http://:80/ ...
```

## 4.51 — tcpSocket readinessProbe na redis (6379)

**Naredba**
```yaml
      readinessProbe:
        tcpSocket: {port: 6379}
        periodSeconds: 5
```
**Objašnjenje:** readiness provjerava je li kontejner **spreman primati promet**; ako padne, Pod se **izbaci iz Servicea** (ne restarta). tcpSocket samo provjeri otvara li se port.
**Dokaz**
```bash
kubectl describe pod <p>     # Readiness: tcp-socket :6379
```

## 4.52 — startupProbe (budžet za ~2 min boot)

**Naredba**
```yaml
      startupProbe:
        httpGet: {path: /, port: 80}
        failureThreshold: 24
        periodSeconds: 5
```
**Objašnjenje:** startupProbe daje sporim aplikacijama vremena da se podignu prije nego liveness počne. Budžet = `failureThreshold × periodSeconds` = 24 × 5s = **120s**. Dok startup ne uspije, liveness/readiness miruju.
**Dokaz**
```bash
kubectl describe pod <p>     # Startup: ... #failures 24
```

## 4.53 — Zadano ponašanje kad nema proba

**Objašnjenje:** Bez ijedne probe k8s smatra Pod zdravim **čim proces krene** — ne provjerava odgovara li aplikacija stvarno. Zato app može biti "Running" i u Serviceu iako ne poslužuje promet.
**Dokaz:** Pod bez proba je odmah Ready; padne li app interno, k8s to ne primijeti.

---
---

# LO5 — Troubleshooting (43 pitanja, pun tretman)
> Svaki zadatak: **pokvareni kod → dijagnoza → ispravak → dokaz → objašnjenje.**
> Format predaje: označi `Lo5_<broj>`, priloži screenshot greške (dokaz da problem postoji) i screenshot nakon ispravka.

---

## DIO 1 — podman run greške (5.1–5.8)

### 5.1 — `-p 80:8080`: stranica nedostupna na očekivanom portu
**Pokvareno**
```bash
podman run -d --name web -p 80:8080 docker.io/library/nginx
```
**Dijagnoza**
```bash
podman ps                    # PORTS: 0.0.0.0:80->8080/tcp  (mapira na krivi kontejnerski port)
curl http://localhost:80     # ne odgovara / connection refused
podman exec web sh -c "ss -tlnp | grep 80"   # nginx zapravo sluša na 80, ne 8080
```
**Ispravak**
```bash
podman rm -f web
podman run -d --name web -p 8080:80 docker.io/library/nginx
```
**Dokaz**
```bash
podman ps                    # PORTS: 0.0.0.0:8080->80/tcp
curl http://localhost:8080   # nginx welcome stranica
```
**Objašnjenje:** `-p` koristi format `HOST:KONTEJNER`. nginx unutar kontejnera sluša na **80**, a originalna naredba je kao kontejnerski port navela 8080, pa promet nije išao na nginx. Lijevi broj je port na hostu, desni port na kojem app stvarno sluša u kontejneru.

### 5.2 — `-e MYSQL_ROOT_PASSWORD` bez vrijednosti: kontejner padne pri initu
**Pokvareno**
```bash
podman run -d --name db -e MYSQL_ROOT_PASSWORD docker.io/library/mysql
```
**Dijagnoza**
```bash
podman ps -a            # db: Exited (1)
podman logs db          # "Database is uninitialized and password option is not specified"
```
**Ispravak**
```bash
podman rm -f db
podman run -d --name db -e MYSQL_ROOT_PASSWORD=tajna123 docker.io/library/mysql
```
**Dokaz**
```bash
podman ps               # db ostaje Up
podman logs db          # "ready for connections"
```
**Objašnjenje:** `-e MYSQL_ROOT_PASSWORD` bez `=vrijednost` postavlja praznu varijablu, a MySQL init odbija startati bez root lozinke (ili `MYSQL_ALLOW_EMPTY_PASSWORD`). Zato kontejner izađe tijekom inicijalizacije.

### 5.3 — `--network host` + `-p`: -p se ignorira
**Pokvareno**
```bash
podman run -d --name app --network host -p 8080:80 docker.io/library/nginx
```
**Dijagnoza**
```bash
podman run ...          # podman čak ispiše upozorenje da je -p ignoriran uz host mrežu
curl http://localhost:80    # nginx je zapravo na host:80, NE na 8080
```
**Ispravak** (izaberi JEDNU strategiju)
```bash
podman rm -f app
podman run -d --name app -p 8080:80 docker.io/library/nginx        # bridge + publish
# ILI
podman run -d --name app --network host docker.io/library/nginx    # host mreža, bez -p
```
**Dokaz**
```bash
# host varijanta: nginx odmah na portu 80 hosta
curl http://localhost:80
```
**Objašnjenje:** S `--network host` kontejner dijeli mrežni namespace hosta, pa su mu portovi već "na hostu" — `-p` (koji radi NAT/port-mapping za bridge mrežu) nema učinka. Koristiš ili bridge + `-p`, ili host mrežu bez `-p`.

### 5.4 — busybox odmah ode u Exited (0)
**Pokvareno**
```bash
podman run -d --name c1 docker.io/library/busybox
```
**Dijagnoza**
```bash
podman ps               # c1 se NE vidi (gleda samo žive)
podman ps -a            # c1: Exited (0)  ← bitno je -a !
podman logs c1          # prazno
```
**Ispravak**
```bash
podman rm -f c1
podman run -d --name c1 docker.io/library/busybox sleep infinity
```
**Dokaz**
```bash
podman ps               # c1: Up
```
**Objašnjenje:** Kontejner živi koliko i njegov glavni proces (PID 1). busybox bez zadane naredbe nema dugotrajan proces, pa odmah izađe s kodom 0. `sleep infinity` daje proces koji se nikad ne završi, pa kontejner ostaje živ. (Trik: `podman ps -a` jer obični `ps` ne pokazuje zaustavljene.)

### 5.5 — `--rm -d ... echo`: nema logova
**Pokvareno**
```bash
podman run --rm -d --name job docker.io/library/alpine echo hello
podman logs job          # greška: no such container
```
**Dijagnoza**
```bash
podman ps -a             # job se NE vidi — već je obrisan
```
**Ispravak**
```bash
podman run -d --name job docker.io/library/alpine echo hello
podman logs job          # hello
```
**Dokaz**
```bash
podman logs job          # "hello"
```
**Objašnjenje:** `echo` odmah završi, a `--rm` automatski obriše kontejner čim glavni proces stane — pa kad zatražiš logove, kontejnera više nema. Za kratke naredbe čiji izlaz želiš vidjeti poslije, izostavi `--rm`.

### 5.6 — `--memory 8m` za mysql: nikad ne postane healthy
**Pokvareno**
```bash
podman run -d -p 8080:80 --memory 8m docker.io/library/mysql
```
**Dijagnoza**
```bash
podman ps -a             # restarta se / Exited
podman logs <c>          # "Out of memory" / init prekinut
```
**Ispravak**
```bash
podman run -d --memory 512m -e MYSQL_ROOT_PASSWORD=tajna docker.io/library/mysql
```
**Dokaz**
```bash
podman ps                # Up; logs -> "ready for connections"
```
**Objašnjenje:** 8 MB je daleko premalo za MySQL — kernel ubije proces (OOM) tijekom inicijalizacije pa baza nikad ne postane spremna. Memory limit mora biti realan za workload (MySQL ~512m+).

### 5.7 — DNS po imenu pada na defaultnoj mreži
**Pokvareno**
```bash
podman run -d --name a docker.io/library/alpine sleep 1d
podman run -d --name b docker.io/library/alpine sleep 1d
podman exec a ping -c1 b     # bad address 'b'
```
**Dijagnoza**
```bash
podman exec a ping -c1 b     # ime se ne razrješava (default mreža nema DNS)
```
**Ispravak**
```bash
podman network create net
podman rm -f a b
podman run -d --name a --network net docker.io/library/alpine sleep 1d
podman run -d --name b --network net docker.io/library/alpine sleep 1d
podman exec a ping -c1 b
```
**Dokaz**
```bash
podman exec a getent hosts b     # ime "b" -> IP
podman exec a ping -c1 b         # odgovara
```
**Objašnjenje:** Default (podman) mreža nema ugrađeni DNS, pa se kontejneri ne vide po imenu. Na **user-defined** mreži radi DNS i razlučivanje po imenu kontejnera proradi.

### 5.8 — bind mount: datoteke nevidljive / SELinux odbija
**Pokvareno**
```bash
podman run -d --name web -v ./html:/usr/share/nginx/html docker.io/library/nginx
```
**Dijagnoza**
```bash
podman exec web ls /usr/share/nginx/html     # prazno ili Permission denied
podman logs web                              # "Permission denied" (SELinux)
```
**Ispravak**
```bash
podman rm -f web
podman run -d --name web -v ./html:/usr/share/nginx/html:Z docker.io/library/nginx
```
**Dokaz**
```bash
podman exec web ls /usr/share/nginx/html     # datoteke s hosta vidljive
```
**Objašnjenje:** Na Red Hatu SELinux blokira pristup kontejnera host-direktoriju bez ispravne oznake. `:Z` kaže podmanu da preoznači direktorij (privatni label za taj kontejner). (Ako `:Z` na root-owned mapi padne s `lsetxattr ... operation not permitted` → `sudo chcon -Rt container_file_t <dir>` pa mount bez `:Z`.)

---

## DIO 2 — Containerfile greške (5.9–5.18)
> Postupak za svaki: napravi datoteku (`nano Containerfile`), `podman build -t test .` (padne = dokaz), popravi, build opet (prođe = dokaz).

### 5.10 — `apt-get install` bez `update`
**Pokvareno**
```dockerfile
FROM debian:12
RUN apt-get install -y nginx
```
**Dijagnoza:** `podman build -t test .` → `E: Unable to locate package nginx`.
**Ispravak**
```dockerfile
FROM debian:12
RUN apt-get update && apt-get install -y nginx && rm -rf /var/lib/apt/lists/*
```
**Dokaz:** build prođe → `Successfully tagged ...`.
**Objašnjenje:** Bez `apt-get update` lokalna lista paketa je prazna, pa install ne nađe nginx. Cleanup (`rm -rf /var/lib/apt/lists/*`) je u istom RUN-u da ne napuhne sliku.
> Napomena iz VM-a: ako `deb.debian.org` daje "Network is unreachable" (IPv6 problem), koristi `FROM ubuntu:24.04` (logika je ista) ili dodaj `RUN echo 'Acquire::ForceIPv4 "true";' > /etc/apt/apt.conf.d/99force-ipv4 && ...`.

### 5.11 — `CMD python app.py` (shell forma): ne hvata signale
**Pokvareno**
```dockerfile
FROM python:3.11
COPY app.py /app/app.py
WORKDIR /app
CMD python app.py
```
**Dijagnoza:** `podman stop <c>` traje ~10s pa kontejner biva ubijen silom (SIGKILL), umjesto da uredno stane.
**Ispravak**
```dockerfile
CMD ["python", "app.py"]
```
**Dokaz:** `podman stop <c>` stane gotovo odmah.
**Objašnjenje:** Shell forma (`CMD python app.py`) pokreće app preko `/bin/sh -c`, pa je shell PID 1 i signali (SIGTERM kod stopa) ne stignu do Pythona. Exec forma (JSON niz) čini Python PID 1, pa uredno prima signale.

### 5.12 — `COPY .` prije `npm install`: cache se uvijek poništi
**Pokvareno**
```dockerfile
FROM node:20
COPY . /app
WORKDIR /app
RUN npm install
```
**Dijagnoza:** svaka sitna promjena koda pokrene puni `npm install` (spor build).
**Ispravak**
```dockerfile
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
```
**Dokaz:** promijeni neki `.js` i rebuildaj → `npm install` layer je `Using cache`.
**Objašnjenje:** Layer cache se poništi čim se promijeni išta kopirano prije njega. Kad prvo kopiraš samo `package*.json`, sloj s `npm install` ostaje keširan dok se ovisnosti ne promijene; kod kopiraš tek poslije.

### 5.13 — `ENV PATH=/app/bin`: curl/sh nestanu
**Pokvareno**
```dockerfile
FROM alpine:3.20
ENV PATH=/app/bin
RUN apk add --no-cache curl
```
**Dijagnoza:** `RUN` padne s `apk: not found` (jer ni `apk` više nije u PATH-u).
**Ispravak**
```dockerfile
ENV PATH=/app/bin:$PATH
```
**Dokaz:** build prođe; `podman run --rm test which curl` nađe curl.
**Objašnjenje:** Postavljanjem samo `/app/bin` izbrisao si originalni PATH (`/usr/bin`, `/bin`...), pa sistemski alati nisu nađeni. Treba **dopisati**, ne zamijeniti: `$PATH` zadrži postojeće putanje.

### 5.14/5.15 — `EXPOSE 8080` + app sluša na 3000: ništa ne odgovara
**Pokvareno**
```dockerfile
FROM ubuntu:24.04
EXPOSE 8080
CMD ["python3", "-m", "http.server", "3000"]
```
**Dijagnoza:** `podman run -d -p 8080:8080 test` → `curl localhost:8080` ne odgovara (app je na 3000).
**Ispravak**
```dockerfile
EXPOSE 3000
CMD ["python3", "-m", "http.server", "3000"]
```
```bash
podman run -d -p 8080:3000 test
```
**Dokaz:** `curl localhost:8080` vraća file listing.
**Objašnjenje:** `EXPOSE` je samo **dokumentacija** — ne objavljuje port niti mijenja na čemu app sluša. Stvarni port određuje aplikacija (3000) i `-p` mapping; nesklad EXPOSE/app/-p znači da promet ide na port gdje nitko ne sluša.

### 5.16 — slika ogromna (cache paketa)
**Pokvareno**
```dockerfile
FROM debian:12
RUN apt-get update && apt-get install -y build-essential
```
**Dijagnoza:** `podman images` → slika stotinama MB veća nego treba.
**Ispravak**
```dockerfile
RUN apt-get update && apt-get install -y build-essential && rm -rf /var/lib/apt/lists/*
```
**Dokaz:** `podman images` → manja slika.
**Objašnjenje:** Cleanup mora biti u **istom RUN-u** jer svaki RUN je novi layer; ako počistiš u kasnijem RUN-u, datoteke i dalje žive u ranijem layeru i ne smanjuju veličinu.

### 5.17 — `USER appuser` prije COPY/pip: permission denied
**Pokvareno**
```dockerfile
FROM python:3.11
USER appuser
COPY requirements.txt /app/requirements.txt
RUN pip install -r /app/requirements.txt
```
**Dijagnoza:** build padne s `Permission denied` na `/app`.
**Ispravak**
```dockerfile
FROM python:3.11
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
RUN useradd -m appuser
USER appuser
```
**Dokaz:** build prođe; `podman run --rm test whoami` → `appuser`.
**Objašnjenje:** Nakon `USER appuser` taj korisnik nema prava pisati u sistemske putanje ni instalirati pakete. Kopiranje i instalaciju radiš kao root, a `USER` prebaciš **na kraj** (ili koristiš `COPY --chown`).

### 5.18 — golang slika ~1 GB
**Pokvareno**
```dockerfile
FROM golang:1.22
COPY . /src
WORKDIR /src
RUN go build -o app .
CMD ["/src/app"]
```
**Dijagnoza:** `podman images` → ~1 GB (sadrži cijeli Go toolchain).
**Ispravak (multi-stage)**
```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN go build -o app .

FROM alpine:3.20
COPY --from=build /src/app /app
CMD ["/app"]
```
**Dokaz:** `podman images` → finalna slika par MB.
**Objašnjenje:** Finalna slika ne treba compiler ni izvorni kod — samo gotov binary. Multi-stage build u prvoj fazi kompajlira, a u drugu (minimalnu) fazu kopira samo rezultat.

---

## DIO 3 — Kubernetes dijagnoza (5.19–5.43)
> Prije svega: `minikube status` (ako `i/o timeout` → `minikube start`).
> Tri glavna alata: `kubectl describe pod <p>` (Events na dnu), `kubectl logs <p> --previous` (zadnji crash), `kubectl get events --sort-by=.lastTimestamp`.

### 5.19 — Pod Pending
**Dijagnoza**
```bash
kubectl describe pod <p>     # Events: "Insufficient cpu/memory" / "didn't match nodeSelector" / "pod has unbound PVC"
```
**Ispravak:** ovisno o uzroku — smanji `resources.requests`, ukloni/ispravi `nodeSelector`, ili stvori PVC.
**Dokaz:** `kubectl get pod <p>` → Running.
**Objašnjenje:** Pending znači da scheduler ne može smjestiti Pod. Najčešće: traži više resursa nego ijedan čvor ima, traži labelu čvora koja ne postoji, ili čeka volumen koji ne postoji.

### 5.20 — Deployment ne deploya zbog pomaknutih razmaka
**Dijagnoza**
```bash
kubectl apply -f deploy.yaml     # error: ... mapping values / did not find expected key
kubectl apply --dry-run=server --validate=true -f deploy.yaml   # pokaže točan redak
```
**Ispravak:** popravi uvlaku (YAML koristi **razmake, nikad tabove**); poravnaj ključeve na istu razinu.
**Dokaz:** `kubectl apply -f deploy.yaml` → created/configured.
**Objašnjenje:** YAML je osjetljiv na indentaciju — i jedan višak/manjak razmaka mijenja strukturu i ruši parsiranje. `--dry-run=server` provjeri bez primjene.

### 5.21 — ImagePullBackOff
**Pokvareno (primjer)**
```yaml
    image: nginx:1.25-alpinee     # typo u tagu
```
**Dijagnoza**
```bash
kubectl describe pod <p>     # Events: "Failed to pull image ... not found / manifest unknown"
```
**Ispravak:** ispravi ime/tag; za privatni registry dodaj `imagePullSecrets`; za DockerHub limit kreiraj docker-registry Secret.
**Dokaz:** `kubectl get pod <p>` → Running.
**Objašnjenje:** k8s ne može povući sliku — krivi tag/ime, privatni registry bez kredencijala, ili rate-limit. `describe` u Events pokazuje točan razlog.

### 5.22 — CrashLoopBackOff
**Dijagnoza**
```bash
kubectl describe pod <p>         # Last State: Terminated, Exit Code
kubectl logs <p> --previous      # izlaz prije zadnjeg pada (zašto se srušila)
```
**Ispravak:** ovisi o logu — npr. fali env varijabla, kriva komanda, app baca grešku → popravi uzrok.
**Dokaz:** `kubectl get pod <p>` → Running, RESTARTS prestaje rasti.
**Objašnjenje:** App se pokrene, sruši, k8s je restarta — i tako u krug, sve s rastućim backoff razmakom. `--previous` čita log **mrtve** instance jer je trenutna možda još ne postoji.

### 5.23 — OOMKilled
**Dijagnoza**
```bash
kubectl get pod <p> -o yaml | grep -A3 lastState   # reason: OOMKilled
kubectl describe pod <p>                            # Last State: Terminated, Reason: OOMKilled
```
**Ispravak:** digni `resources.limits.memory`, ili popravi app da troši manje.
**Dokaz:** Pod ostaje Running bez OOM restarta.
**Objašnjenje:** Kontejner je prešao svoj memory **limit**, pa ga kernel ubije (Out Of Memory). Ili je limit prenizak za workload, ili app curi memoriju.

### 5.24 — Pod nikad ne postane Ready
**Dijagnoza**
```bash
kubectl describe pod <p>     # Readiness probe failed: ...
```
**Ispravak:** popravi readiness probu (točan port/putanja) ili app da odgovara na nju.
**Dokaz:** `kubectl get pod <p>` → READY 1/1.
**Objašnjenje:** Readiness proba pada, pa k8s drži Pod "not ready" i **izbacuje ga iz Servicea** (ne restarta ga). Često: proba gađa krivi port/putanju ili app još nije spremna.

### 5.25 — Service vraća prazno
**Dijagnoza**
```bash
kubectl get endpoints <svc>          # prazno = nijedan Pod nije pokriven
kubectl describe svc <svc>           # vidi Selector
kubectl get pods --show-labels       # usporedi labele s selektorom
```
**Ispravak:** uskladi `spec.selector` Servicea s labelama Podova.
**Dokaz:** `kubectl get endpoints <svc>` → popunjeno IP-evima.
**Objašnjenje:** Service rutira prema Podovima koje hvata **selektor**; ako labele Podova ne odgovaraju selektoru, endpoints su prazni i Service nema kamo slati promet.

### 5.26 — `selector.matchLabels` ≠ `template.labels`
**Pokvareno**
```yaml
spec:
  selector:
    matchLabels: {app: web}
  template:
    metadata:
      labels: {app: webserver}    # ne odgovara selektoru
```
**Dijagnoza**
```bash
kubectl apply -f deploy.yaml     # error: selector does not match template labels
```
**Ispravak:** poravnaj labele (`app: web` na oba mjesta).
**Dokaz:** apply prođe; `kubectl get deploy` → READY.
**Objašnjenje:** Deployment mora moći "uhvatiti" svoje Podove preko selektora; ako se selektor i labele predloška razilaze, validacija odbije manifest.

### 5.27 — `requests` veći od kapaciteta čvora
**Dijagnoza**
```bash
kubectl describe pod <p>     # "Insufficient cpu" / "Insufficient memory"
kubectl describe node        # usporedi Allocatable s traženim
```
**Ispravak:** smanji `resources.requests` na realnu vrijednost.
**Dokaz:** Pod prijeđe iz Pending u Running.
**Objašnjenje:** Ako Pod traži više CPU/RAM-a nego ijedan čvor ima slobodno, scheduler ga ne može smjestiti i ostaje Pending.

### 5.28 — Pod montira nepostojeći PVC
**Dijagnoza**
```bash
kubectl describe pod <p>     # "persistentvolumeclaim ... not found" / FailedMount
```
**Ispravak:** stvori PVC prije Poda
```bash
kubectl apply -f pvc.yaml
```
**Dokaz:** Pod prijeđe u Running.
**Objašnjenje:** Pod čeka volumen koji ne postoji, pa ostaje Pending uz FailedMount. PVC mora postojati (i biti Bound) da bi se Pod pokrenuo.

### 5.29 — ubuntu bez komande → Completed/CrashLoop
**Pokvareno**
```yaml
apiVersion: v1
kind: Pod
metadata: {name: tool}
spec:
  containers:
    - name: tool
      image: docker.io/library/ubuntu:24.04
```
**Dijagnoza**
```bash
kubectl describe pod tool        # Last State: Terminated, Reason: Completed, Exit Code: 0
kubectl logs tool --previous     # prazno — nema greške, samo izađe
```
**Ispravak**
```yaml
    - name: tool
      image: docker.io/library/ubuntu:24.04
      command: ["sleep", "infinity"]
```
```bash
kubectl delete pod tool && kubectl apply -f tool.yaml
```
**Dokaz**
```bash
kubectl get pod tool     # Running, RESTARTS 0
```
**Objašnjenje:** ubuntu nema dugotrajan glavni proces — kontejner se pokrene, nema što raditi i izađe (0). Uz default `restartPolicy: Always` k8s ga stalno diže → CrashLoopBackOff. `sleep infinity` daje proces koji ne završava. **(Ovo je ista logika kao podman busybox #5.4 — kontejner živi koliko PID 1.)**

### 5.30 — Triage svega što se dogodilo Podu
**Dijagnoza**
```bash
kubectl get events --sort-by=.lastTimestamp        # kronološki svi događaji
kubectl get events --field-selector involvedObject.name=<p>
```
**Objašnjenje:** Kad ne znaš odakle početi, kronološki popis Eventa pokaže slijed (scheduling → pull → start → crash) i točku gdje je pošlo po zlu.

### 5.31 — Debug DNS-a iznutra Poda
**Dijagnoza**
```bash
kubectl exec -it <p> -- sh
  nslookup <svc>
  cat /etc/resolv.conf
```
**Ispravak:** ako kratko ime ne radi, probaj FQDN `<svc>.<ns>.svc.cluster.local`; provjeri da `resolv.conf` pokazuje na cluster DNS.
**Dokaz:** `nslookup` vrati ClusterIP servisa.
**Objašnjenje:** Problemi povezivosti su često DNS, ne mreža. `resolv.conf` pokazuje koji DNS i koje `search` domene Pod koristi.

### 5.32 — Distroless / no-shell kontejner
**Dijagnoza/alat**
```bash
kubectl debug -it <p> --image=busybox --target=<container>
```
**Objašnjenje:** Minimalne (distroless) slike nemaju shell pa `kubectl exec ... sh` ne radi. `kubectl debug` priloži privremeni kontejner (busybox) u isti Pod, koji dijeli namespace s ciljanim, pa imaš alate za dijagnozu.

### 5.33 — Test povezanosti iz throwaway poda
**Dijagnoza/alat**
```bash
kubectl run tmp --rm -it --image=busybox -- sh
  wget -qO- <svc>
  nc -zv <svc> <port>
```
**Objašnjenje:** Privremeni Pod testira doseže li servis **iznutra iz clustera**, neovisno o tome je li izložen vani. `--rm` ga obriše na izlazu (`exit`). Ako "already exists", prvo `kubectl delete pod tmp`.

### 5.34 — Čvor NotReady
**Dijagnoza**
```bash
kubectl get nodes               # NotReady
kubectl describe node <node>    # Conditions: DiskPressure / MemoryPressure / kubelet
minikube logs
```
**Ispravak:** ovisi — restart kubeleta, oslobađanje diska, provjera CNI (mrežni dodatak).
**Objašnjenje:** NotReady znači da kubelet na čvoru ne javlja zdravlje — uzroci su disk/memory pressure, pao kubelet ili mrežni dodatak (CNI).

### 5.35 — Pod zaglavljen u Terminating
**Dijagnoza**
```bash
kubectl get pod <p>          # Terminating dugo
kubectl describe pod <p>     # finalizers, grace period
```
**Ispravak**
```bash
kubectl delete pod <p> --grace-period=0 --force
```
**Dokaz:** Pod nestane iz `kubectl get pods`.
**Objašnjenje:** Finalizeri ili predug grace period drže Pod u Terminating. Force-delete ga makne iz API-ja, ali **rizik** je da pripadni resursi (volumen, vanjski zapis) možda nisu uredno počišćeni.

### 5.36 — Krivi apiVersion/kind par
**Pokvareno**
```yaml
apiVersion: apps/v1
kind: Pod
```
**Dijagnoza**
```bash
kubectl apply -f f.yaml      # error: no matches for kind "Pod" in version "apps/v1"
```
**Ispravak:** `Pod` je u `v1` (ne `apps/v1`); `Deployment/ReplicaSet/StatefulSet/DaemonSet` su u `apps/v1`; `Job/CronJob` u `batch/v1`.
**Dokaz:** apply prođe.
**Objašnjenje:** Svaki kind pripada točno određenoj API grupi/verziji; pogrešan par k8s ne prepoznaje.

### 5.37 — CreateContainerConfigError (fali ključ)
**Dijagnoza**
```bash
kubectl describe pod <p>     # "couldn't find key X in ConfigMap/Secret"
```
**Ispravak:** dodaj ključ u ConfigMap/Secret, ili ispravi `configMapKeyRef`/`secretKeyRef` da pokazuje na postojeći ključ.
**Dokaz:** Pod prijeđe u Running.
**Objašnjenje:** Pod referencira ključ iz ConfigMapa/Secreta koji ne postoji, pa k8s ne može sastaviti konfiguraciju kontejnera.

### 5.38 — Secret iz drugog namespacea
**Dijagnoza**
```bash
kubectl get secret <s> -n <ns_poda>     # nije tu (Secret je u drugom ns)
kubectl describe pod <p>                # Secret not found
```
**Ispravak:** kopiraj Secret u isti namespace kao Pod
```bash
kubectl get secret <s> -n <drugi_ns> -o yaml \
  | sed 's/namespace: <drugi_ns>/namespace: <ns_poda>/' | kubectl apply -f -
```
**Dokaz:** Pod montira Secret i prijeđe u Running.
**Objašnjenje:** Secreti su **namespace-scoped** — Pod vidi samo Secrete u vlastitom namespaceu. Nema "posuđivanja" preko namespacea.

### 5.39 — ProgressDeadlineExceeded
**Dijagnoza**
```bash
kubectl rollout status deployment/<d>    # "exceeded its progress deadline"
kubectl describe deployment <d>          # razlog (ImagePullBackOff / probe / Pending)
```
**Ispravak:** riješi temeljni uzrok (ispravi sliku/probu/resurse), pa rollout nastavi.
**Dokaz:** `kubectl rollout status` → "successfully rolled out".
**Objašnjenje:** Deployment nije uspio dovršiti rollout u zadanom roku (`progressDeadlineSeconds`, default 600s). To je simptom — pravi uzrok je obično ImagePullBackOff, pala proba ili Pending podovi.

### 5.40 — Uhvati typo prije apply
**Dijagnoza/alat**
```bash
kubectl apply --dry-run=server --validate=true -f f.yaml
```
**Objašnjenje:** `--dry-run=server` provjeri manifest na serveru bez stvarne primjene i uhvati greške poput `imagePullpolicy` (treba `imagePullPolicy`) ili krivo ugniježđenih polja prije nego dođu u cluster.

### 5.41 — App ne doseže bazu (Service)
**Dijagnoza (redom)**
```bash
kubectl get pods                 # 1. baza Running?
kubectl get svc <db-svc>         # 2. Service postoji?
kubectl get endpoints <db-svc>   # 3. endpoints popunjeni?
kubectl exec -it <app> -- nslookup <db-svc>   # 4. DNS razrješava?
kubectl describe svc <db-svc>    # 5. točan port/targetPort?
```
**Ispravak:** popravi prvu slomljenu kariku (npr. selektor → endpoints, ili krivi port).
**Objašnjenje:** Povezivost je lanac: Pod radi → Service postoji → endpoints popunjeni → DNS razrješava → ispravan port. Provjeravaš redom dok ne nađeš gdje puca.

### 5.42 — Razlog ponovljenih restarta
**Dijagnoza**
```bash
kubectl get pod <p> -o jsonpath='{.status.containerStatuses[0].restartCount}'; echo
kubectl get pod <p> -o jsonpath='{.status.containerStatuses[0].lastState}'; echo
```
**Objašnjenje:** `restartCount` pokaže koliko je puta restartan, a `lastState` (Terminated + reason: OOMKilled/Error + exitCode) otkriva **zašto** — pa znaš je li problem memorija, greška aplikacije ili nešto treće.

### 5.43 — OpenShift Route vs Ingress (teorijski)
**Objašnjenje:** Oba izlažu HTTP servis vani. **Route** je OpenShift-ugrađen objekt — router radi out-of-the-box, TLS i hostname se postave lako, ali je vezan uz OpenShift. **Ingress** je standardni Kubernetes objekt, prenosiv između distribucija, ali zahtijeva da sam instaliraš i održavaš **ingress controller** (npr. nginx-ingress). Ukratko: Route = jednostavnije na OpenShiftu, Ingress = prenosivije i standardno.

---
---

# LO6 — Teorija (23 pitanja)
> Pisani odgovori. Formula: **(1) jasan stav → (2) 2–3 argumenta → (3) 1 protuargument.** Nema naredbi.

**6.1 Mreža k8s vs Docker** — Docker: bridge na jednom hostu, DNS po imenu, `-p` publish. k8s: svaki Pod ima IP (flat network bez NAT-a), Servisi za stabilne adrese, CNI dodaci, NetworkPolicy. k8s složeniji ali skalabilan preko čvorova.

**6.2 Storage k8s vs Podman** — k8s: PVC + StorageClass + CSI → dinamičko provisioniranje, fleksibilno za stateful. Podman: vezan uz host volumene. k8s fleksibilniji jer apstrahira pohranu od čvora.

**6.3 Operator pattern vs čisti manifesti** — Operator (CRD + controller) automatizira day-2 operacije (backup, failover) stateful servisa; manifesti to traže ručno. Više kontrole, ali trud da se napiše operator.

**6.4 Plain k8s vs OpenShift S2I/konzola** — k8s (kubectl+manifesti) optimizira za fleksibilnost/kontrolu; OpenShift S2I+konzola za brzinu i developer experience. Svaki za drugu publiku.

**6.5 Migracija OpenShift → k8s** — gubiš Route/SCC/S2I specifičnosti, moraš ih zamijeniti (Ingress, ručni security). Vrijedi ako želiš izbjeći lock-in, ali je trud znatan.

**6.6 CI/CD agenti u k8s vs VM** — u k8s: autoscaling, izolacija, efikasnost; rizik noisy-neighbor (riješi resource limitima). VM: jača izolacija, ali skuplje i statično.

**6.7 k8s + cloud** — integrira se s cloud autoscalerima (cluster autoscaler), dinamičkim PV-ovima, LoadBalancerima. Skalira kapacitet po potražnji.

**6.8 Sigurnost OpenShift vs vanilla k8s** — OpenShift stroži po defaultu (SCC blokira root, integriran registry+scan, mrežne politike). Zato regulirane industrije biraju OpenShift — manje ručnog hardeninga.

**6.9 Krivulja učenja OpenShift vs k8s** — OpenShift apstrakcije (konzola, S2I) pomažu timovima **novima** u kontejnerima, ali skrivaju "ispod haube" pa zbune kad treba dublje.

**6.10 Full k8s vs k3s (mali on-prem)** — k3s lagan (jedan binary, SQLite), isti API; idealan za edge/mali on-prem. Full k8s overkill kad nemaš puno čvorova/složene potrebe.

**6.11 Compose na jednom hostu vs k8s (mali tim)** — Compose jednostavno/jeftino; k8s tek kad treba skaliranje/HA. Prerano uvođenje k8s = velik teret bez koristi.

**6.12 Vendor lock-in** — k8s open source/CNCF → manji lock-in, prenosivo. OpenShift Red Hat ekosustav → podrška ali vezanost. Utječe na dugoročnu fleksibilnost.

**6.13 Препоruka OpenShift za integriranu platformu** — DA ako tvrtka želi vendor-podržano, integrirano (konzola, CI/CD, sigurnost) i ne želi sama sastavljati alate. Minus: cijena, lock-in.

**6.14 Storage k8s vs VM** — k8s: deklarativni PVC, dinamičko provisioniranje; VM: ručno montiranje diskova. k8s automatiziraniji i prenosiviji.

**6.15 Startup, 2 inženjera, brzo/jeftino** — Compose na jednom hostu (ili managed servis): niska cijena, mali overhead. k8s kasnije kad rast to opravda.

**6.16 Docker vs Podman (arhitektura/rootless)** — Docker daemon (root, SPOF); Podman daemonless + rootless (manja napadna površina). Sigurnosno svjestan tim bira Podman.

**6.17 Day-2 self-managed vs managed k8s** — self-managed: kontrola, ali ti radiš upgrade/scaling/backup. Managed: provider preuzima teret, uz cijenu i manju kontrolu.

**6.18 Streaming, spiky promet** — k8s na cloudu + HPA + cluster autoscaler + više regija/CDN. Elastično skalira na nepredvidiv promet.

**6.19 Outgrown Swarm → migrirati na k8s?** — vagaš trud migracije (prepisivanje manifesta, učenje) protiv koristi (skaliranje, ekosustav, podrška). Ako rast i pouzdanost rastu, isplati se.

**6.20 k8s za mikroservise** — DA: orkestracija mnoštva servisa, self-healing, rolling update, bogat ekosustav (Helm, Istio, Prometheus). Minus: složenost; za malo servisa pretjerano.

**6.21 k8s za enterprise** — DA: skalabilnost, golema zajednica, bogatstvo značajki. Minus: operativni teret; treba ekspertiza.

**6.22 OpenShift za mikroservise** — DA: orkestracija + ugrađeni alati + sigurnost + podrška. Minus: cijena, lock-in.

**6.23 OpenShift za enterprise** — DA: skalabilnost + enterprise podrška + compliance. Minus: trošak licence, vezanost uz Red Hat.




6.1 — Kubernetes networking model vs Docker networking

Docker's networking is host-centric and simple: containers attach to bridge networks on a single host, get DNS resolution by container name on user-defined networks, and you publish ports to the host with -p. It works well for a handful of containers on one machine.

Kubernetes networking is distributed and built for many nodes. Every Pod gets its own routable IP in a flat network, so any Pod can reach any other Pod without NAT. Stable access is provided by Services (a virtual IP and DNS name in front of a changing set of Pods), the actual network is implemented by a CNI plugin (Calico, Flannel, etc.), and traffic can be segmented with NetworkPolicy.

The Kubernetes model is more complex because it must stay consistent across a whole cluster of nodes, handle Pods that come and go, and provide service discovery and load balancing automatically.

Counterargument: that power costs simplicity — for a single host with a few containers, Docker's model is easier to reason about and Kubernetes networking is overkill.

6.2 — Kubernetes storage abstraction vs Podman storage

Kubernetes abstracts storage from the node through PersistentVolumeClaims (PVCs), StorageClasses, and the CSI interface, which enables dynamic provisioning — a workload asks for storage and the cluster creates it automatically from whatever backend is configured. This makes it very flexible for stateful workloads that may move between nodes.

Podman storage is tied much more closely to the host: named volumes and bind mounts live on that one machine, with no built-in concept of dynamic provisioning across a cluster.

Kubernetes is more flexible for stateful workloads precisely because the storage request is decoupled from any specific node or disk.

Counterargument: the abstraction adds moving parts (provisioners, classes, binding) that are unnecessary effort when you only run on one host, where Podman's direct volumes are simpler and predictable.

6.3 — Operator pattern (CRDs + controllers) vs plain manifests

The operator pattern extends Kubernetes with Custom Resource Definitions plus a controller that encodes operational knowledge. For a stateful service (a database, for example), an operator can automate day-2 operations like backups, failover, and version upgrades — actions that plain manifests cannot express on their own.

With only plain manifests you describe the desired state, but every operational task (backup, scaling a cluster member, recovery) is manual or scripted outside Kubernetes.

So operators give more control and automation for complex stateful systems.

Counterargument: writing or adopting an operator is significant effort and adds complexity; for simple, stateless apps it's unnecessary and plain manifests are clearer.

6.4 — Plain Kubernetes vs OpenShift S2I and developer console

Plain Kubernetes (kubectl + YAML manifests) optimizes for flexibility and control — you decide every detail and nothing is hidden, which suits experienced platform teams.

OpenShift's Source-to-Image (S2I) and developer console optimize for developer experience and speed — a developer can go from source code to a running app without hand-writing manifests, and the web console makes the platform approachable.

Each targets a different audience: raw k8s for teams that want maximum control, OpenShift for teams that want to ship quickly with guardrails.

Counterargument: OpenShift's convenience hides the underlying mechanics, which can slow a team down when they eventually need to debug or customize at the Kubernetes level.

6.5 — Migrating a workload from OpenShift to Kubernetes

Such a migration is feasible but not free. You lose OpenShift-specific objects and conveniences — Routes (replaced by Ingress), Security Context Constraints (replaced by manually configured PodSecurity), and S2I builds (replaced by an external CI/CD pipeline).

It's worth doing if the main goal is to avoid vendor lock-in or reduce licensing cost and run on any conformant cluster.

Counterargument: the effort is substantial — you must rebuild security, ingress, and build tooling that OpenShift provided out of the box, and re-test everything, so the lock-in you escape may cost more in migration work than it saves.

6.6 — CI/CD build agents in Kubernetes vs dedicated VMs

Running build agents (Jenkins agents, GitLab runners, Tekton) inside Kubernetes gives autoscaling (spin agents up on demand, down when idle), good resource efficiency, and isolation between jobs in separate Pods.

Dedicated VMs give stronger isolation and predictable performance, but they are more expensive and largely static — idle VMs still cost money.

For elastic, bursty CI load, Kubernetes is usually the better fit.

Counterargument: builds are resource-hungry and noisy; without careful resource limits a heavy build can starve neighbors ("noisy neighbor"), and some workloads need the harder isolation a VM provides.

6.7 — How Kubernetes integrates with cloud environments and dynamic capacity

Kubernetes integrates tightly with cloud providers: the Cluster Autoscaler adds or removes nodes based on pending Pods, cloud CSI drivers provision PersistentVolumes dynamically, and LoadBalancer Services create real cloud load balancers automatically.

This lets the cluster scale capacity up and down with demand instead of running fixed hardware.

Counterargument: this deep cloud integration ties you to that provider's features and APIs, which is a form of lock-in and can complicate multi-cloud or on-prem portability.

6.8 — Default security/hardening: OpenShift vs vanilla Kubernetes

OpenShift is significantly stricter by default. Security Context Constraints (SCCs) block containers from running as root unless explicitly allowed, it ships an integrated image registry with vulnerability scanning, and it enables network policies and stronger defaults out of the box.

Vanilla Kubernetes leaves most of this to the operator — you must assemble and configure security yourself, which is error-prone.

This is exactly why enterprises in regulated industries (finance, healthcare) often choose OpenShift: less manual hardening means fewer chances to misconfigure something a compliance audit will catch.

Counterargument: those strict defaults can get in the way during development (e.g. images that expect to run as root fail), and the security can be replicated on vanilla k8s by a team with the right expertise — at the cost of effort.

6.9 — Learning curve and skill set: OpenShift vs vanilla Kubernetes

For a team new to containers, OpenShift's abstractions (web console, S2I, opinionated defaults) lower the barrier — they can deploy without mastering every Kubernetes primitive first.

For a team that already knows Kubernetes deeply, the extra OpenShift layer can feel like indirection.

So whether the abstraction helps or hinders depends on the team's starting point.

Counterargument: abstractions hide what's underneath, so when something breaks at the Kubernetes level, a team that learned only "the OpenShift way" may struggle to diagnose it — sometimes learning plain Kubernetes first builds more durable skill.

6.10 — Resource footprint: full Kubernetes vs k3s for small on-prem

k3s is a certified, lightweight Kubernetes distribution — a single binary, a small memory footprint, and SQLite instead of etcd by default — while exposing the same API and kubectl. It's ideal for edge, IoT, and small on-prem deployments where resources are limited.

Full Kubernetes carries a heavier footprint (etcd, separate control-plane components) and more operational complexity, justified only when you have many nodes and advanced requirements.

Full Kubernetes is overkill for a small on-prem setup with few nodes — k3s gives the same compatibility at a fraction of the resources and effort.

Counterargument: at large scale or with strict HA needs, k3s's simplifications (e.g. single-server SQLite by default) become limitations, and full Kubernetes' robustness is worth its weight.

6.11 — Docker Compose on one host vs jumping straight to Kubernetes (small team)

A small team should start with Compose on a single host: it's simple, cheap, has a short learning curve, and describes the whole stack in one file, so the team spends time on the product rather than on infrastructure.

Kubernetes should come only when there's a real need for multi-node scaling, self-healing, or high availability.

Introducing Kubernetes too early imposes a large operational burden (cluster upgrades, networking, RBAC) with no matching benefit.

Counterargument: a single Compose host is a single point of failure with no built-in resilience or autoscaling, so if the product is expected to need those soon, the early Kubernetes investment can pay off.

6.12 — Vendor lock-in across Kubernetes vs OpenShift

Kubernetes is open source and CNCF-governed, so it runs on many providers and on-prem with relatively low lock-in and good portability.

OpenShift, as a Red Hat product, ties you to its ecosystem (Routes, SCCs, S2I, the oc tooling) and licensing, in exchange for integrated features and vendor support.

The choice directly affects long-term flexibility: more portability with k8s, more support but more vendor dependence with OpenShift.

Counterargument: "pure" Kubernetes can still lock you in subtly through provider-specific add-ons (cloud load balancers, storage classes), so vendor neutrality is a spectrum, not a guarantee.

6.13 — Recommending OpenShift for an integrated, vendor-supported platform

Yes — if a company wants an integrated platform with built-in developer and operations tooling and does not want to assemble and maintain those pieces itself, OpenShift is a strong recommendation.

It bundles the web console, CI/CD, monitoring, logging, and security defaults, all backed by Red Hat support, which reduces the team's operational load.

Counterargument: this comes with licensing cost and vendor lock-in, so a team with strong in-house Kubernetes skills and a portability requirement might prefer to build the same stack on vanilla k8s.

6.14 — Storage provisioning: Kubernetes vs regular VM workloads

Kubernetes provisions storage declaratively: a Pod requests a PVC and the cluster (via StorageClass/CSI) creates and attaches the volume automatically, even as Pods move between nodes.

Traditional VM workloads require manual disk provisioning and mounting, tied to a specific machine.

Kubernetes is more automated and portable because storage follows the workload rather than the host.

Counterargument: for a single, long-lived VM with a fixed disk, the manual approach is simpler and avoids the abstraction layers Kubernetes introduces.

6.15 — Startup with two engineers shipping quickly and cheaply

Recommendation: Docker/Podman Compose on a single host (or a small managed service), not self-managed Kubernetes.

This minimizes cost (one server), keeps operational overhead near zero, and lets two engineers focus on the product instead of running a cluster.

Move to Kubernetes — ideally a managed offering — only when growth genuinely demands scaling and high availability.

Counterargument: if the product is expected to scale fast or needs resilience from day one, starting on a managed Kubernetes service avoids a disruptive migration later.

6.16 — Docker vs Podman: architecture (daemon vs daemonless) and rootless security

Docker relies on a central daemon (dockerd) that runs as root and mediates all requests — a single point of failure and a larger attack surface. Podman is daemonless: each command launches the container directly, with no persistent privileged process.

Podman is also designed for rootless operation, running containers under a normal user's UID via user namespaces, so a container breakout does not hand the attacker root on the host.

A security-conscious team prefers Podman because removing the privileged daemon and running rootless meaningfully reduces the blast radius of a compromise.

Counterargument: Docker has a more mature ecosystem and its daemon enables features like Docker Swarm and centralized management; Docker also now offers a rootless mode, narrowing the gap.

6.17 — Day-2 overhead: self-managed Kubernetes vs managed Kubernetes

Self-managed Kubernetes gives full control but puts all day-2 work on you — cluster upgrades, node scaling, etcd backups, security patching.

Managed Kubernetes (EKS, GKE, AKS, etc.) offloads much of that to the provider, trading some control and added cost for far less operational burden.

For most teams, managed is the pragmatic choice unless control or compliance requires self-management.

Counterargument: managed services can lag on versions, limit deep customization, and still leave you responsible for workloads and some cluster config — "managed" is not "hands-off."

6.18 — Media-streaming company with spiky, unpredictable global traffic

Recommendation: Kubernetes on a cloud provider with Horizontal Pod Autoscaler (HPA) and Cluster Autoscaler, plus multi-region deployment and a CDN.

This elastically scales Pods and nodes up during spikes and down when traffic falls, handling unpredictable global load cost-effectively, while a CDN absorbs edge traffic.

Counterargument: this architecture is complex and expensive to design and operate; a smaller or more predictable service might be over-engineered by it, and heavy cloud reliance increases provider lock-in.

6.19 — Company that has outgrown Docker Swarm / a single Compose host

You must weigh migration effort against long-term benefit. Migrating to Kubernetes means rewriting manifests, learning new concepts, and re-testing — real cost.

But the payoff is a richer ecosystem, robust autoscaling and self-healing, broad community support, and a platform that scales far beyond Swarm or a single host.

If the company's scaling and reliability needs are clearly growing, the migration is justified.

Counterargument: if current needs are modest and stable, the migration effort may not pay off soon — sometimes a managed container service or simply scaling the existing setup is the better near-term move.

6.20 — Would you choose Kubernetes for a microservices architecture?

Yes. Kubernetes is well suited to microservices: it orchestrates many independent services, provides self-healing (restarts failed Pods), supports rolling updates and rollbacks per service, and offers built-in service discovery.

Its ecosystem reinforces this — Helm for packaging, a service mesh like Istio for traffic management, and Prometheus for monitoring are all designed for microservice patterns.

Counterargument: that power brings significant complexity; for an application with only a few services or a small team, Kubernetes can be more overhead than the architecture warrants — Compose or a managed platform may serve better.

6.21 — Would you choose Kubernetes for enterprise-grade applications?

Yes. Kubernetes offers the scalability enterprises need, an enormous and active community, and a rich feature set (autoscaling, declarative config, rolling updates, RBAC, extensibility via operators).

Its CNCF governance and broad vendor support make it a safe long-term platform choice.

Counterargument: it demands real operational expertise and day-2 effort; an enterprise without that skill set may be better served by a managed service or by OpenShift, which packages enterprise features and support on top of Kubernetes.

6.22 — Would you choose OpenShift for a microservices architecture?

Yes. OpenShift builds on Kubernetes' orchestration and adds integrated tooling (web console, CI/CD, S2I), stronger security defaults, and vendor support — all helpful when running many microservices.

For teams that want microservice orchestration without assembling the surrounding platform themselves, it's a strong fit.

Counterargument: it carries licensing cost and Red Hat lock-in, and its opinionated defaults may constrain teams that want full control over their microservice tooling.

6.23 — Would you choose OpenShift for enterprise-grade applications?

Yes. OpenShift suits enterprise applications through Kubernetes-level scalability combined with enterprise support, built-in security/compliance features, and integrated developer and operations tooling — attractive to large or regulated organizations.

The vendor backing reduces operational risk for mission-critical systems.

Counterargument: the cost of licensing and the dependence on Red Hat's ecosystem are real downsides; an enterprise prioritizing portability and cost control might prefer vanilla Kubernetes with its own tooling.

---
---

# CHEAT — najčešće greške koje rješavaš u sekundi
```
podman ps             vs  podman ps -a       # -a pokazuje i zaustavljene (Exited)
ime "already in use"      -> podman rm -f <ime>  ili  --replace
pod "already exists"      -> kubectl delete pod <ime>  (ili --grace-period=0 --force)
dial tcp ... i/o timeout  -> minikube status; minikube start
lsetxattr ... not permitted -> sudo chcon -Rt container_file_t /dir  pa mount bez :Z
toomanyrequests           -> podman login docker.io
```
