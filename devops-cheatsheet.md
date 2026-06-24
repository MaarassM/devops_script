# Intro to DevOps — Cheat-Sheet (naredbe + što se mijenja)

> Brzi podsjetnik na naredbe. Za *koncepte i zašto* gledaj glavnu skriptu.

## Kako čitati ovaj cheat-sheet
- Dijelovi u `<UGLATIM_ZAGRADAMA>` su ono što **ti mijenjaš** prema zadatku (zagrade NE pišeš).
- Sve ostalo (flagovi, ključne riječi) piše se **točno tako**.
- Ispod naredbe je kratko objašnjenje + popis promjenjivih dijelova.
- `podman` = `docker`, `Containerfile` = `Dockerfile` (zamjenjivo).

---

# LO3 — PODMAN: KONTEJNERI

## Pokretanje kontejnera

```
podman run -d --name <IME> -p <HOST_PORT>:<KONT_PORT> <SLIKA>
```
Pokreni kontejner u pozadini i objavi port.
- `<IME>` — ime kontejnera (proizvoljno, npr. `web`)
- `<HOST_PORT>` — port na tvom računalu (npr. `8081`)
- `<KONT_PORT>` — port na kojem app sluša UNUTRA (npr. `80` za nginx)
- `<SLIKA>` — npr. `docker.io/library/httpd`
- **Sitnica:** redoslijed je uvijek `HOST:KONTEJNER`. Zamjena = klasična greška.
- `-d` = detached (pozadina). Bez njega zauzme terminal.

Korisni dodatni flagovi (slažeš ih u istu `run` naredbu):
- `--hostname <HOSTNAME>` — postavi hostname unutar kontejnera
- `--restart=always` — podman ga ponovo digne ako padne/ugasi se
- `--memory <N>m` `--cpus <N>` — limiti (npr. `--memory 256m --cpus 0.5`)
- `--env <KLJUC>=<VRIJEDNOST>` — jedna env varijabla
- `--env-file <DATOTEKA>` — env varijable iz datoteke
- `--network <MREZA>` — spoji na mrežu | `--network none` (bez mreže) | `--network host` (dijeli host mrežu, tada `-p` nema učinka)
- `--user <UID_ILI_IME>` — pokreni kao ne-root korisnik
- `--rm` — obriši kontejner čim završi (oprez: gubiš logove)
- `-v <VOLUMEN>:<PUTANJA>` — montiraj named volume
- `-v <HOST_DIR>:<PUTANJA>:ro` — bind mount samo za čitanje (`:ro`)
- `-v <HOST_DIR>:<PUTANJA>:Z` — `:Z` rješava SELinux blokadu (Red Hat!)
- `--label <KLJUC>=<VRIJEDNOST>` — dodaj labelu
- `--secret <IME_SECRETA>` — ubaci secret kao datoteku u `/run/secrets/<IME_SECRETA>`
- `-p <HOST_IP>:<HOST_PORT>:<KONT_PORT>` — veži port samo na jednu IP (npr. `127.0.0.1:8082:80`)

## Pregled i dokazivanje (za screenshotove)

```
podman ps
podman ps -a
podman ps --filter "label=<KLJUC>=<VRIJEDNOST>"
```
Popis kontejnera. `-a` = i zaustavljeni. `--filter` = samo oni s labelom.

```
podman inspect <IME>
podman inspect -f '{{<GO_TEMPLATE>}}' <IME>
```
Sve o kontejneru / izvuci jedno polje.
- `<GO_TEMPLATE>` primjeri: `.NetworkSettings.IPAddress`, `.RestartCount`, `.Config.Env`

```
podman logs --tail <N> --since <VRIJEME> -f <IME>
```
Logovi. `--tail <N>` zadnjih N redaka | `--since <VRIJEME>` (npr. `10m`) | `-f` prati uživo.

```
podman exec -it <IME> <NAREDBA>
```
Pokreni naredbu unutar kontejnera. `-it` = interaktivno (npr. `sh`, `bash`).
- Dokazi identitet: `podman exec <IME> id` ili `whoami`
- Dokazi env: `podman exec <IME> env`

```
podman stats --no-stream <IME>
podman top <IME>
podman port <IME>
podman diff <IME>
```
Resursi (CPU/RAM) | procesi unutra | objavljeni portovi | promjene u filesystemu (A=dodano, C=promijenjeno, D=obrisano).

```
podman cp <IZVOR> <ODREDISTE>
```
Kopiraj datoteku host↔kontejner. Primjer u kontejner: `./file.txt <IME>:/putanja`; iz kontejnera: `<IME>:/putanja ./file.txt`.

```
podman commit <IME> <NOVA_SLIKA>
podman rename <STARO_IME> <NOVO_IME>
podman pause <IME>
podman unpause <IME>
podman events
```
Spremi izmijenjeni kontejner u sliku | preimenuj | zamrzni procese | odmrzni | gledaj događaje uživo.

## Mreže

```
podman network create <MREZA>
podman network inspect <MREZA>
podman network ls
podman network rm <MREZA>
podman network prune -f
```
- `<MREZA>` — ime mreže (proizvoljno). Na vlastitoj mreži kontejneri se nalaze **po imenu** (DNS). Na defaultnoj NE.

## Volumeni

```
podman volume create <VOLUMEN>
podman volume ls
podman volume inspect <VOLUMEN>
podman volume rm <VOLUMEN>
podman volume prune -f
```
- `<VOLUMEN>` — ime volumena. Koristiš ga s `-v <VOLUMEN>:<PUTANJA>` u `run`.

## Secrets

```
printf '<TAJNA>' | podman secret create <IME_SECRETA> -
podman secret ls
podman secret rm <IME_SECRETA>
```
- `<TAJNA>` — vrijednost (npr. lozinka). `-` na kraju = čitaj sa stdina.
- `<IME_SECRETA>` — ime secreta. U `run` dodaš `--secret <IME_SECRETA>` → datoteka `/run/secrets/<IME_SECRETA>`.

## Podovi

```
podman pod create --name <POD> -p <HOST_PORT>:<KONT_PORT>
podman run -d --pod <POD> --name <IME> <SLIKA>
podman pod inspect <POD>
podman pod ps
podman pod rm -f <POD>
podman pod rm -f -a
```
- Kontejneri u podu dijele mrežu → vide se preko `localhost`. Port objavljuješ na razini PODA.
- `rm -f -a` = silom obriši SVE podove (čišćenje zaglavljenih stackova).

Most prema Kubernetesu:
```
podman generate kube <POD_ILI_KONTEJNER> > <DATOTEKA>.yaml
podman kube play <DATOTEKA>.yaml
podman kube play --down <DATOTEKA>.yaml
```
Generiraj k8s YAML iz poda | pokreni iz YAML-a | sruši.

## Compose

```
podman compose up -d
podman compose ps
podman compose logs
podman compose up --scale <SERVIS>=<N>
podman compose exec <SERVIS> <NAREDBA>
podman compose down
podman compose down --volumes
```
- `<SERVIS>` — ime servisa iz `compose.yaml`. `<N>` — broj replika.
- `down` čuva volumene; `down --volumes` ih briše.

### Struktura compose.yaml (mijenjaš označene dijelove)
```yaml
services:
  <SERVIS_APP>:
    image: <SLIKA>
    ports:
      - "<HOST_PORT>:<KONT_PORT>"
    env_file: <ENV_DATOTEKA>          # npr. .env
    depends_on:
      <SERVIS_DB>:
        condition: service_healthy
    networks:
      - <MREZA>
  <SERVIS_DB>:
    image: <SLIKA_DB>
    environment:
      <KLJUC>: <VRIJEDNOST>
    volumes:
      - <VOLUMEN>:<PUTANJA_U_KONTEJNERU>
    networks:
      - <MREZA>
    healthcheck:
      test: ["CMD-SHELL", "<NAREDBA_PROVJERE>"]   # npr. pg_isready -U postgres
      interval: 5s
      retries: 5
networks:
  <MREZA>:
    internal: true                    # baza nedostupna s hosta
volumes:
  <VOLUMEN>:
```

---

# CONTAINERFILE + GRADNJA SLIKA (za LO5 troubleshooting i build)

## Gradnja i registri

```
podman build -t <SLIKA>:<TAG> -f <CONTAINERFILE> <KONTEKST>
```
- `<SLIKA>:<TAG>` — npr. `myapp:1.0`. `-f <CONTAINERFILE>` — ako se ne zove `Containerfile`. `<KONTEKST>` — mapa (obično `.`).
- Dodaci: `--build-arg <KLJUC>=<VRIJEDNOST>` (vrijednost za ARG) | `--no-cache` (ignoriraj cache slojeva)

```
podman login <REGISTRY>
podman tag <SLIKA> <REGISTRY>/<NAMESPACE>/<IME>:<TAG>
podman push <REGISTRY>/<NAMESPACE>/<IME>:<TAG>
```
- `<REGISTRY>` — npr. `docker.io`, `quay.io`. **Rješava `toomanyrequests`** (login = veći pull limit).
- Kredencijali se spremaju u `${XDG_RUNTIME_DIR}/containers/auth.json`.

```
podman pull <SLIKA>@sha256:<DIGEST>
podman save -o <DATOTEKA>.tar <SLIKA>
podman load -i <DATOTEKA>.tar
podman history <SLIKA>
podman image inspect <SLIKA>
podman images
podman rmi <SLIKA>
podman image prune
```
Pull po digestu (reproducibilno) | spremi u tar (air-gap) | učitaj iz tara | slojevi | detalji | popis | obriši | makni "dangling".

## Containerfile direktive (mijenjaš vrijednosti)
```dockerfile
FROM <SLIKA>:<TAG>                  # bazna slika; tag može biti ARG
ARG <KLJUC>=<DEFAULT>               # varijabla SAMO pri buildu (--build-arg)
ENV <KLJUC>=<VRIJEDNOST>            # varijabla i pri buildu i u radu; PAZI: ENV PATH=... bez :$PATH razbije alate
WORKDIR <PUTANJA>                   # radni direktorij
COPY <IZVOR> <ODREDISTE>            # kopiraj iz konteksta (preporučeno)
COPY --chown=<KORISNIK> <IZVOR> <ODR>  # kopiraj i postavi vlasnika
ADD <IZVOR> <ODREDISTE>            # kao COPY + raspakira tar / skida URL
RUN <NAREDBA>                       # izvrši pri buildu (svaki RUN = novi sloj)
USER <KORISNIK>                     # od ove točke ne-root (COPY/RUN ovisni o pravima!)
EXPOSE <PORT>                       # SAMO dokumentacija, NE objavljuje port
HEALTHCHECK CMD <NAREDBA>           # provjera zdravlja
LABEL <KLJUC>=<VRIJEDNOST>
CMD ["<PROGRAM>", "<ARG>"]          # exec forma (PREPORUKA: hvata signale)
ENTRYPOINT ["<PROGRAM>"]            # fiksni dio naredbe; CMD daje zadane argumente
```
**Sitnice (česta pitanja LO5):**
- `apt-get install` bez `apt-get update` prije → fail. Spoji: `RUN apt-get update && apt-get install -y <PAKET>`.
- Čišćenje cachea mora biti u **istom** RUN-u: `... && rm -rf /var/lib/apt/lists/*`.
- Layer caching: prvo kopiraj `package.json`/`requirements.txt` pa instaliraj, tek onda `COPY . .`.
- `CMD python app.py` (shell forma) ne hvata signale → koristi exec formu `["python","app.py"]`.
- Multi-stage za male slike: build u prvoj `FROM`, kopiraj samo rezultat u drugu minimalnu `FROM`.

---

# LO4 — KUBERNETES (kubectl + minikube)

## minikube
```
minikube start --driver=<DRIVER>
minikube status
minikube ip
minikube service <SVC> --url
minikube tunnel
minikube kubectl -- <KUBECTL_ARGUMENTI>
```
- `<DRIVER>` — `podman` / `docker`. `service --url` daje URL za NodePort. `tunnel` daje EXTERNAL-IP LoadBalanceru.

## Deploymenti
```
kubectl get nodes
kubectl create deployment <IME> --image=<SLIKA> --replicas=<N>
kubectl apply -f <DATOTEKA>.yaml
kubectl get deployment,rs,pods
kubectl get <RESURS> <IME> -o wide
kubectl get <RESURS> <IME> -o yaml
kubectl describe <RESURS> <IME>
kubectl delete <RESURS> <IME>
kubectl edit <RESURS> <IME>
```
- `<RESURS>` — `deployment`, `pod`, `service`/`svc`, `rs`, `pvc`, `configmap`, `secret`...
- `-o wide` pokaže IMAGES; `-o yaml` puni spec; `describe` ima Events (uzroci).

```
kubectl scale deployment <IME> --replicas=<N>
kubectl set image deployment/<IME> <KONTEJNER>=<SLIKA>:<TAG>
kubectl rollout status deployment/<IME>
kubectl rollout history deployment/<IME>
kubectl rollout undo deployment/<IME>
kubectl annotate deployment/<IME> kubernetes.io/change-cause="<RAZLOG>"
```
Skaliraj | promijeni sliku (rolling update) | prati | povijest | rollback | postavi CHANGE-CAUSE.
- `<KONTEJNER>` — ime kontejnera iz template-a (npr. `nginx`).

```
kubectl get pods --selector <KLJUC>=<VRIJEDNOST>
kubectl label <RESURS> <IME> <KLJUC>=<VRIJEDNOST>
```
Filtriraj po labeli (Deployment→RS→Pod veza ide preko selektora labela).

## Servisi
```
kubectl expose deployment <IME> --port=<PORT> --target-port=<TARGET_PORT>
kubectl expose deployment <IME> --port=<PORT> --target-port=<TARGET_PORT> --type=NodePort --name=<SVC_IME>
kubectl get svc
kubectl get endpoints <SVC>
```
- `<PORT>` — port Servicea | `<TARGET_PORT>` — port Poda | (NodePort dobiva i `nodePort` na čvoru).
- `--type=` → `ClusterIP` (default, samo unutar) | `NodePort` (izvana preko čvora) | `LoadBalancer` (na minikube `<pending>` → `minikube tunnel`).
- `endpoints` = stvarne IP Podova iza Servicea (prazno = selektor ne hvata Podove).

```
kubectl run <IME> --image=<SLIKA> -it --rm --restart=Never -- <NAREDBA>
kubectl exec -it <POD> -- <NAREDBA>
kubectl port-forward <RESURS>/<IME> <LOKALNI>:<UDALJENI>
```
Privremeni test Pod (npr. `busybox -- wget -qO- http://<SVC>`) | shell u Pod | preusmjeri port lokalno.
- DNS unutar clustera: `<SVC>` (isti namespace) ili FQDN `<SVC>.<NAMESPACE>.svc.cluster.local` (iz drugog namespacea).

## ConfigMap i Secret
```
kubectl create configmap <IME> --from-literal=<KLJUC>=<VRIJEDNOST>
kubectl create configmap <IME> --from-file=<DATOTEKA>
kubectl create secret generic <IME> --from-literal=<KLJUC>=<VRIJEDNOST>
kubectl create secret generic <IME> --from-file=<DATOTEKA>
kubectl create secret tls <IME> --cert=<CERT_FILE> --key=<KEY_FILE>
kubectl create secret docker-registry <IME> --docker-server=<SERVER> --docker-username=<USER> --docker-password=<PASS>

kubectl create secret generic db-cred \
  --from-literal=DB_USER=appuser \
  --from-literal=DB_PASSWORD=Horvat
kubectl get secret <IME> -o yaml
```
- Secret je **base64-kodiran, NE šifriran**. `docker-registry` secret + `imagePullSecrets` rješava DockerHub limit u k8s (LO4 #2).
- Korištenje: kao volumen (svaki ključ = datoteka), `subPath` (jedan ključ na putanju), `envFrom` + `secretRef`/`configMapRef` (svi ključevi kao env).

## Jobs / CronJobs / StatefulSet / DaemonSet
```
kubectl get storageclass
kubectl get pv
kubectl get pvc
kubectl create job <IME> --from=cronjob/<CRONJOB_IME>
```
- StatefulSet → stabilna imena + PVC po replici + ide uz headless Service (`clusterIP: None`).
- DaemonSet → jedan Pod po čvoru. Job → jednom do kraja. CronJob → po rasporedu.

## Probe (u manifestu, dio container spec-a)
```yaml
livenessProbe:    # živ? ako padne -> restart
  httpGet: { path: <PUTANJA>, port: <PORT> }
readinessProbe:   # spreman? ako ne -> izbačen iz Servicea
  tcpSocket: { port: <PORT> }
startupProbe:     # vrijeme za boot prije liveness provjera
  httpGet: { path: <PUTANJA>, port: <PORT> }
  failureThreshold: <N>
  periodSeconds: <SEK>
```

---

# LO5 — TROUBLESHOOTING (dijagnostika)

## Kubernetes dijagnostika
```
kubectl describe pod <POD>
kubectl logs <POD>
kubectl logs <POD> --previous
kubectl get events --sort-by=.lastTimestamp
kubectl get pod <POD> -o jsonpath='{.status.containerStatuses[0].restartCount}'
kubectl get pod <POD> -o yaml
kubectl get endpoints <SVC>
kubectl apply --dry-run=server --validate=true -f <DATOTEKA>.yaml
kubectl debug -it <POD> --image=<SLIKA> --target=<KONTEJNER>
kubectl run tmp --rm -it --image=busybox --restart=Never -- sh
```
- `describe` → čitaj **Events** na dnu (uzrok). `--previous` → log zadnjeg pada. `dry-run` → uhvati typo/indentaciju prije primjene. `debug` → za distroless/no-shell kontejnere.

## Brza tablica stanja (uzrok → fix)
- **Pending** → nema resursa / kriv nodeSelector / PVC ne postoji → `describe`, smanji requests / popravi / stvori PVC
- **ImagePullBackOff** → kriv naziv/tag ili privatni registry bez secreta → ispravi / dodaj imagePullSecret
- **CrashLoopBackOff** → app pada u krug → `logs --previous`, popravi uzrok
- **OOMKilled** → prešao memory limit → digni limit / popravi app
- **CreateContainerConfigError** → fali ključ u configMap/secretKeyRef → dodaj ključ
- **never Ready** → readiness probe pada → popravi probu/app
- **Service prazan** → selektor ≠ Pod labele → uskladi (`get endpoints`)
- **ProgressDeadlineExceeded** → rollout dugo ne uspijeva → `describe deployment`
- **Terminating zaglavljen** → finalizeri/grace period → `--grace-period=0 --force` (rizik)

## Podman tipične greške (iz LO5 pitanja)
- `-p` obrnut → `HOST:KONTEJNER` (npr. `8081:80`)
- `-e KLJUC` bez `=vrijednost` → init baze pada → daj vrijednost
- `--network host` + `-p` → `-p` se ignorira → makni jedno
- busybox/alpine odmah Exited(0) → dodaj `sleep infinity`
- `--rm -d ... echo` → logovi nestanu (auto-remove) → makni `--rm`
- `--memory 8m` za bazu → premalo, nije healthy → digni limit
- dva kontejnera na default mreži, `ping` po imenu pada → napravi vlastitu mrežu
- `-v ./dir:/path` na Red Hatu → SELinux blokira → dodaj `:Z`

---

# LO6 — TEORIJA (nema naredbi)
Usporedbe i argumentacija: Kubernetes vs Docker mreža/storage, OpenShift vs vanilla k8s, Podman vs Docker (daemonless/rootless), Docker Swarm, k3s, vendor lock-in, kad što odabrati. Detalji u glavnoj skripti.

---

# NAJČEŠĆE ZAMKE (zapamti za ispit)
1. `-p HOST:KONTEJNER` — nikad obrnuto.
2. DNS po imenu radi samo na **vlastitoj** mreži (podman) / preko **Servicea** (k8s).
3. `toomanyrequests` → `podman login docker.io` / docker-registry Secret.
4. Volumen preživi `rm`; bez volumena podaci nestaju.
5. Secret = base64, **NIJE** šifriran.
6. `EXPOSE` ne objavljuje port (samo dokumentacija); objavljuje `-p`.
7. Skaliranje = isti RS; promjena slike/template = novi RS.
8. Za svaki zadatak: znaj **dokaznu naredbu** za screenshot (`get`, `describe`, `inspect`, `ps`).
