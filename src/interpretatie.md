# Interpretatie van de opdrachten

## CI/CD & Advanced GitOps
Wanneer we het hebben over CI/CD zijn er dus duidelijk twee delen die we moeten bespreken CI en CD, ik zal even uitleggen hoe deze in zijn werking gaan met de gekozen omgeving waar wij gebruik van maken.

### CI

Bij continous integration praten wij voornamelijk over het bouwen en testen van onze applicatie. Hier wordt in deze opdracht Github Actions voor gebruikt. Wij beginnen van zelf aangemaakte workflows (in de .github folder in onze repo) met daarin verschillende Github Actions die wij op onze applicatie (een website) gaan uitvoeren.

Wij starten van Markdown files die wij via een Github action in onze workflow gaan ombouwen in een statische website. Deze action noemt MdBook en converteert simpele Markdown files in een statische website. Normaal gezien kan deze dan gepublished worden op een aparte branch genaamd gh-pages. Wij hebben dit echter wat anders gedaan.

Wij starten dus ook van Markdown files, maar gaan hier dan eerst wat syntax testen op uitvoeren (testing pipeline), dan dus de MdBook action uitvoeren die deze files in een website omzetten (building pipeline) en de resulterende files, samen met een Dockerfile gebruiken om een Docker Image te maken van deze bestanden. In de Dockerfile staat dat wij Nginx gaan gebruiken als webserver en dat de files naar de correcte folder in Nginx moeten gekopieerd worden. Deze image wordt dan opgeladen naar mijn Docker Hub (We hebben gewoon eender wie zijn Docker Hub gekozen).

Deze image kan dan gebruikt worden om te deployen op onze cluster, meer specifiek in ArgoCD.

### CD

Nu dat wij een applicatie hebben om te deployen willen we dat eventuele veranderingen meteen doorgevoerd worden zodat wij de deployment niet steeds moeten naar beneden halen om dan terug te moeten opzetten.

Hier komt ArgoCD dan handig te pas, via een manifest (yaml file) van het type Application kan mijn een applicatie dus deployen in de ArgoCD omgeving. In deze manifest gaat men het path naar de repository intstellen waar al onze andere manifest files zich bevinden. Wat ArgoCD dan kan doen is deze repository monitoren en indien er veranderingen gebeuren aan deze manifest files gaat ArgoCD deze vanzelf syncen (indien auto-sync is ingesteld in de application yaml). 

Dit zorgt er nog niet voor dat onze container images worden geüpdate echter en hoewel dus dat ArgoCD updates aan onze deployment zal uitvoeren aan de hand van de manifests in onze repository gaat verandering aan de website dus niet zichtbaar zijn. Hier kunnen wij echter een module van ArgoCD voor gebruiken genaamd Image Updater. Via enkele annotaties in onze application manifest gaan wij de locatie van onze Image Registry (Docker hub) doorgeven aan deze Image Updater. Deze gaat dan de images in onze manifests vergelijken met de images op deze registry en indien Image Updater een nieuwere versie van de image ziet op Docker Hub zal hij deze "pullen" en zal het nieuwe containers opstarten met de nieuwere versie van onze image.

In ons geval hebben 3 aparte workflows (1 per branch) en afhankelijk van op welke branch gepushed wordt zal er dus een image gemaakt worden met een aangepaste naam. De docker hub heeft dus 3 images namelijk sdo:latest, sdo:dev en sdo:qa.

Image updater werkt echter enkel met Kustomize of Helm, wij hebben gebruikt gemaakt van Kustomize op de dev en qa branch en hier worden de images wel degelijk geüpdate via Image Updater. Op main gebruiken wij Helm en alhoewel er dus hashes gemaakt worden van de nieuwe image worden de containers niet geüpdate.

## Advanced GitOps

Voor advanced GitOps dacht dat wij spraken over het Argo Rollouts gedeelte van de opdracht. Nog een andere module van ArgoCD waarbij een "rollout" kan doen van de applicatie. Dit kan met een blue/green deployment of een canary deployment. 
- Blue/green inhoudend dat wanneer er een nieuwe versie van de deployment is die zal actief gezet worden terwijl de oude naar beneden wordt gehaald.
- Canary inhoudend dat de deployment geleidelijk aan gebeurd waarbij een nieuwe deployment slechts een deel van de trafiek naar de applicatie krijgt (in ons geval dus een website) bijvoorbeeld 20%. Indien het dit aankan kan dan de deployment gepromoveerd worden naar bijvoorbeeld 40% en zo tot wij aan 100% geraken en dan de nieuwe versie van de applicatie de oude volledig kan overnemen.

## Renovatebot

Renovatebot is een automatische dependency update tool, deze kan images (in ons geval docker images) in onze manifest nakijken en checken of deze nog up-to-date zijn. Indien niet kan Renovatebot pull requests maken met de nodige updates aan deze dependancies. Bij ons gaat dit bijvoorbeeld over Nginx waarop de docker image van onze website gebouwd is. Alsook de versies van de Actions in onze workflows.


## Autoscaling with Keda

Keda is een autoscaler die pods kan scalen (meer pods aanmaken of pods verminderen van de applicatie) op basis van Kubernetes events. Keda kan bijvoorbeeld op basis van een cron schedule het aantal pods omhoog halen tijdens drukke periodes en naar beneden halen tijdens de kalmere uren.

Maar het kan ook pods scalen op basis van metrics die hij uit de Kubernetes omgeving haalt. Een cpu scaler kan bijvoorbeeld aan de hand van metrics die Keda krijgt van de Kubernetes metrics server (die standaard bij de installatie van een Kubernetes omgeving zit) zien of er veel cpu wordt gebruikt in een pod en indien ja meer pods opstarten om de lading te verdelen.

Nog een mogelijkheid is om een monitoring tool als Prometheus te installeren die metrics kan scrapen van de verschillende pods via bijvoorbeeld in ons geval een Nginx exporter. Dit kan gaan over http requests bijvoorbeeld, veel trafiek naar de pods. Keda kan dan aan de hand van deze gescrapte metrics meer pods opstarten om de toename in trafiek te verdelen.

## Gateway API Spec

Binnenkomde trafiek wordt in een standaard Kubernetes omgeving behandeld door een Ingress resource. 
Ingress kan enkel omgaan met http en https trafiek. Met de Kubernetes Gateway API gaat men ook TCP en UDP trafiek kunnen routeren, men kan ook aan load-balancing doen. De Gateway API komt ook met security features die men niet krijgt met de standaard Ingress.