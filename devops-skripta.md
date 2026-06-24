# Intro to DevOps — Skripta za učenje (LO3–LO6)

> Ova skripta objašnjava **koncepte** koji se kriju iza practice pitanja, a ne rješava pitanja jedno po jedno. Cilj je da nakon čitanja razumiješ *zašto* nešto radiš, pa onda svaki praktični zadatak možeš sam izvesti.
>
> **Alati koje koristiš:** `podman` (umjesto Dockera — gotovo iste naredbe), `podman compose`, `kubectl` + `minikube` (Kubernetes), te malo OpenShift teorije.
>
> **Mala napomena o terminologiji:** Podman i Docker su praktički zamjenjivi. Gdje piše `podman`, u Dockeru bi pisalo `docker`. `Containerfile` = `Dockerfile`. Razlike objašnjavam u LO6.

---

# LO3 — Isporuka aplikacija kontejnerima, mreže i sigurnost komponenti

Ovo je cjelina o tome kako **više kontejnera spojiti u jednu aplikaciju** (npr. web aplikacija + baza podataka), kako oni međusobno komuniciraju, kako čuvaš podatke i kako to izoliraš i osiguravaš.

## 1. Pojedinačni kontejner vs. više povezanih kontejnera

Jedan kontejner = jedan proces (npr. samo web server, ili samo baza). Prava aplikacija ima više dijelova ("tierova"): npr. **frontend/aplikacija** i **baza podataka**. Ti dijelovi moraju moći razgovarati međusobno, ali ne želiš da su svi izloženi vanjskom svijetu.

## 2. Pod (podman pod) — dijeljeni mrežni prostor

**Pod** je grupa kontejnera koji dijele isti **network namespace**. Što to znači:

- Kontejneri u istom podu vide se preko `localhost`. Ako app sluša na portu 8080, a baza na 5432, app se bazi obraća na `localhost:5432` — kao da su na istom računalu.
- Pod ima jednu zajedničku IP adresu prema vani.
- Port objavljuješ **na razini poda** (`podman pod create -p 8080:80`), ne na pojedinom kontejneru.

```bash
podman pod create --name mojpod -p 8080:80
podman run -d --pod mojpod --name app mojaslika
podman run -d --pod mojpod --name baza postgres
# app sada doseže bazu preko localhost:5432
```

Ovaj koncept je **most prema Kubernetesu** — u Kubernetesu je pod osnovna jedinica i radi na potpuno isti način (kontejneri u podu dijele mrežu i mogu dijeliti volumene).

**Namespaces** (Linux koncept izolacije): kontejneri u podu po defaultu dijele **network** i **IPC** namespace, ali **NE** dijele **PID** namespace (svaki vidi samo svoje procese). To provjeriš s `podman pod inspect`.

## 3. User-defined mreže i DNS po imenu

Ako ne koristiš pod nego zasebne kontejnere, povezuješ ih preko **vlastite (user-defined) mreže**:

```bash
podman network create mreza1
podman run -d --network mreza1 --name baza postgres
podman run -d --network mreza1 --name app mojaslika
```

Ključna stvar: na **vlastitoj mreži Podman ima ugrađeni DNS**, pa se kontejneri međusobno pronalaze **po imenu**. App se bazi obraća jednostavno preko hostname-a `baza`, ne preko IP adrese (koja se mijenja).

> **VAŽNO (vraća se u LO5):** Na **defaultnoj** mreži (`podman` bez `--network`) DNS razlučivanje po imenu **NE radi**. Zato dva kontejnera na defaultnoj mreži ne mogu pingati jedan drugog po imenu — rješenje je staviti ih na vlastitu mrežu.

Mrežu inspektiraš da nađeš IP kontejnera:
```bash
podman inspect -f '{{.NetworkSettings.IPAddress}}' app
```

## 4. Two-tier stack (aplikacija + baza)

"Two-tier" = dva sloja: aplikacijski sloj + sloj baze. Primjeri parova: **Ghost + MySQL**, **Gitea + PostgreSQL**, **Redmine + PostgreSQL**. Tipičan postupak:

1. Napraviš mrežu.
2. Pokreneš bazu (s lozinkom, korisnikom, imenom baze preko env varijabli).
3. Pokreneš aplikaciju i kažeš joj kako da nađe bazu (host = ime kontejnera baze, port, korisnik, lozinka).
4. Otvoriš app u browseru i prođeš "first-run setup" (početno podešavanje, admin korisnik itd.).

Bit je razumjeti da aplikacija dobiva podatke o bazi kroz konfiguraciju/env varijable, a bazu pronalazi po imenu na zajedničkoj mreži.

## 5. Perzistencija podataka — named volumes

Problem: kad obrišeš kontejner baze, **nestaju i svi podaci** (jer žive unutar kontejnera). Rješenje je **named volume** — spremište podataka koje živi *izvan* kontejnera i preživljava brisanje/ponovno stvaranje.

```bash
podman volume create dbdata
podman run -d --name baza -v dbdata:/var/lib/postgresql/data postgres
# obriši kontejner, napravi novi s istim volumenom -> podaci su i dalje tu
podman rm -f baza
podman run -d --name baza -v dbdata:/var/lib/postgresql/data postgres
```

**Bind mount vs named volume** (vrlo važna razlika):
- **Named volume** (`-v ime:/putanja`) — Podman njime upravlja, idealan za **podatke** (baza). Preživljava sve.
- **Bind mount** (`-v /putanja/na/hostu:/putanja`) — direktno mapiraš mapu s host računala u kontejner. Idealan za **konfiguraciju** ili kod tijekom razvoja, jer odmah vidiš promjene s diska.

Zašto bi koristio oba u istom kontejneru: konfiguraciju mijenjaš s hosta (bind mount), a podatke čuvaš trajno i odvojeno (named volume).

## 6. Backup i restore volumena

Volumen sigurnosno kopiraš tako da pokreneš mali pomoćni kontejner (alpine/busybox) koji **montira volumen** i spakira ga u `.tar`:

```bash
# backup
podman run --rm -v dbdata:/data -v $(pwd):/backup alpine \
  tar czf /backup/backup.tar.gz -C /data .
# restore u novi volumen
podman volume create dbdata2
podman run --rm -v dbdata2:/data -v $(pwd):/backup alpine \
  tar xzf /backup/backup.tar.gz -C /data
```
Logika: pomoćni kontejner vidi volumen kao mapu i samo arhivira/odarhivira njen sadržaj.

## 7. Sigurnost: podman secrets umjesto --env

Lozinku baze **ne bi** trebao slati kroz `--env MYSQL_PASSWORD=tajna`, jer:
- vidi se u `podman inspect`, u `ps`, u historiji shella.

Bolje: **secret** — Podman ga čuva šifrirano i dostavlja kontejneru kao datoteku, ne kao vidljivu varijablu.

```bash
echo "tajna123" | podman secret create db_pass -
podman run -d --secret db_pass --name baza ...
# secret se pojavi kao datoteka u /run/secrets/db_pass unutar kontejnera
```
Sigurnosna korist: tajna nije u procesnom okruženju ni u inspect ispisu.

## 8. podman compose — opisivanje cijelog stacka u jednoj datoteci

Umjesto da ručno tipkaš 5 `podman run` naredbi, sve opišeš u `compose.yaml` i pokreneš jednom naredbom:

```yaml
services:
  app:
    image: mojaslika
    ports:
      - "8080:80"
    depends_on:
      baza:
        condition: service_healthy
  baza:
    image: postgres
    env_file: .env          # kredencijali iz datoteke, ne inline
    networks:
      - backend             # interna mreža, bez published porta
    healthcheck:
      test: ["CMD", "pg_isready"]
      interval: 5s
      retries: 5
networks:
  backend:
    internal: true          # nedostupna s hosta
```

```bash
podman compose up -d      # pokreni sve u pozadini
podman compose ps         # pregled servisa
podman compose logs       # logovi svih servisa
podman compose down       # zaustavi i obriši
```

Ključni koncepti iz compose dijela:

- **`env_file`** — kredencijale držiš u zasebnoj `.env` datoteci umjesto da ih upisuješ ravno u YAML (čišće i sigurnije).
- **Interna mreža + bez published porta** — bazu staviš na `internal: true` mrežu i ne objaviš joj port. Tako joj se može pristupiti samo iz drugih kontejnera, a **NE** s hosta. To je segmentacija: izloži samo app, sakrij bazu.
- **`healthcheck` + `depends_on: condition: service_healthy`** — app čeka da baza bude *zdrava* (ne samo pokrenuta) prije nego krene. Rješava problem da app padne jer se javio bazi prerano.
- **Skaliranje** — `podman compose up --scale app=3` pokrene 3 kopije app servisa; promet se raspodjeljuje među njima (load balancing).
- **CPU/memorija po servisu** — limite postaviš u compose datoteci i provjeriš s `podman stats`.
- **`podman compose down`** — po defaultu **ostavlja** named volumene (podaci preživljavaju). S `--volumes` ih i **obriše**.

## 9. Reverse proxy (nginx ispred aplikacije)

**Reverse proxy** je kontejner koji stoji "ispred" aplikacije i prima sav promet s hosta pa ga prosljeđuje unutarnjoj aplikaciji. Zašto: jedna ulazna točka, lakše TLS/HTTPS, možeš sakriti više aplikacija iza jednog porta.

Postaviš nginx i app na zajedničku mrežu, host promet ide na nginx (npr. port 80), nginx ga prosljeđuje na `app:80` po imenu kontejnera.

## 10. DB admin kontejneri

Alati poput **adminer** ili **pgAdmin** su kontejneri koje staviš na istu mrežu kao baza pa kroz web sučelje pregledavaš/uređuješ bazu. Korisno za provjeru je li podatak stvarno spremljen.

## 11. Most prema Kubernetesu

- `podman generate kube` — iz pokrenutog poda generira **Kubernetes YAML**. Ovo doslovno povezuje LO3 i LO4: tvoj lokalni pod prevedeš u k8s manifest.
- `podman kube play datoteka.yaml` — pokrene pod *iz* k8s YAML-a.
- `podman kube play --down datoteka.yaml` — sruši ga.

## 12. Segmentacija mreže (frontend/backend)

Jedan kontejner možeš spojiti na **dvije mreže** odjednom — npr. `frontend` (gdje je reverse proxy) i `backend` (gdje je baza). Korist: app komunicira s oba sloja, ali frontend nikad ne vidi backend direktno. To je sigurnosna **segmentacija** — smanjuješ "napadnu površinu".

**Rizik dijeljenog read-write volumena:** ako dvije replike iste aplikacije pišu u isti named volume istovremeno, mogu si pokvariti podatke (race conditions, korupcija datoteka). Dijeljeni volumen je u redu za **čitanje** konfiguracije, opasan za istovremeno **pisanje**.

---

# LO4 — Ubrzana isporuka višeslojnih aplikacija (Kubernetes)

Kubernetes (k8s) je **orkestrator** — sustav koji automatski upravlja s puno kontejnera preko više računala (čvorova/nodeova): pokreće ih, skalira, liječi kad padnu, ažurira bez prekida. Ovdje učiš kroz `kubectl` (alat za komunikaciju s clusterom) i `minikube` (mali lokalni cluster za vježbu).

## A. Osnovna hijerarhija: Pod → ReplicaSet → Deployment

Ovo je temelj cijelog LO4:

- **Pod** — najmanja jedinica; jedan ili više kontejnera koji dijele mrežu i mogu dijeliti volumene (isto kao podman pod). Pod je "potrošan" — ako padne, ne oporavlja se sam.
- **ReplicaSet** — pazi da uvijek postoji točno N kopija (replika) poda. Ako jedna padne, stvara novu.
- **Deployment** — sloj iznad ReplicaSeta; omogućuje **rolling update** i **rollback**. Ti praktički uvijek radiš s Deploymentom, on stvara ReplicaSet, koji stvara Podove.

Veza ide preko **labela i selektora**: Deployment ima selektor (`matchLabels`) koji "hvata" Podove s odgovarajućim labelama. Tako Deployment zna koji su Podovi njegovi.

```bash
# imperativno (brzo, jednom naredbom)
kubectl create deployment web --image=nginx:1.25 --replicas=3
# izvezi generirani YAML u datoteku (deklarativni pristup)
kubectl get deployment web -o yaml > web.yaml
```

**Imperativno vs deklarativno:**
- *Imperativno* = naredbe koje nešto rade odmah (`kubectl create`, `kubectl scale`). Brzo, ali nije zapisano.
- *Deklarativno* = opišeš željeno stanje u YAML manifestu i kažeš `kubectl apply -f web.yaml`. Kubernetes sam postiže to stanje. Bolje za produkciju i verzioniranje.

## B. Skaliranje

Dva načina povećanja broja replika:
```bash
kubectl scale deployment web --replicas=5     # imperativno
# ili izmijeniš replicas: 5 u YAML-u pa kubectl apply -f web.yaml
```
Rezultat: ReplicaSet podigne broj Podova na 5.

## C. Rolling update, rollout, rollback

**Rolling update** = postupna zamjena starih Podova novima, bez prekida usluge (npr. nova verzija slike):
```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web        # gledaj napredak
kubectl rollout history deployment/web       # povijest revizija
kubectl rollout undo deployment/web          # vrati na prethodnu reviziju
```

- **CHANGE-CAUSE** stupac u history pokazuje *zašto* je nastala revizija. Popuni ga anotacijom `kubectl annotate ... kubernetes.io/change-cause="..."` ili zastavicom `--record`.
- **revisionHistoryLimit: 3** — koliko starih ReplicaSetova k8s čuva za rollback. Manji broj = manje "smeća", ali možeš se vratiti samo toliko verzija unatrag.

**Strategije ažuriranja:**
- **RollingUpdate** (default) s `maxSurge` (koliko *viška* Podova smije nastati tijekom update-a) i `maxUnavailable` (koliko ih smije nedostajati). Npr. `maxSurge: 1, maxUnavailable: 0` znači: uvijek imaš puni broj dostupnih Podova + jedan novi viška → **nula prekida**.
- **Recreate** — prvo ugasi SVE stare, pa digne nove. Ima kratak prekid. Nužno kad stara i nova verzija ne smiju raditi istovremeno (npr. nekompatibilna shema baze, ili exkluzivni lock na resurs).

**Zaglavljen rollout:** ako staviš nepostojeći tag slike, novi Podovi ne mogu se povući (ImagePullBackOff), ali **stari Podovi i dalje poslužuju promet** — to je sigurnosna značajka Deploymenta. Ako rollout dugo ne uspije → `ProgressDeadlineExceeded`.

## D. Resursi: requests i limits

- **requests** = koliko CPU/RAM-a Pod *garantirano* dobije (k8s ga koristi za raspoređivanje na čvorove).
- **limits** = gornja granica koju Pod *ne smije prijeći* (ako prekorači RAM → kontejner biva ubijen, **OOMKilled**).

Ako `requests` traže više nego ijedan čvor ima → Pod ostaje **Pending** (Insufficient cpu/memory).

## E. Ostali načini pokretanja (controlleri)

| Objekt | Čemu služi |
|---|---|
| **Deployment** | Stateless aplikacije (web serveri). Podovi su zamjenjivi. |
| **StatefulSet** | Stateful aplikacije (baze). Daje **stabilna imena** (web-0, web-1...), stvara/gasi Podove **redom**, svaka replika dobiva **svoj PVC** (svoju disk particiju). |
| **DaemonSet** | Točno **jedan Pod po čvoru** (npr. agent za logove/monitoring). Doda li se novi čvor, automatski dobije svoj Pod. |
| **Job** | Pokreni zadatak **jednom do završetka** (npr. izračun). Pokaže `COMPLETIONS`, rezultat čitaš iz logova. |
| **CronJob** | Job po rasporedu (cron sintaksa, npr. svake minute). Možeš ga `suspend`-ati i ručno pokrenuti s `kubectl create job --from=cronjob/ime`. |
| **bare Pod** | Pod bez controllera. Obrišeš ga → nestaje zauvijek (nitko ga ne ponovo-stvara). Za razliku od Deployment-Poda koji se ponovo digne. |

**Headless Service** (`clusterIP: None`) ide uz StatefulSet — vraća **A-zapise za svaki Pod posebno** (web-0, web-1...) umjesto jedne dijeljene virtualne IP-e. Tako se možeš obratiti točno određenoj replici.

## F. Dijeljenje i izolacija unutar Poda

- **Sidecar** — drugi kontejner u istom Podu koji pomaže glavnom (npr. logiranje, proxy). Dijele mrežu (`localhost`) i mogu dijeliti volumen.
- **emptyDir** — privremeni volumen koji žive dok živi Pod; dva kontejnera u Podu mogu njime razmjenjivati datoteke (jedan piše, drugi čita). `sizeLimit` ograničava veličinu; prekoračenje → Pod biva izbačen.
- **initContainer** — kontejner koji se izvrši **prije** glavnog (npr. pripremi podatke u dijeljeni volumen), pa tek onda glavni krene.
- **readOnly: true** na mountu — kontejner ne smije pisati u taj volumen (writes se odbijaju).
- **nodeSelector** — Pod ide samo na čvor s odgovarajućom labelom; ako nema takvog čvora → Pod ostaje **Pending**.

## G. Storage: PV, PVC, StorageClass

- **PersistentVolume (PV)** — komad stvarnog spremišta u clusteru.
- **PersistentVolumeClaim (PVC)** — "zahtjev" Poda za spremištem ("trebam 1 GB"). Pod montira PVC, ne PV direktno.
- **StorageClass** — recept za **dinamičko** stvaranje PV-a na zahtjev (kad nastane PVC, automatski se napravi PV).

```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc
```
StatefulSet sa 3 replike → stvori **3 PVC-a** (po jedan po Podu). Brisanjem StatefulSeta PVC-ovi **ostaju** (da se podaci ne izgube) — moraš ih obrisati ručno.

## H. ConfigMap i Secret (konfiguracija)

- **ConfigMap** — nepovjerljiva konfiguracija (postavke, varijable).
- **Secret** — povjerljivi podaci (lozinke, certifikati). **Bitno:** Secret je samo **base64-kodiran, NIJE šifriran** — svatko s pristupom ga može dekodirati. Base64 je kodiranje, ne enkripcija.

Načini korištenja:
```bash
# ConfigMap/Secret kao volumen -> svaki ključ postane datoteka
# subPath -> montiraš samo JEDAN ključ na točnu putanju (caveat: takav mount se NE osvježava automatski)
# envFrom + secretRef -> učitaj SVE ključeve Secreta kao env varijable odjednom
```

Tipovi Secreta:
- **generic** (`--from-literal` ili `--from-file`) — opći podaci, npr. user/password ili certifikat+ključ.
- **kubernetes.io/tls** — TLS certifikat + privatni ključ; koristi se npr. na Ingressu za HTTPS.
- **docker-registry** (image-pull secret) — kredencijali za privatni registry; referenciraš ga preko `imagePullSecrets` da k8s smije povući privatnu sliku.

Secret kao **volumen** (datoteke, restriktivne dozvole) vs kao **env var**: env varijable se lakše procure (npr. u logovima, u `inspect`), pa je volumen često sigurniji.

## I. Servisi (Service) — kako se dolazi do Podova

Podovi su prolazni i mijenjaju IP. **Service** daje **stabilnu adresu i DNS ime** i radi **load balancing** preko svih Podova koje selektira (po labelama).

| Tip Service | Što radi |
|---|---|
| **ClusterIP** (default) | Dostupan **samo unutar** clustera. Razlučuje se DNS-om: `svc.namespace.svc.cluster.local`. |
| **NodePort** | Otvori isti port na **svakom čvoru**; dostupan izvana preko `IP_čvora:nodePort`. Na minikube: `minikube service <svc> --url`. |
| **LoadBalancer** | Traži vanjsku IP od cloud providera. Na minikube ostaje `<pending>` (nema cloud LB-a) → koristiš `minikube tunnel`. |
| **headless** (`clusterIP: None`) | Bez zajedničke IP; vraća pojedinačne A-zapise (za StatefulSet). |

**port vs targetPort vs nodePort** (često zbunjuje):
- **port** — port na kojem *Service* sluša unutar clustera.
- **targetPort** — port na kojem *Pod/kontejner* stvarno sluša.
- **nodePort** — vanjski port na čvoru (samo kod NodePort tipa).

**Endpoints / EndpointSlice** — lista stvarnih Pod IP-eva iza Servicea. Puni se automatski iz selektora. Ako je prazna → selektor ne hvata nijedan Pod (labela ne odgovara) → Service "ne radi".

**DNS i namespacei:**
- Iz drugog namespacea moraš koristiti **FQDN**: `svc.namespace.svc.cluster.local`. Kratko ime radi samo unutar istog namespacea.
- `kubectl exec` u Pod pa `nslookup svc` i `cat /etc/resolv.conf` za debug DNS-a.

**port-forward** — `kubectl port-forward` privremeno preusmjeri lokalni port na Service ili Pod (za testiranje, bez izlaganja vani). Na Service ide preko load balancinga, na Pod direktno na taj jedan Pod.

**Throwaway debug pod** za testiranje povezanosti iznutra:
```bash
kubectl run tmp --rm -it --image=busybox -- sh
# pa unutra: wget / nc / nslookup prema Serviceu
```

## J. Probe (zdravstvene provjere)

Kubernetes provjerava jesu li Podovi živi i spremni:
- **livenessProbe** — je li kontejner **živ**? Ako padne → k8s ga restarta. (npr. HTTP GET na `/` port 80)
- **readinessProbe** — je li kontejner **spreman primati promet**? Ako ne → izbaci ga iz Servicea (ne ubije ga). (npr. TCP socket na 6379 za redis)
- **startupProbe** — daje aplikaciji **vrijeme da se digne** prije nego krenu liveness provjere. Budžet = `failureThreshold × periodSeconds` (npr. 24 × 5s = 120s za boot od ~2 min).

**Default kad nema proba:** k8s smatra Pod zdravim čim kontejner krene (samo gleda je li proces živ), bez ikakve provjere aplikacijskog stanja. Zato aplikacija može "raditi" iako zapravo ne odgovara.

## K. DockerHub autentikacija (pull limit) — *označeno pitanje*

DockerHub ograničava broj povlačenja slika za **anonimne** korisnike. Da to izbjegneš, prijaviš se kao autentificiran korisnik. U Kubernetesu to znači stvoriti **docker-registry Secret** s tvojim DockerHub kredencijalima i referencirati ga preko **imagePullSecrets** u Podu/Deploymentu (ili ServiceAccountu). Tip resursa koji trebaš = **Secret (kubernetes.io/dockerconfigjson)**.

---

# LO5 — Rješavanje problema u isporuci aplikacija

Ova cjelina je **debugging**: dobiješ pokvarenu naredbu/datoteku/Pod, prepoznaš grešku, popraviš je i objasniš zašto je bila kriva. Nema novih koncepata — testira razumijevanje LO1–LO4. Grupiram po tipovima grešaka.

## A. Greške u `podman run` naredbama

**1. Obrnuto mapiranje porta** — `-p 80:8080` znači **host:kontejner**. Ako nginx unutra sluša na 80, a ti napišeš `-p 80:8080`, šalješ promet na port 8080 u kontejneru gdje nitko ne sluša. Ispravak: `-p 8080:80` (host 8080 → kontejner 80). **Pravilo: `-p HOST:CONTAINER`.**

**2. Env varijabla bez vrijednosti** — `-e MYSQL_ROOT_PASSWORD` (bez `=vrijednost`) ne postavlja lozinku, pa MySQL init padne. Pogledaj `podman logs`. Ispravak: `-e MYSQL_ROOT_PASSWORD=tajna` (ili koristi secret).

**3. `--network host` ignorira `-p`** — u host mreži kontejner dijeli mrežu hosta direktno, pa `-p` nema smisla (port je već "na hostu"). Ispravak: ili makni `--network host` i koristi `-p`, ili prihvati da app sluša direktno na host portu.

**4. busybox odmah izađe (Exited 0)** — busybox nema dugotrajni proces; pokrene se i završi. Ispravak: daj mu naredbu koja "drži" kontejner živim, npr. `sleep infinity` ili `sleep 1d`.

**5. `--rm` + detached gotcha** — `--rm -d ... echo hello`: kontejner odradi echo, završi, i **odmah se obriše** (`--rm`). Kad pokušaš `podman logs`, kontejnera više nema. Ispravak: makni `--rm` ako trebaš logove poslije.

**6. Premali memory limit** — `--memory 8m` za MySQL: baza treba puno više RAM-a, pa nikad ne postane healthy (biva ubijena). Ispravak: digni limit (npr. 512m+).

**7. DNS po imenu na defaultnoj mreži ne radi** — dva kontejnera bez `--network`, `ping b` ne uspijeva jer default mreža nema DNS razlučivanje. Ispravak: napravi vlastitu mrežu i spoji oba na nju (vidi LO3, točka 3).

**8. SELinux / nedostaje `:Z` zastavica** — `-v ./html:/usr/share/nginx/html` bez `:Z` ili `:z`: SELinux blokira pristup, datoteke se ne vide. Ispravak: `-v ./html:/usr/share/nginx/html:Z` (`:Z` = privatni label za taj kontejner, `:z` = dijeljeni). Ovo je čest gotcha na Red Hat / Fedora sustavima.

## B. Greške u Containerfile / Dockerfile

**Nedostaje `apt-get update` prije install** — `RUN apt-get install -y nginx` padne jer popis paketa nije osvježen. Ispravak: `RUN apt-get update && apt-get install -y nginx` (u istom RUN-u).

**Shell vs exec forma CMD-a** — `CMD python app.py` (shell forma) pokreće proces kroz shell, pa app ne dobiva signale (ne može se čisto zaustaviti). Ispravak: exec forma `CMD ["python", "app.py"]` — proces je PID 1 i prima SIGTERM.

**Loš redoslijed za layer caching** — `COPY . /app` pa `RUN npm install`: svaka promjena izvornog koda poništi cache i `npm install` se vrti iznova. Ispravak: prvo kopiraj samo `package.json`, pokreni `npm install`, pa tek onda `COPY . /app`. Tako se ovisnosti keširaju i ne instaliraju ponovo dok se ne promijene.

**Prepisivanje PATH-a** — `ENV PATH=/app/bin` izbriše originalni PATH, pa se `curl`, `sh` i ostali alati više ne nalaze. Ispravak: dopiši, ne zamijeni: `ENV PATH=/app/bin:$PATH`.

**EXPOSE ne objavljuje port** — `EXPOSE 8080` je samo **dokumentacija** (govori koji port aplikacija koristi), ne otvara ništa prema hostu. Plus, ako app sluša na 3000 a ti mapiraš 8080:8080, ne poklapa se. Ispravak: uskladi port na kojem app sluša s `-p`/`targetPort`; EXPOSE sam po sebi ne radi publish.

**Velika slika — čisti u istom layeru** — `apt-get install` ostavi cache koji napuhne sliku. Ispravak: u istom RUN-u počisti: `RUN apt-get update && apt-get install -y build-essential && rm -rf /var/lib/apt/lists/*`. Mora biti **isti layer** jer svaki RUN stvara novi layer — brisanje u kasnijem layeru ne smanjuje veličinu (datoteke i dalje žive u ranijem layeru).

**USER prije COPY → permission denied** — postaviš `USER appuser` pa `COPY` + `RUN pip install`: appuser nema prava pisati. Ispravak: kopiraj i instaliraj kao root (ili koristi `COPY --chown=appuser`), pa tek na kraju prebaci na `USER appuser`.

**Ogromna slika → multi-stage build** — `golang:1.22` slika ostane ~1 GB jer sadrži cijeli Go toolchain. Ispravak: **multi-stage**: u prvoj fazi build (`FROM golang ... go build`), u drugoj kopiraš samo gotov binarni file u minimalnu sliku (`FROM alpine` ili `scratch`). Konačna slika je par MB.

## C. Kubernetes debugging — stanja Podova i kako ih dijagnosticirati

Glavni alati: `kubectl describe pod <ime>` (vidi Events na dnu), `kubectl logs` (i `--previous` za prošli crash), `kubectl get events --sort-by=.lastTimestamp`.

| Stanje / greška | Uzrok | Popravak |
|---|---|---|
| **Pending** | Nema mjesta/čvora: nedovoljno CPU/RAM, nepostojeći nodeSelector, PVC ne postoji | `describe pod` → pročitaj Events; smanji requests / popravi selektor / stvori PVC |
| **ImagePullBackOff** | Kriv naziv/tag slike, ili privatni registry bez pull-secreta | Ispravi naziv/tag; dodaj imagePullSecret |
| **CrashLoopBackOff** | App se sruši pa restarta u krug | `kubectl logs --previous` → pročitaj zadnji crash; popravi uzrok |
| **OOMKilled** | Kontejner prešao memory limit | `get pod -o yaml`/`describe` potvrdi; digni memory limit ili popravi app |
| **Completed / CrashLoop** (ubuntu bez naredbe) | Slika nema dugotrajni proces, završi odmah | Dodaj pravu naredbu (npr. dugotrajni proces) |
| **CreateContainerConfigError** | Fali ključ u configMapKeyRef/secretKeyRef | `describe` → nađi koji ključ fali; dodaj ga |
| **never Ready** | Readiness probe pada | Popravi probu ili app da odgovara na nju |
| **stuck Terminating** | Finalizeri / grace period | Razumij finalizere; `--grace-period=0 --force` (rizik: prljavo gašenje) |

**Service vraća prazno / ne radi** — najčešće **nesklad labela** između Service selektora i Pod labela. Provjeri `kubectl get endpoints <svc>` — ako je prazno, selektor ne hvata Podove.

**Selektor ne odgovara templateu** — ako `spec.selector.matchLabels` ≠ `spec.template.metadata.labels`, `kubectl apply` vrati validacijsku grešku. Moraju se poklapati.

**Krivi apiVersion/kind par** — npr. `apiVersion: apps/v1` + `kind: Pod` (Pod je u `v1`, ne `apps/v1`). Ispravi grupu/verziju za taj kind.

**Secret iz drugog namespacea** — Secret i Pod moraju biti u **istom namespaceu** (Secreti su namespace-scoped). Kopiraj Secret u pravi namespace.

**Sistematska dijagnoza "app ne doseže bazu":** provjeri **redom** → Pod radi? → Service postoji? → Endpoints popunjeni? → DNS razlučuje ime? → točan port? Prva karika koja pukne je uzrok.

**Korisni alati za dubinski debug:**
```bash
kubectl describe pod <ime>                    # Events, razlozi
kubectl logs <pod> --previous                 # zadnji crash
kubectl get events --sort-by=.lastTimestamp   # vremenski slijed
kubectl exec -it <pod> -- sh                   # shell u Pod (DNS: nslookup, cat /etc/resolv.conf)
kubectl debug -it <pod> --image=busybox --target=<c>   # za distroless/no-shell kontejnere
kubectl run tmp --rm -it --image=busybox -- sh # throwaway pod za test povezanosti
kubectl apply --dry-run=server --validate=true -f m.yaml  # uhvati typo prije primjene
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].restartCount}'  # broj restarta
```

- **Whitespace/indentacija u YAML-u** (LO5 #20, #40) — YAML je osjetljiv na razmake; krivo uvlačenje ili typo (`imagePullpolicy` umjesto `imagePullPolicy`) ruši apply. Uhvati to s `--dry-run=server` / `--validate=true`.
- **NotReady čvor** — provjeri kubelet, disk pressure, CNI mrežu; na minikube `kubectl describe node` i `minikube logs`.
- **ProgressDeadlineExceeded** — rollout predugo ne uspijeva (npr. zbog ImagePullBackOff); `describe deployment` / `rollout status` da nađeš uzrok.

## D. OpenShift Route vs Kubernetes Ingress

Oboje izlažu HTTP(S) servise vani, ali:
- **Ingress** (vanilla k8s) — zahtijeva zaseban **ingress controller** (nginx, traefik...) da bi uopće radio; ti instaliraš i konfiguriraš.
- **Route** (OpenShift) — ugrađen u platformu, radi "out of the box" s integriranim routerom, jednostavnija TLS konfiguracija. OpenShift-specifičan resurs.

---

# LO6 — Evaluacija sustava za orkestraciju (teorija)

Ovo je **isključivo teorijska** cjelina. Pitanja su tipa "usporedi", "kritički procijeni", "bi li odabrao X i zašto". Ne treba praksa — treba ti razumijevanje koncepata i argumenti za obje strane. Ispod su sve teme koje pitanja pokrivaju, s ključnim argumentima.

## 1. Docker/Podman vs Kubernetes — mreža

- **Docker/Podman mreža** — jednostavna, na jednom hostu: bridge mreže, DNS po imenu kontejnera, port publishing. Dobra za malo kontejnera na jednom računalu.
- **Kubernetes mreža** — distribuirana preko više čvorova: svaki Pod ima vlastitu IP (flat network), Servisi daju stabilne adrese i load balancing, CNI dodaci (Calico, Flannel) implementiraju mrežu, NetworkPolicy za segmentaciju. Složenija, ali skalabilna preko cijelog clustera.

## 2. Storage: Kubernetes (CSI, StorageClass, dinamičko provisioniranje) vs Podman

- **Kubernetes** — apstrahira spremište: PVC traži prostor, StorageClass ga dinamički stvara, **CSI** (Container Storage Interface) omogućuje priključivanje bilo kojeg storage providera (cloud diskovi, NFS, Ceph...). Vrlo fleksibilno za **stateful** workloade.
- **Podman** — jednostavniji volume sustav, vezan uglavnom uz lokalni host. Manje fleksibilno za distribuirane stateful aplikacije.
- **Zaključak:** Kubernetes je fleksibilniji za stateful workloade jer odvaja zahtjev za spremištem od konkretne implementacije i radi preko više čvorova.

## 3. Operator pattern (CRD + controller) vs obični manifesti

- **Obični manifesti** — ti ručno opisuješ stanje; za složene stateful servise (baze s backupom, failoverom) moraš sam izvoditi "day-2" operacije.
- **Operator** — proširuje k8s vlastitim resursom (**CRD** = Custom Resource Definition) + controllerom koji "zna" kako voditi tu aplikaciju (automatski backup, upgrade, failover). Više kontrole i automatizacije, ali veći trud za izradu/održavanje operatora.

## 4. Plain Kubernetes vs OpenShift S2I i developer konzola

- **Plain k8s** (`kubectl` + manifesti) — maksimalna kontrola i fleksibilnost; optimiran za inženjere koji žele sve sami konfigurirati.
- **OpenShift** — **S2I (Source-to-Image)** automatski gradi sliku iz izvornog koda (bez ručnog pisanja Containerfilea); web developer konzola; optimiran za **brzinu i developer experience**, manje "ručnog rada".

## 5. Migracija OpenShift → Kubernetes

Izazovi: OpenShift ima vlastite resurse (Routes, DeploymentConfigs, BuildConfigs, SCC sigurnosne politike) koji ne postoje u vanilla k8s; gubiš integrirane alate; moraš sve to prepisati u standardne k8s resurse (Ingress, Deployment) i sam riješiti ono što je OpenShift davao "besplatno". Korist: manje vendor lock-ina, više fleksibilnosti.

## 6. CI/CD agenti (Jenkins/GitLab/Tekton) u k8s vs na dedicated VM-ovima

- **U Kubernetesu** — agenti se dižu na zahtjev (autoscaling), bolja iskorištenost resursa, izolacija po podu. Rizik **noisy-neighbor** (jedan build troši sve resurse) ublažava se resource limitima.
- **Na VM-ovima** — jača izolacija, ali statički kapacitet (plaćaš i kad ne radiš), nema auto-skaliranja.

## 7. Kubernetes i cloud / dinamičko provisioniranje

K8s se integrira s cloud providerima: **cluster autoscaler** dodaje/uklanja čvorove prema potražnji, **LoadBalancer** servisi traže cloud LB, cloud storage se priključuje preko CSI. Tako kapacitet raste i pada automatski s prometom.

## 8. Sigurnost: OpenShift vs vanilla Kubernetes

- **OpenShift** — sigurniji "out of the box": po defaultu kontejneri ne smiju kao root, **SCC** (Security Context Constraints), integrirani RBAC, ugrađeni registry. Zato ga regulirane industrije (banke, zdravstvo) često biraju — manje truda za usklađenost (compliance).
- **Vanilla k8s** — sigurnost moraš sam posložiti (PodSecurity standardi, NetworkPolicy, RBAC).

## 9. Krivulja učenja: OpenShift vs plain Kubernetes

- **OpenShift** — dodatna apstrakcija olakšava timovima **novima u kontejnerima** (web konzola, S2I, gotove zadane vrijednosti), ali ti slojevi mogu zbuniti kad treba zaviriti "ispod haube".
- **Plain k8s** — strmija početna krivulja, ali bolje razumiješ što se događa.

## 10. Full Kubernetes vs k3s (mali on-prem)

- **k3s** — lagana distribucija k8s (jedan binarni file, mali memorijski otisak), idealna za rub mreže / male on-prem / IoT.
- **Full k8s** — veći otisak resursa, opravdano tek kad imaš puno čvorova i složene potrebe. Za malu instalaciju je **overkill**.

## 11. Docker Compose na jednom hostu vs odmah Kubernetes (mali tim)

- **Compose** — jednostavno, brzo, dovoljno za jedan host i malu aplikaciju; nema otpornosti (host padne = sve padne) ni skaliranja preko više računala.
- **Kubernetes** — otpornost i skaliranje, ali velik overhead za mali tim.
- **Obrana:** mali tim počinje s Composeom (manja složenost), prelazi na k8s tek kad zatreba skaliranje/visoka dostupnost.

## 12. Vendor lock-in: Kubernetes (CNCF, open source) vs OpenShift (Red Hat)

- **k8s** — open source, CNCF, radi na bilo kojem cloudu → manji lock-in, veća dugoročna fleksibilnost.
- **OpenShift** — Red Hatov proizvod s podrškom i integriranim alatima, ali vežeš se uz njihov ekosustav/licenciranje.

## 13. Obrana OpenShifta za tvrtku koja želi integriranu, podržanu platformu

Argumenti za: sve na jednom mjestu (CI/CD, registry, monitoring, dev konzola), komercijalna podrška, sigurnost po defaultu, manje vremena na slaganju alata → tim se fokusira na proizvod, ne na infrastrukturu.

## 14. Storage: Kubernetes vs klasični VM workloadi

- **VM** — disk je obično fiksno vezan uz VM.
- **Kubernetes** — spremište je apstrahirano (PVC/StorageClass), dinamički se dodjeljuje i može se "seliti" s Podom, odvojeno od životnog ciklusa kontejnera.

## 15. Startup s 2 inženjera, brzo i jeftino

Preporuka: **Compose na jednom hostu** ili **managed/lagano rješenje** (k3s ili managed k8s) — nizak operativni overhead, malo troškova. Full self-managed k8s bi pojeo premalen tim na održavanje.

## 16. Docker vs Podman — arhitektura i rootless sigurnost

- **Docker** — **daemon** arhitektura: centralni proces `dockerd` radi kao root; ako ga netko kompromitira, ima root nad hostom.
- **Podman** — **daemonless**: nema pozadinskog demona, svaki kontejner je običan proces korisnika. Podržava **rootless** (kontejneri bez root prava). Zato ga sigurnosno svjesni timovi često preferiraju — manja napadna površina, nema privilegiranog demona.

## 17. Day-2 overhead: self-managed k8s vs managed k8s servis

- **Self-managed** — sam radiš upgrade clustera, skaliranje čvorova, backupe, zakrpe → puno posla.
- **Managed** (npr. cloud k8s) — provider preuzima upravljanje control planeom, upgradee i dio operacija → manje operativnog tereta, ali manje kontrole i veći trošak.

## 18. Media-streaming tvrtka, nagli i nepredvidiv globalni promet

Preporuka: **Kubernetes na cloudu** s **autoscalingom** (HPA za Podove + cluster autoscaler za čvorove) i globalnom distribucijom (više regija, CDN). K8s se nosi sa spiky prometom dizanjem/spuštanjem kapaciteta automatski.

## 19. Izrastao iz Docker Swarma / jednog Compose hosta → migrirati na k8s?

Vagaš: trud migracije (preпisivanje manifesta, učenje, novi alati) protiv dugoročne koristi (skaliranje, otpornost, golem ekosustav, podrška zajednice). Ako rast i potreba za pouzdanošću rastu — migracija se isplati.

## 20.–23. "Bi li odabrao Kubernetes / OpenShift za mikroservise / enterprise?"

Ovo su argumentacijska pitanja — zauzmi stav i potkrijepi ga. Ključni argumenti koje možeš koristiti:

**Kubernetes ZA mikroservise** — orkestracija velikog broja servisa, self-healing, service discovery, golem ekosustav (Helm, Istio, Prometheus), otpornost. Protuargument: složenost.

**Kubernetes ZA enterprise** — skalabilnost, ogromna zajednica i podrška, bogatstvo značajki, radi na svakom cloudu. Protuargument: treba jaki tim i day-2 operacije.

**OpenShift ZA mikroservise** — sve od k8s + integrirani CI/CD, S2I, sigurnost po defaultu, jedinstvena platforma; brži time-to-market. Protuargument: lock-in, cijena, dodatni slojevi.

**OpenShift ZA enterprise** — komercijalna podrška, certificiran za regulirane industrije, integrirani alati i sigurnost, predvidiva podrška Red Hata. Protuargument: licenca, vezanost uz vendora, veći resursni otisak.

> **Savjet za ispit:** kod ovih pitanja uvijek (1) zauzmi jasan stav, (2) navedi 2–3 konkretna argumenta ZA, (3) priznaj 1 protuargument / kompromis. To pokazuje da razumiješ, a ne samo recitiraš.

---

## Brzi rječnik pojmova

- **Kontejner** — izolirani proces s vlastitim filesystemom; lagana alternativa VM-u.
- **Slika (image)** — "kalup" iz kojeg se stvara kontejner.
- **Pod** — najmanja k8s jedinica; 1+ kontejnera koji dijele mrežu.
- **Deployment** — upravlja stateless Podovima, omogućuje rolling update/rollback.
- **StatefulSet** — za stateful aplikacije, stabilna imena + vlastiti PVC po replici.
- **Service** — stabilna adresa + load balancing za Podove.
- **Volume / PVC** — trajno spremište odvojeno od životnog vijeka kontejnera.
- **ConfigMap / Secret** — konfiguracija / povjerljivi podaci (Secret = base64, NE šifriran).
- **Namespace** — logička particija unutar clustera (ili Linux mehanizam izolacije, ovisno o kontekstu).
- **Orkestracija** — automatsko upravljanje životnim ciklusom mnogo kontejnera.

---

# Dodatak za LO6: Docker Swarm i k3s (potvrđeno na ispitu)

LO6 izričito uključuje i pregled **Docker Swarma** i **k3s**. Evo što trebaš znati o njima kao orkestratorima, uz Kubernetes.

## Docker Swarm

Swarmov je Dockerov **vlastiti, ugrađeni orkestrator**. Filozofija: jednostavnost.

- **Arhitektura:** više čvorova udruženo u "swarm" → **manager** čvorovi (donose odluke) i **worker** čvorovi (vrte kontejnere).
- **Deploy:** umjesto pojedinačnih kontejnera deklariraš **service** (npr. 3 replike nginxa) preko `docker service create`. Za cijeli stack koristiš **compose datoteku** i `docker stack deploy` — pa ako znaš Compose, već znaš 80% Swarma.
- **Ugrađeno:** load balancing (routing mesh), service discovery po imenu, rolling update, secrets — slično k8s, ali jednostavnije.
- **Prednosti:** vrlo niska krivulja učenja, malo overheada, brzo postavljanje.
- **Mane:** puno manji ekosustav i manje značajki od k8s; industrijski je uvelike **potisnut Kubernetesom** (praktički u "održavanju"). 
- **Kad ima smisla:** mali/srednji klasteri gdje je jednostavnost važnija od bogatstva značajki.

## k3s

k3s je **lagana, certificirana distribucija Kubernetesa** (Rancher/SUSE). Bitno: **to JEST Kubernetes** — isti `kubectl`, isti API — samo "smršavljen".

- **Lagan:** jedan binarni file (~malo desetaka MB), mali memorijski otisak; po defaultu koristi SQLite umjesto teškog etcd-a, izbacuje zastarjele/nepotrebne komponente.
- **Instalacija:** doslovno jedna `curl | sh` naredba.
- **Prednosti:** vrti se na rubu mreže, IoT, Raspberry Pi, malim VM-ovima; puna kompatibilnost s k8s API-jem (tvoji manifesti rade isto).
- **Mane:** neki kompromisi na vrlo velikoj skali.
- **Kad ima smisla:** edge/IoT, razvoj, mali on-prem — gdje je **puni Kubernetes overkill**.

## Sažetak (tri orkestratora)

- **Docker Swarm** — najjednostavniji, Docker-native, ali u opadanju.
- **k3s** — pravi Kubernetes, samo lagan; za male/edge instalacije.
- **Puni Kubernetes** — najmoćniji i najsloženiji; za velike, ozbiljne sustave.

---

# Strategija za ispit (logistika i dokazivanje)

Ispit je na VM-u u Algebra cloudu (pristup preko VPN-a); sve radiš unutar te VM. Ključno za bodove:

## Što je dopušteno i kako to iskoristiti

- **Dokumentacija je otvorena** (docs.podman.io, kubernetes.io, minikube docs, docs.docker.com, registry.access.redhat.com). Ne pamti svaki flag — razumij koncept, a sintaksu brzo nađi. Vježbaj i samo pretraživanje docsa.
- **Vlastite GitHub bilješke su dopuštene.** Napravi uredan repo s ovom skriptom + gotovim naredbama koje samo kopiraš. Najveća prednost na ispitu.
- **AI nije dopušten** tijekom ispita. Oslanjaš se na bilješke + docs.

## Screenshot = dokaz (najvažnije!)

Ispit traži screenshotove rezultata. Za svaki zadatak moraš pokazati naredbu koja **dokazuje** da je odrađen. Zbirka "go-to" verifikacijskih naredbi:

**Kontejneri (LO1–LO3):**
```bash
podman ps                              # radi li kontejner, portovi, status
podman ps --filter "label=..."         # filtriranje po labeli
podman inspect <c>                      # sve o kontejneru
podman inspect -f '{{.NetworkSettings.IPAddress}}' <c>   # izvuci jedno polje
podman logs <c>                         # zašto je pao / izlaz
podman stats --no-stream <c>            # CPU/RAM (dokaz limita)
podman top <c>                          # procesi unutra
podman port <c>                         # objavljeni portovi
podman exec <c> id                       # dokaz --user / non-root
podman exec <c> env                      # dokaz env varijabli
podman network inspect <net>            # IP-evi, koji su kontejneri spojeni
podman volume ls                         # postoji li volumen
podman pod inspect <pod>                # koji namespacei se dijele
podman diff <c>                          # A/C/D promjene u filesystemu
```

**Compose:**
```bash
podman compose ps                        # svi servisi i status
podman compose logs                      # logovi svih servisa
```

**Kubernetes (LO4):**
```bash
kubectl get pods,deploy,svc,pvc,endpoints   # brzi pregled stanja
kubectl describe pod <p>                     # Events = uzrok problema
kubectl logs <p> --previous                  # zadnji crash
kubectl rollout status deployment/<d>        # je li update prošao
kubectl rollout history deployment/<d>       # revizije
kubectl get pod <p> -o yaml                  # puni spec (OOMKilled, limiti)
kubectl get pod <p> -o jsonpath='{.status.containerStatuses[0].restartCount}'
kubectl get endpoints <svc>                  # je li Service spojen na Podove
```

## Pripremi unaprijed

- Računi: **hub.docker.com** i **quay.io** (bez 2FA, znaj lozinke). Znaj i **Red Hat Academy** lozinku.
- **VPN** po "Environment access instructions" — postavi i testiraj prije ispita.
- Kubernetes vježbaj s **minikube + kubectl** (ne OpenShift `oc`), jer ispitna VM koristi minikube.

---

# Format ispita i predaja odgovora (na temelju prošlog kolokvija)

Stvarni kolokvij (LO1–LO3) pokazao je točan oblik zadataka. Final (LO4–LO6) gotovo sigurno ima isti format.

**Bodovanje i vrijeme:** svaki LO ≈ 12 bodova / 30 min; pojedini zadatak ≈ 4 boda (poneki 8). Ispit je **open book** (smiješ dokumentaciju i svoje GitHub bilješke).

**Dvije težine zadataka:**
- **`_M` (osnovni)** — samo napravi i pokaži da radi.
- **`_D` (teži)** — napravi **i objasni/demonstriraj** (npr. "objasni kako si pronašao port", "dokaži da kontejner može pisati u mount").
- Praktično: kod `_D` zadatka uvijek dodaj rečenicu-dvije objašnjenja, ne samo naredbu.

**Zadaci su vrlo konkretni** — točna slika, točni portovi, točna imena, točne env varijable. Čitaj doslovno i ispuni baš to što traže (npr. "port 8080 kontejnera → 9080 čvora", "tag `firstcontainer:1`", "spremi u `filec.tar`").

**Personalizacija (protiv prepisivanja):** zadaci znaju tražiti tvoje ime/prezime — npr. env varijabla `STUDENT`, tekst na web stranici, ime resursa. Pazi na to i ubaci svoje podatke gdje traže.

**Stack = obrazac, ne aplikacija:** prošli put je bio Drupal+PostgreSQL, a practice traži druge parove (Ghost/Gitea/Redmine). Uče te **obrazac** (app + baza koja komunicira po imenu + perzistencija), neovisno o aplikaciji.

**Docker Hub limit MORAŠ riješiti:** ako te zaustavi `toomanyrequests`, obavezno se autenticiraj (`podman login docker.io` ili docker-registry Secret u k8s). Zadatak neriješen zbog limita se **ne priznaje**.

## Kako se predaje (vrlo važno za bodove)

- **Jedna datoteka**, naziv: `<infoeduka_username>_group_<grupa>` (markdown / Word / LibreOffice / WordPad).
- Svaki zadatak označi oznakom `Lo<outcome>_<broj>` (npr. `Lo4_3`).
- Za svaki zadatak stavi: (1) **naredbu / compose / YAML** koju si koristio, i (2) **screenshot da radi**.
- Za web aplikacije: screenshot **UI-a u pregledniku** (dokaz da app stvarno odgovara). Zato u k8s trebaš NodePort + `minikube service <svc> --url` da dobiješ URL koji možeš otvoriti i screenshotati.
- Screenshotovi se rade unutar ispitne VM: Applications > Utilities > Screenshot.
- Upload u mapu "Intro to DevOps" na results.vua.cloud.

## Predložak odgovora (kopiraj ovaj oblik za svaki zadatak)
```
## Lo4_3 — Scale web na 5 replika

Naredba:
kubectl scale deployment web --replicas=5

Dokaz:
[screenshot: kubectl get pods pokazuje 5 Podova u Running]

Objašnjenje (obavezno kod _D zadataka):
Skaliranje mijenja broj replika na ISTOM ReplicaSetu — ne stvara novi.
```

## Sitnice koje mogu spasiti bodove
- Ako `podman compose` javi grešku o provideru → instaliraj: `python3 -m pip install --user podman-compose` (ili `sudo dnf install -y podman-compose`).
- Tvrdoglave/skalirane stackove čistiš s `podman pod rm -f -a`.
- Prije `kubectl apply` provjeri YAML: `kubectl apply --dry-run=server --validate=true -f <datoteka>`.
- Uvijek priloži dokaznu naredbu uz svaki zadatak — bez screenshota dokaza, zadatak je nepotpun.
