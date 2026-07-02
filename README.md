### Aufsetzen von Homeserver 
Ich möchte auf meinen aktuellen drei Mini Dells ein Kubernetes Cluster aufbauen. 
Da ich beruflich viel mit Rancher Kubernetes arbeite, bieten sich Produkte aus diesem Bereich an. 


Ich bin schmerzbefreit, ob es nun k8s oder die k3s Version von Rancher wird. 
Es soll zügig für mich möglich sein, Software auf einem cluster bereitzustellen. 

## Anforderungen
Zweck können folgende Beispiele sein:
- Data Engineering Training (Airflow, dbt, duckdb, Python, ...) 
- Verwaltung von Datenbanken
- Bereitstellung von Applikationen für mein zu Hause (Mediaserver, Jellyfin, Digitalisierung von DvDs)
- Bereitstellung von eigenen entwickelten Applikationen 

- Perspektivisch soll evaluiert werden, ob bestimmte Services auch von extern erreichbar sein sollen 

## Hardware 
- es handelt sich um aktuell drei Dell Minis 5040s (8GB Ram, Intel Core i5, 8500T @ 2,1 GHz - 8 GB DDR4 - 250 GB SSD
- installiert ist Debian 13 
- perspektivisch sollen weitere Rechne hinzukommen können 

die namen der Rechner lauten: 
- k8s-master-01 
- k8s-worker-01
- k8s-worker-02

## Software 
Folgende Software Systeme möchte ich gerne perspektivisch auf dem Cluster nutzen können: 
- Grafana, Loki, Prometheus
- was ist mit Longhorn? sind diese in Verbindung mit den drei physischen Dells sinnvoll? soll nur für die Applikationen und Replikationen sein 
    - für Medien werde ich perspektivisch einen separaten Speicher bereitstellen
    - welche Möglichkeiten hat man hier? ein NAS-System? 
- testweise zukünftig Istio oder Kyverno (diese Tools nutzen auf der Arbeit, ich weiß, dass diese komplex sein können und für bestimmte 



