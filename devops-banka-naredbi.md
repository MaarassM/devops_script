# Intro to DevOps — BANKA NAREDBI (svaka practice question → točna naredba)

> Ovo je operativni dodatak skripti: za **svaku** practice question gotova naredba/YAML koju kopiraš.
> Skripta objašnjava *zašto*; ovdje je *kako*, korak po korak.
> Zamijeni `Horvat` / `Ivan Horvat` svojim podacima. `podman` = `docker`, `Containerfile` = `Dockerfile`.

---

# 0. TEMELJI (ono što fali, a treba na svakom koraku)

## 0.1 Linux osnovne naredbe
```bash
sudo <naredba>                 # pokreni kao root (treba za pisanje izvan home-a, npr. u /)
mkdir -p /putanja              # napravi direktorij (+ roditelje, bez greške ako postoji)
chmod 777 /putanja             # dozvole: svi čitaju/pišu/izvršavaju (gruba palica za demo)
chown -R $(id -u):$(id -g) /p  # promijeni vlasnika na tebe (finije od chmod 777)
ls -l /putanja                 # popis + dozvole (DOKAZ da je datoteka nastala)
cat /putanja/file              # ispiši sadržaj datoteke (DOKAZ upisa)
touch /putanja/file            # napravi praznu datoteku (test pisanja)
rm -rf /putanja                # obriši (OPREZ)
pwd                            # gdje se trenutno nalazim
cd /putanja                    # promijeni direktorij
echo "tekst" > file            # upiši tekst u datoteku (prepiše)
echo "tekst" >> file           # dopiši na kraj datoteke
```

## 0.2 Kako napraviti datoteku iz YAML/compose bloka (DVA načina)

**A) Editor (nano) — preglednije:**
```bash
nano app-pod.yaml
# zalijepi sadržaj, pa: Ctrl+O, Enter (spremi), Ctrl+X (izlaz)
```
(vi/vim: `vi file.yaml` → `i` → zalijepi → `Esc` → `:wq` → Enter)

**B) Heredoc — zalijepiš cijeli blok odjednom:**
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
EOF
```
> `cat > file << 'EOF'` = upiši sve do retka `EOF` u datoteku. YAML koristi **razmake, nikad tabove**.

Zatim primijeniš:
```bash
kubectl apply -f app-pod.yaml
```

## 0.3 ⭐ Generiranje YAML kostura (NAJVAŽNIJI trik za k8s)
Ne piši YAML iz glave — neka ga `kubectl` generira, pa samo dotjeraj:
```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3 --dry-run=client -o yaml > web.yaml
kubectl run mypod --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl expose deployment web --port=80 --target-port=80 --dry-run=client -o yaml > svc.yaml
kubectl create configmap cm --from-literal=K=V --dry-run=client -o yaml > cm.yaml
kubectl create secret generic s --from-literal=K=V --dry-run=client -o yaml > sec.yaml
kubectl create job j --image=busybox --dry-run=client -o yaml -- echo hi > job.yaml
kubectl create cronjob c --image=busybox --schedule="* * * * *" --dry-run=client -o yaml -- date > cron.yaml
```
> `--dry-run=client` = ne dira cluster, samo ispiše. `-o yaml` = u YAML formatu. `>` = spremi u datoteku.
> NE radi za StatefulSet/PV/DaemonSet/initContainer — njih pišeš ručno (predlošci u dijelu LO4).

Provjera prije primjene (uhvati typo/indentaciju):
```bash
kubectl apply --dry-run=server --validate=true -f web.yaml
```

## 0.4 SELinux i bind mount (Red Hat) — `:Z` i kad zakaže
Bind mount na Red Hatu treba SELinux oznaku. Dodaj `:Z` na mount:
```bash
podman run -d -v /host/dir:/cont/path:Z <slika>
```
> `:Z` = privatni label za taj kontejner | `:z` (malo) = dijeljeni label za više kontejnera.

**Ako `:Z` na root-owned mapi padne s `lsetxattr ... operation not permitted`** (rootless podman ne smije relabelirati `/drupal_data`): postavi oznaku ručno kao root i montiraj BEZ `:Z`:
```bash
sudo mkdir -p /drupal_data
sudo chmod 777 /drupal_data
sudo chcon -Rt container_file_t /drupal_data     # ručni SELinux label
podman run -d -v /drupal_data:/cont/path <slika>  # bez :Z
```
> Zaobilazno: stavi mapu u home (`~/data`) gdje `:Z` radi bez sudo-a.

## 0.5 Računi i pull limit (PRIJE svega)
```bash
podman login docker.io          # riješi toomanyrequests (anonimni limit)
podman login quay.io
# k8s: docker-registry Secret + imagePullSecrets (vidi LO4 #2, #36)
```

---

# LO1 — Kontejneri i container servisi (23 pitanja)

```bash
# 1. httpd detached, port 80->8081, custom name+hostname, dokaz inspect
podman run -d --name web --hostname mojweb -p 8081:80 docker.io/library/httpd
podman inspect -f '{{.Name}} {{.Config.Hostname}}' web

# 2. --restart=always, ubij glavni proces, dokaži restart
podman run -d --name r --restart=always docker.io/library/httpd
podman exec r pkill httpd        # ili: podman kill r
podman ps                        # restart count raste / opet Up
podman inspect -f '{{.RestartCount}}' r

# 3. limiti memorije i CPU-a, verify
podman run -d --name lim --memory=256m --cpus=0.5 docker.io/library/httpd
podman stats --no-stream lim
podman inspect -f '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}' lim

# 4. --env vs --env-file
podman run -d --name e1 --env KEY=VALUE docker.io/library/httpd
printf 'A=1\nB=2\n' > vars.env
podman run -d --name e2 --env-file vars.env docker.io/library/httpd
podman exec e1 env ; podman exec e2 env

# 5. user-defined mreža, attach, nađi IP
podman network create mreza1
podman run -d --name c --network mreza1 docker.io/library/httpd
podman network inspect mreza1
podman inspect -f '{{.NetworkSettings.Networks.mreza1.IPAddress}}' c

# 6. pause / unpause (procesi su zamrznuti u memoriji, ne ubijeni)
podman pause c ; podman unpause c

# 7. rename
podman rename c novi_c

# 8. logs flagovi
podman logs --tail 20 --since 10m -f web

# 9. inspect jedno polje (Go template)
podman inspect -f '{{.NetworkSettings.IPAddress}}' web
podman inspect -f '{{.RestartCount}}' web

# 10. cp host<->container
podman cp ./file.txt web:/tmp/file.txt
podman cp web:/tmp/file.txt ./izvuceno.txt

# 11. exec shell, instaliraj paket (NE preživi recreate)
podman exec -it web bash
#  (unutra) apt-get update && apt-get install -y curl   # nestane kad recreate-aš kontejner

# 12. top + stats
podman top web ; podman stats --no-stream web

# 13. --rm (obriše se po završetku — gubiš logove)
podman run --rm docker.io/library/alpine echo hello

# 14. bind-mount read-only, dokaži da pisanje pada
mkdir -p ./ro && echo hi > ./ro/f
podman run --rm -v ./ro:/data:ro docker.io/library/alpine sh -c "echo x > /data/new"  # Permission denied

# 15. labele + filter
podman run -d --name L1 --label env=prod docker.io/library/httpd
podman run -d --name L2 --label env=dev docker.io/library/httpd
podman ps --filter "label=env=prod"

# 16. commit izmijenjenog kontejnera u sliku
podman commit web mojaslika:1.0
podman run -d --name iz_commita mojaslika:1.0

# 17. diff (A=dodano, C=promijenjeno, D=obrisano)
podman diff web

# 18. --network none (bez mreže)
podman run --rm --network none docker.io/library/alpine ping -c1 8.8.8.8   # ne radi

# 19. port vezan samo na jednu IP
podman run -d --name onlylocal -p 127.0.0.1:8082:80 docker.io/library/httpd

# 20. port + inspect
podman port web
podman inspect -f '{{.NetworkSettings.Ports}}' web

# 21. ne-root korisnik, dokaži UID
podman run --rm --user 1001 docker.io/library/alpine id

# 22. systemd / Quadlet (start na boot)
podman generate systemd --new --name web > web.service
# ili Quadlet: ~/.config/containers/systemd/web.container -> systemctl --user daemon-reload

# 23. events uživo (u drugom terminalu start/stop)
podman events
```

---

# LO2 — Slike i Containerfile (23 pitanja)

```dockerfile
# 1. ARG za base tag
FROM alpine:${TAG}
ARG TAG=3.20
```
```bash
podman build --build-arg TAG=3.19 -t a:319 .
podman build --build-arg TAG=3.20 -t a:320 .
```
```dockerfile
# 2. multi-stage (usporedi veličinu)
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN go build -o app .
FROM alpine:3.20
COPY --from=build /src/app /app
CMD ["/app"]
```
```bash
podman images   # finalna slika je par MB vs ~1GB single-stage
```
```bash
# 3. .containerignore (kao .gitignore za build kontekst)
echo "secret.txt" > .containerignore

# 4. tri taga odjednom + konvencija registry/namespace/ime:tag
podman build -t app:1.0 -t app:1 -t app:latest .

# 5. history + smanji slojeve
podman history app:1.0

# 10. save -> rm -> load (air-gap)
podman save -o app.tar app:1.0
podman rmi app:1.0
podman load -i app.tar

# 11. pull po digestu (reproducibilno)
podman pull docker.io/library/nginx@sha256:<DIGEST>

# 12. --no-cache
podman build --no-cache -t app:nocache .

# 13. ne-default ime Containerfilea
podman build -f Containerfile.dev -t app:dev .

# 16. login + push u privatni repo (kredencijali u auth.json)
podman login quay.io
podman tag app:1.0 quay.io/<user>/app:1.0
podman push quay.io/<user>/app:1.0

# 17. prune dangling + rmi
podman image prune        # makni "dangling" (bez taga, zaostali slojevi)
podman rmi app:1.0

# 20. inspect nepoznate slike (entrypoint/cmd/ports/env)
podman image inspect docker.io/library/redis

# 22. tag+push u dva registra (mirroring)
podman tag app:1.0 docker.io/<user>/app:1.0 && podman push docker.io/<user>/app:1.0
podman tag app:1.0 quay.io/<user>/app:1.0 && podman push quay.io/<user>/app:1.0

# 23. popis OS paketa u slici (vrijednost minimalne baze)
podman run --rm docker.io/library/ubuntu dpkg -l
```
```dockerfile
# 6. ne-root USER + writable WORKDIR
FROM alpine:3.20
RUN adduser -D appuser && mkdir /data && chown appuser /data
USER appuser
WORKDIR /data
# whoami -> appuser

# 7. HEALTHCHECK
FROM nginx
HEALTHCHECK --interval=5s --retries=3 CMD curl -f http://localhost/ || exit 1
# podman ps -> kolona STATUS (healthy); ručno: podman healthcheck run <c>

# 8. ENTRYPOINT vs CMD (override CMD pri run-u)
FROM alpine
ENTRYPOINT ["echo"]
CMD ["zadano"]
# podman run slika  -> "zadano" ; podman run slika novo -> "novo"

# 9. ADD vs COPY (ADD raspakira tar/skida URL; inače COPY)
COPY app.py /app/
ADD arhiva.tar.gz /app/    # auto-raspakira

# 14. COPY --chown
COPY --chown=appuser:appuser app.py /app/

# 15. ARG (build-time) vs ENV (run-time)
ARG VERZIJA=1.0
ENV APP_VERZIJA=$VERZIJA

# 18. statički file server
FROM python:3.11
WORKDIR /web
COPY . /web
EXPOSE 8000
CMD ["python", "-m", "http.server", "8000"]

# 19. pinned verzija paketa
RUN apt-get update && apt-get install -y nginx=1.24.* && rm -rf /var/lib/apt/lists/*

# 21. multi-line RUN + cleanup u ISTOM layeru
RUN apt-get update && apt-get install -y build-essential && rm -rf /var/lib/apt/lists/*
```

---

# LO3 — Isporuka, mreže, sigurnost (23 pitanja, podman)

```bash
# 1. pod (kontejneri se vide preko localhost)
podman pod create --name p -p 8080:80
podman run -d --pod p --name app docker.io/library/httpd
podman run -d --pod p --name db docker.io/library/postgres:16 \
  -e POSTGRES_PASSWORD=pw       # app doseže bazu na localhost:5432

# 2. user-defined mreža, app+postgres, razlučivanje po imenu
podman network create net
podman run -d --network net --name db -e POSTGRES_PASSWORD=pw docker.io/library/postgres:16
podman run -d --network net --name app docker.io/library/httpd
podman exec app getent hosts db    # ime "db" -> IP (DNS)

# 3. two-tier (NE Drupal) — npr. Gitea + PostgreSQL
podman network create gitea-net
podman volume create giteadb
podman run -d --name db --network gitea-net -v giteadb:/var/lib/postgresql/data \
  -e POSTGRES_USER=gitea -e POSTGRES_PASSWORD=gitea -e POSTGRES_DB=gitea docker.io/library/postgres:16
podman run -d --name gitea --network gitea-net -p 3000:3000 \
  -e GITEA__database__DB_TYPE=postgres -e GITEA__database__HOST=db:5432 \
  -e GITEA__database__NAME=gitea -e GITEA__database__USER=gitea -e GITEA__database__PASSWD=gitea \
  docker.io/gitea/gitea:latest
# otvori http://localhost:3000 i prođi first-run setup

# 4. perzistencija named volume (preživi rm)
podman run -d --name db2 -v dbdata:/var/lib/postgresql/data -e POSTGRES_PASSWORD=pw docker.io/library/postgres:16
podman rm -f db2
podman run -d --name db2 -v dbdata:/var/lib/postgresql/data -e POSTGRES_PASSWORD=pw docker.io/library/postgres:16
podman exec db2 psql -U postgres -c "\l"   # podaci tu

# 5. secret umjesto --env
printf 'tajna123' | podman secret create db_pass -
podman run -d --name s --secret db_pass docker.io/library/postgres:16 \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/db_pass     # tajna kao datoteka, ne vidi se u inspect

# 6. compose up
podman compose up -d
podman compose ps

# 7. db na internoj mreži (skrivena), samo app izložena — compose.yaml:
#   networks: backend: {internal: true}; db bez ports:; gitea ports: + obje mreže
curl -s --max-time 3 localhost:5432; echo "<- db nedostupna s hosta"

# 8. healthcheck + depends_on (compose.yaml):
#   db.healthcheck.test: ["CMD-SHELL","pg_isready -U gitea"]
#   gitea.depends_on: db: {condition: service_healthy}

# 9. env_file u compose:  gitea.env_file: .env

# 10. skaliranje
podman compose up -d --scale app=3

# 11. bind mount (config) + named volume (data) u istom kontejneru
podman run -d --name mix -v ./conf:/etc/app:Z -v appdata:/var/lib/app docker.io/library/httpd

# 12. backup volumena u tar + restore
podman run --rm -v dbdata:/data -v $(pwd):/backup docker.io/library/alpine tar czf /backup/b.tar.gz -C /data .
podman volume create dbdata2
podman run --rm -v dbdata2:/data -v $(pwd):/backup docker.io/library/alpine tar xzf /backup/b.tar.gz -C /data

# 13. nginx reverse proxy ispred app (ista mreža; nginx prosljeđuje na app:80)
podman run -d --network net --name app docker.io/library/httpd
podman run -d --network net --name proxy -p 80:80 -v ./nginx.conf:/etc/nginx/nginx.conf:Z docker.io/library/nginx
# nginx.conf:  proxy_pass http://app:80;

# 14. db-admin (adminer) na istu mrežu
podman run -d --network net --name adminer -p 8081:8080 docker.io/library/adminer

# 15. generate kube iz poda
podman generate kube p > pod.yaml

# 16. kube play + down
podman kube play pod.yaml
podman kube play --down pod.yaml

# 17. CPU/mem limiti po servisu (compose) + provjera
#   service.deploy.resources.limits: {cpus: "0.5", memory: 256M}   (ili mem_limit)
podman stats --no-stream

# 18. pod namespaces (dijele network/IPC, NE PID po defaultu)
podman pod inspect p

# 19. razlučivanje po imenu pada na različitim mrežama -> spoji na zajedničku
podman network connect net app    # dodaj kontejner na mrežu

# 20. jedan kontejner na dvije mreže (frontend+backend) — segmentacija
podman network create frontend && podman network create backend
podman run -d --name app --network backend docker.io/library/httpd
podman network connect frontend app

# 21. compose troubleshoot
podman compose logs ; podman compose ps

# 22. dijeljeni named volume kroz replike (rizik: istovremeno pisanje = korupcija)
#   isti -v sharedvol:/data na više kontejnera

# 23. compose down (čuva volumene) vs --volumes (briše)
podman compose down
podman compose down --volumes
```

---

# LO4 — Kubernetes (53 pitanja)

## Preduvjet
```bash
minikube status || minikube start    # uvijek prvo! "dial tcp ... i/o timeout" = cluster ne radi
```

## Deploymenti, skaliranje, rollout (1–12)
```bash
# 1. deployment + izvoz YAML
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl get deployment web -o yaml > web.yaml

# 2. DockerHub authenticated pull -> resurs = docker-registry Secret (+ imagePullSecrets)
kubectl create secret docker-registry dockerhub \
  --docker-server=docker.io --docker-username=<USER> --docker-password=<PASS>

# 3. scale dva načina + RS
kubectl scale deployment web --replicas=5            # način 1
kubectl edit deployment web                          # način 2: replicas: 5 (ili apply uređeni yaml)
kubectl get rs

# 4. rolling update + status
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web

# 5. history + rollback + CHANGE-CAUSE
kubectl annotate deployment/web kubernetes.io/change-cause="update na 1.27"
kubectl rollout history deployment/web
kubectl rollout undo deployment/web

# 8. requests/limits -> verify
kubectl set resources deployment web --requests=cpu=100m,memory=128Mi --limits=cpu=200m,memory=256Mi
kubectl get pod -l app=web -o jsonpath='{.items[0].spec.containers[0].resources}'

# 10. nepostojeći tag -> rollout zaglavi, stari p"odovi serviraju
kubectl set image deployment/web nginx=nginx:9.99
kubectl rollout status deployment/web    # zaglavi; stari podovi i dalje rade

# 11. label selector + Deployment->RS->Pod
kubectl get pods -l app=web

# 12. expose (stvara Service, selektor iz deployment labela)
kubectl expose deployment web --port=80 --target-port=80
```
YAML-only stavke (generiraj kostur s `--dry-run` pa dodaj):
```yaml
# 6. strategija RollingUpdate (spec:)
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
# 7. strategija Recreate (kad stara+nova ne smiju raditi zajedno)
  strategy:
    type: Recreate
# 9. revisionHistoryLimit (spec:)
  revisionHistoryLimit: 3
```

## Pod-level: sidecar, nodeSelector, bare pod (13–15)
```yaml
# 13. sidecar (drugi kontejner u istom podu — dijele network/volume)
    spec:
      containers:
        - name: web
          image: nginx
        - name: sidecar
          image: busybox
          command: ["sh","-c","while true; do date; sleep 5; done"]
# 14. httpd:2.4, 2 replike, named port, nodeSelector (Pending ako nema čvora)
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
```bash
# 15. bare Pod (obrisan = nestaje; Deployment-pod se ponovo digne)
kubectl run solo --image=busybox --restart=Never -- sleep 1d
```

## Controlleri: StatefulSet, DaemonSet, Job, CronJob (16–21)
```yaml
# 16/21/27/41. StatefulSet redis + headless Service
apiVersion: v1
kind: Service
metadata: {name: redis}
spec:
  clusterIP: None          # headless -> A-zapis po podu
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
# pods: redis-0/1/2 (stabilna imena, redom); svaka replika svoj PVC
```
```yaml
# 17. DaemonSet (jedan pod po čvoru)
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
```bash
# 18. Job (izračun jednom, COMPLETIONS, logovi)
kubectl create job izracun --image=busybox -- sh -c "echo $((6*7))"
kubectl get job izracun         # COMPLETIONS 1/1
kubectl logs job/izracun        # 42

# 19. CronJob (svake minute), suspend, popis Jobova
kubectl create cronjob datum --image=busybox --schedule="* * * * *" -- date
kubectl patch cronjob datum -p '{"spec":{"suspend":true}}'
kubectl get jobs

# 20. ručno okini Job iz CronJoba
kubectl create job rucno --from=cronjob/datum
```

## Volumeni: emptyDir, ConfigMap, Secret, PVC, initContainer (22–37)
```bash
# 23. popis storagea
kubectl get storageclass,pv,pvc

# 24/26/35/37. ConfigMap/Secret create
kubectl create configmap app-config --from-literal=OWNER="Ivan Horvat"
kubectl create secret generic db-cred --from-literal=DB_USER=appuser --from-literal=DB_PASSWORD=Horvat

# 31. dokaz da je Secret base64 (NE šifriran)
kubectl get secret db-cred -o jsonpath='{.data.DB_USER}' | base64 -d   # appuser

# 32/33. Secret iz datoteka / TLS
kubectl create secret generic certs --from-file=tls.crt --from-file=tls.key
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key

# 36. docker-registry secret (vidi #2) -> referenciraš imagePullSecrets u podu
```
```yaml
# 22/24/25/26/28/29/30/34/37 — pod s svim tipovima mountova
apiVersion: v1
kind: Pod
metadata: {name: vol}
spec:
  initContainers:                       # 28. init puni shared volume prije glavnog
    - name: init
      image: busybox
      command: ["sh","-c","echo hi > /shared/f"]
      volumeMounts: [{name: shared, mountPath: /shared}]
  containers:
    - name: app
      image: nginx
      envFrom:                          # 35. SVI ključevi Secreta kao env
        - secretRef: {name: db-cred}
      volumeMounts:
        - {name: cfg, mountPath: /etc/config}              # 24. ConfigMap kao volume (ključ=datoteka)
        - {name: cfg, mountPath: /etc/one.conf, subPath: OWNER}  # 25. samo jedan ključ na path
        - {name: sec, mountPath: /etc/secret, readOnly: true}    # 26/29/34. readOnly, restriktivno
        - {name: shared, mountPath: /shared}                     # 22. emptyDir dijeljen
  volumes:
    - {name: cfg, configMap: {name: app-config}}
    - name: sec
      secret: {secretName: db-cred, defaultMode: 0400}    # 26. restriktivne dozvole
    - name: shared
      emptyDir: {sizeLimit: 100Mi}                        # 30. sizeLimit
```
```yaml
# 27. PVC (StatefulSet stvara po jedan po replici; ostaju nakon brisanja STS)
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: data-pvc}
spec:
  accessModes: ["ReadWriteOnce"]
  resources: {requests: {storage: 1Gi}}
```

## Servisi + DNS (38–49)
```bash
# 38. ClusterIP + DNS iz drugog poda
kubectl expose deployment web --name=web-svc --port=80 --target-port=80
kubectl run tmp --rm -it --image=busybox -- sh
  nslookup web-svc
  wget -qO- web-svc

# 39/49. NodePort + URL
kubectl expose deployment web --name=np --type=NodePort --port=80
minikube service np --url
echo "$(minikube ip):<nodePort>"

# 40. LoadBalancer (na minikube <pending> -> tunnel)
kubectl expose deployment web --name=lb --type=LoadBalancer --port=80
minikube tunnel        # u drugom terminalu

# 42. Endpoints (puni se iz selektora)
kubectl get endpoints web-svc

# 43. cross-namespace FQDN
#  iz drugog ns:  curl web-svc.default.svc.cluster.local   (kratko ime ne radi)

# 44. port-forward na Service vs Pod
kubectl port-forward svc/web-svc 8080:80
kubectl port-forward pod/<ime> 8080:80

# 46. test iz debug poda
kubectl run tmp --rm -it --image=busybox -- sh -c "wget -qO- web-svc; nc -zv web-svc 80"

# 47. dva Deploymenta + jedan Service (zajednička labela -> load balance preko oba)
kubectl label pods -l app=a tier=shared
#  Service selector: tier: shared

# 48. dijagnoza "pod ne doseže Service": redom DNS->endpoints->selektor->readiness->port
kubectl get endpoints web-svc
kubectl describe svc web-svc
```
```
# 45. port vs targetPort vs nodePort
port       = port Servicea unutar clustera
targetPort = port na podu/kontejneru
nodePort   = vanjski port na čvoru (samo NodePort tip)
tok: host:nodePort -> Service:port -> Pod:targetPort
```

## Probe (50–53)
```yaml
# 50. HTTP liveness | 51. tcpSocket readiness | 52. startupProbe budžet
      livenessProbe:
        httpGet: {path: /, port: 80}
      readinessProbe:
        tcpSocket: {port: 6379}
      startupProbe:
        httpGet: {path: /, port: 80}
        failureThreshold: 24      # 24 × 5s = 120s za boot ~2 min
        periodSeconds: 5
```
```bash
kubectl describe pod <p>     # provjeri Liveness/Readiness/Startup
# 53. bez proba: k8s gleda samo je li proces živ — ne provjerava odgovara li app
```

---

# LO5 — Troubleshooting (43 pitanja)

## podman run greške (1–8)
```
1. -p 80:8080 nginx        -> obrnuto; ISPRAVAK: -p 8080:80 (HOST:KONTEJNER)
2. -e MYSQL_ROOT_PASSWORD  -> nema vrijednosti; init padne; daj =vrijednost
3. --network host + -p     -> -p se ignorira; makni jedno
4. busybox -> Exited(0)    -> nema dugotrajnog procesa; dodaj: sleep infinity
5. --rm -d ... echo        -> kontejner se obriše, nema logova; makni --rm
6. --memory 8m za mysql    -> premalo RAM-a, nije healthy; digni (512m+)
7. dva c. na default mreži, ping po imenu pada -> napravi user-defined mrežu
8. -v ./html:/path bez :Z  -> SELinux blokira; dodaj :Z
```

## Containerfile greške (9–18)
```
10. apt-get install bez update  -> RUN apt-get update && apt-get install -y X
11. CMD python app.py (shell)    -> exec forma: CMD ["python","app.py"] (hvata signale)
12. COPY . prije npm install     -> prvo COPY package.json, RUN npm install, pa COPY . (caching)
13. ENV PATH=/app/bin            -> ENV PATH=/app/bin:$PATH (ne briši stari PATH)
14/15. EXPOSE + krivi port       -> EXPOSE je samo doc; uskladi port s -p/CMD portom
16. velika slika                 -> ... && rm -rf /var/lib/apt/lists/* (isti RUN layer)
17. USER prije COPY/pip          -> kopiraj/instaliraj kao root, USER na kraj (ili COPY --chown)
18. golang ~1GB                  -> multi-stage build (build u prvoj fazi, kopiraj binary u alpine)
```

## Kubernetes dijagnoza (19–43)
```bash
# Glavni alati
kubectl describe pod <p>                 # Events na dnu = uzrok
kubectl logs <p> --previous              # 22. zadnji crash (CrashLoopBackOff)
kubectl get events --sort-by=.lastTimestamp   # 30. vremenski slijed
kubectl get pod <p> -o yaml              # 23. OOMKilled, limiti
kubectl get endpoints <svc>              # 25/41. Service prazan = selektor ne hvata podove
kubectl apply --dry-run=server --validate=true -f f.yaml  # 20/40. uhvati typo/indentaciju
kubectl exec -it <p> -- sh               # 31. DNS: nslookup, cat /etc/resolv.conf
kubectl debug -it <p> --image=busybox --target=<c>   # 32. distroless/no-shell
kubectl run tmp --rm -it --image=busybox -- sh       # 33. throwaway test povezanosti
kubectl get pod <p> -o jsonpath='{.status.containerStatuses[0].restartCount}'  # 42. broj restarta
kubectl describe node <node>             # 34. NotReady (kubelet, disk, CNI)
kubectl delete pod <p> --grace-period=0 --force   # 35. stuck Terminating (rizik!)
```
```
19. Pending          -> describe: nedovoljno resursa / nodeSelector / PVC ne postoji
21. ImagePullBackOff -> describe: krivi tag/ime ili privatni registry bez secreta
24. never Ready      -> readiness probe pada; popravi probu/app
26. selector != template labels -> apply vrati grešku; uskladi labele
27. requests > kapacitet čvora  -> Pending (Insufficient cpu/memory); smanji requests
28. PVC ne postoji   -> FailedMount; stvori PVC
29. ubuntu bez command -> Completed/CrashLoop; dodaj command: ["sleep","infinity"]
36. apiVersion+kind krivo (apps/v1 + Pod) -> Pod je v1; ispravi grupu/verziju
37. CreateContainerConfigError -> fali ključ u configMapKeyRef/secretKeyRef; dodaj ga
38. Secret iz drugog namespacea -> Secreti su namespace-scoped; kopiraj u isti ns
39. ProgressDeadlineExceeded -> rollout dugo ne uspijeva (npr. ImagePullBackOff); describe deployment
43. OpenShift Route vs Ingress -> Route ugrađen (out-of-box router); Ingress treba ingress controller
```

---

# LO6 — Teorija (23 pitanja)
> Potpuno pokriveno u glavnoj skripti (točke 1–23 + Swarm/k3s + savjet za odgovor). Nema novih naredbi.
> Formula odgovora: (1) jasan stav → (2) 2–3 argumenta ZA → (3) 1 protuargument.

---

# CHECKLIST PRI PREDAJI
1. Naredba/YAML u datoteci, označeno `Lo<N>_<broj>`.
2. **Screenshot dokaza** (`ps`/`get`/`describe`/UI u browseru).
3. `_D` zadatak: dopiši **objašnjenje** (1–2 rečenice).
4. Točni detalji iz teksta (slika, port, ime, env var, tvoje ime/prezime).
5. DockerHub limit → `podman login docker.io` / docker-registry Secret PRIJE rješavanja.
6. k8s problem → **uvijek prvo** `minikube status`.
