# tak-bot-k8s

Kubernetes/Helm-Nachbau des [tak-bot](https://github.com/AntonHollube/tak-bot) docker-compose-Stacks.
Läuft als eigenständiger [k3s](https://k3s.io/)-Single-Node-Cluster parallel zum
bestehenden Docker-Compose-Betrieb auf demselben Server – ohne den produktiven
TAK-Server oder andere laufende Dienste anzufassen.

## Warum dieses Repo

`tak-bot` selbst läuft komplett über `docker-compose.yml`. Dieses Repo ist der
bewusst getrennte **Infrastruktur-Teil**: dieselben Services als
Kubernetes-Manifeste/Helm-Chart, um zu zeigen, wie der Stack cloud-native
betrieben werden würde – App-Code und Infra-Code sauber getrennt, statt alles
in einem Repo zu vermischen.

## Mapping docker-compose → Kubernetes

| docker-compose Service | Kubernetes-Objekt | Status |
|---|---|---|
| `tak-db` (Postgres + PostGIS) | `StatefulSet` + `volumeClaimTemplates` (PVC) + `Secret` + `Service` (headless) | **Live** – eigene, frische Instanz/Daten, nicht die Produktiv-DB |
| `grafana` | `Deployment` + `PVC` + `Secret` + `Service` (NodePort) | **Live** – erreichbar über eine neue Subdomain |
| `tak-worker` | `Deployment` | **Live** – ruft nur öffentliche APIs (Overpass, PegelOnline) ab und schreibt JSON-Caches; braucht keinen TAK-Server, daher risikofrei voll lauffähig |
| `tak-server` | `Deployment` + `PVC` + `Service` | **Modelliert, aber `enabled: false`** (siehe unten) |
| `tak-bot` | `Deployment` | **Modelliert, aber `enabled: false`** (siehe unten) |
| `tak-mesh-gateway` | – | **Bewusst ausgeschlossen** (siehe unten) |

## Warum `tak-server`/`tak-bot` deaktiviert sind

`tak-server` ist eine lizenzierte TAK-Server-Distribution mit eigener mTLS-PKI
(Client-Zertifikate für jeden Client). Eine zweite, komplett isolierte Instanz
dafür aufzusetzen – eigene Zertifikatskette, eigene Nutzerkonten, eigene
Datenbank – ist ein eigenständiges Vorhaben für sich und unabhängig vom
eigentlichen Ziel dieses Repos (zu zeigen, *wie* der Stack auf Kubernetes
abgebildet wird). Beide Services sind deshalb vollständig als Helm-Templates
vorhanden ( `templates/tak-server.yaml`, `templates/tak-bot.yaml` ) und über
`values.yaml` (`takServer.enabled`, `takBot.enabled`) togglebar – das
architektonische Verständnis ist da, die produktive Aktivierung ist bewusst
der nächste Schritt.

## Warum `tak-mesh-gateway` ausgeschlossen ist

Der Mesh-Gateway-Dienst braucht direkten Zugriff auf ein angeschlossenes
Meshtastic/LoRa-USB-Gerät (`/dev/ttyACM0`). Das ist an genau ein physisches
Gerät an genau einem Host gebunden – das Gegenteil von dem, wofür Kubernetes
gebaut ist (Pods werden frei über den Cluster verteilt/neu geplant). Für einen
Single-Node-Cluster ließe sich das zwar per `hostPath`/Device-Plugin
nachbilden, aber der Mehrwert steht in keinem Verhältnis zum Aufwand, solange
es nur einen Node und ein Gerät gibt. Bleibt daher bewusst als eigenständiger
Docker-Compose-/Systemd-Dienst nah an der Hardware bestehen.

## Deployment

```bash
helm upgrade --install tak-bot ./chart -n tak-demo --create-namespace
```

Voraussetzung: ein laufender k3s-Cluster mit `--disable=traefik
--disable=servicelb` (vermeidet Portkonflikte mit einer bereits vorhandenen
Reverse-Proxy-Lösung wie nginx-proxy-manager). Grafana wird per `NodePort`
exponiert und über den bestehenden Reverse-Proxy auf eine neue Subdomain
gemappt – kein zweiter Ingress-/TLS-Stack nötig.

**Image-Distribution:** `tak-worker` wird lokal aus dem `Dockerfile.bot` des
[tak-bot](https://github.com/AntonHollube/tak-bot)-Repos gebaut und für diesen
Single-Node-Cluster direkt per `docker save | k3s ctr images import` in die
containerd-Instanz von k3s geladen (`imagePullPolicy: IfNotPresent`) – ohne
Umweg über eine Registry. Das ist für einen einzelnen Node ausreichend und
vermeidet unnötige Registry-Abhängigkeiten; sobald mehrere Nodes im Spiel sind
(z. B. auf GKE), braucht jeder Node Zugriff auf dasselbe Image, dann wird ein
echter Registry-Push (`ghcr.io`) nötig – siehe Roadmap.

## Roadmap

Nächste logische Schritte, um exakt auf den Kubernetes/GCP/Terraform/Helm-Stack
einer produktiven Cloud-Umgebung zu kommen:

1. **Terraform** für reproduzierbares Cluster-Provisioning (statt manuellem `k3s`-Install)
2. Wechsel auf **GKE** (Google Kubernetes Engine) als Multi-Node-Cluster
3. **GitHub Actions**-Pipeline: Image-Build + Push nach `ghcr.io` + `helm upgrade` bei jedem Merge
4. `tak-server`/`tak-bot` produktiv aktivieren, sobald eine isolierte PKI/Zertifikatskette dafür aufgesetzt ist
