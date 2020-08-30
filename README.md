# Der erste Container

##### Aufgabe 1
- Anmelden bei Azure `az login`
- Einstellen der Subscription `az account set --subscription  3fb32f2b-69f5-4766-936a-47170ba0537c`
- Abholen der Credentials für K8s `az aks get-credentials --name managedk8s -g managedk8s`

- Im Order `aufgabe_1` finden sich alle Dateien dazu
- Manifest `pod.yml` anpassen (`<username>` ersetzen)
- Manifest ausrollen / installieren: `kubectl apply -f pod.yml`
- Liste aller laufenden Pods ansehen: `kubectl get pods`
- Logs der Pods anschauen: `kubetl logs helloworld-<username> -f`
- Manifest entfernen: `kubectl delete -f pod.yml`

# Das erste Deployment

Jeden Container / Pod einzeln zu starten ist unpraktisch und wird normalerweise nicht gemacht. Dafür gibt es sog. `deployments`,
eine Sammlung von Pods und etwas Verwaltung außenrum.

##### Aufgabe 2
- In den Ordner `aufgabe_2` wechseln
- Manifest `deployment.yml` anpassen (`<username>` ersetzen)
- Manifest ausrollen: `kubectl apply -f deployment.yml`
- Liste aller laufenden Pods ansehen: `kubectl get pods`
- Wir simulieren eine abstürzende Anwendung: `kubectl delete pod app-<username>-<id>-<id> --wait=false`  
- Nochmal ein Blick in die Pod-Liste: Der Container wird jetzt automatisch neugestartet.
- In der Zeit des Neustarts wäre die Anwendung aber nicht verfügbar, weil nur eine Instanz läuft. Das ändern wir jetzt.
- In der `deployment.yml` die `replicas` auf 2 setzen und neu ausrollen. (Ein `delete` ist nicht notwendig.)
- Ergebnis prüfen
- Manifest wieder entfernen

# Erste Schritte ins Netzwerk

##### Aufgabe 3
- In den Ordner `aufgabe_3` wechseln
- `*.yml` anpassen
- `deployment.yml` ausrollen
- `console.yml` ausrollen
- `k9s` starten
- IP-Adresse des Webservers raussuchen und notieren (In k9s: `:pods<Enter>`)
- Eine Shell im `console`-Pod starten  (In k9s: Pod markieren und `s<Enter>`)
- Mit `curl` die Webseite abrufen  
    `curl http://<ip>:8080`
    
# Ausfallsicherheit

Wir haben die Anwendung jetzt gestartet und können sie per HTTP erreichen (zumindest vom Cluster aus), aber 
ausfallsicher sind sie immer noch nicht.

##### Aufgabe 3.1
- Replicas des Deployments auf 2 erhöhen
- Jetzt haben wir blöderweise für jeden Pod eine IP-Adresse, 
die wir den Nutzern unseres Webservers mitteilen müssten... Sehr unpraktisch
- Dafür gibt es einen `service`, eine virtuelle IP vor mehreren Pods
- Selbst einrichten:  
 Anleitung https://kubernetes.io/docs/concepts/services-networking/service/#defining-a-service
  - Service bitte sinnvoll benennen (`<username>`)
  - Target-Port ist `8080`
- Ziel: Service soll unter `app-<username>` aus dem Console-Pod erreichbar sein. (`curl http://app-<username>:8080`)  
  DNS-Eintrag und -auflösung funktioniert automatisch. 
  
# Externe Verfügbarkeit

Bisher können wir den Service nur intern erreichen.
Jetzt soll er auch extern per Internet erreichbar sein.

Der von Azure direkt unterstützte Weg ist der `type: LoadBalancer`. Das bedeutet, dass der 
K8s-interne Service eine extern erreichbare IP-Adresse bekommt.

##### Aufgabe 3.3
- Den K8s-Service extern verfügbar machen.   
    Referenz für die Service-Definition: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#servicespec-v1-core

- Wenn der Service eine externe IP wird sie in K9s unter `:services` als `External IP` angezeigt.   
    Die Verfügbarkeit kann aber kurz dauern (< 1m).
- Alles wieder entfernen: `deployment`, `service`, `console`




# Komfort und Sicherheit

#### DNS-Name und Ingress

Jetzt haben wir einen ge-loadbalancten Service hinter einer Public-IP. 
Das ist jetzt natürlich für den Endnutzer blöd und auch TLS-Zertifikate bekommen wir dafür nicht.

##### Aufgabe 4.1:
- In den Ordner `aufgabe_4` wechseln 
- Zuerst bitte wieder `*.yml` anpassen
- Die Manifests ausrollen

- Den Service umstellen: Von `type: LoadBalancer` auf `type: ClusterIP`.  
  Ein `kubectl apply` geht hier im Normalfall schief (Zusatzfrage: Warum eigentlich?),
  nötig ist vorab ein `kubectl delete` für den Service.
  
- Wir brauchen jetzt zusätzlich zum Service eine Ingress-Resource.  
  Darin steht für K8s die Beschreibungen, wie und ob der Service exponiert werden soll.  
  Ein Beispiel dazu findet sich hier: https://kubernetes.io/docs/concepts/services-networking/ingress/#name-based-virtual-hosting  
- Hostname für die Ingress-Ressource: `<username>.mank8s.jambit.space`
- Zum Test im Browser ausprobieren.


#### TLS

Der Service ist jetzt über DNS-Namen erreichbar, aber leider nur per HTTP. Das ist heute nicht mehr ganz zeitgemäß. 
Also brauchen wir TLS-Zertifikate. Die erstellen wir natürlich nicht per Hand, sonder automatisch.

Im K8s ist ein `cert-manager` installiert, der automatisch Zertifikate von Let's Encrypt (LE) besorgen kann.  
Es sind zwei `issuer` installiert, vergleichbar mit dem Staging von LE: `staging` und `production`.

Bitte vorerst nur `staging` verwenden und erst wenn das reibungslos funktioniert auf `production` wechseln. Hintergrund:
prod lässt nur wenige Zertifikate am Tag zu und sperrt dann. 

##### Aufgabe 4.2
- Installierte Issuer: `letsencrypt` (Staging) und `letsencrypt-prod` (Production)
- Die Ingress-Ressource erweitern, ein Beispiel dazu ist hier: https://cert-manager.io/docs/usage/ingress/
- Verwendet als `secretName` bitte `<username>-cert`
- Zwischen `apply` und gültigem Zertifikat können bis zu 30 Sekunden vergehen. Bis dahin bekommt ihr ein ungültiges 
Dummy-Zertifikat vom Ingress
- Wenn alles passt, auf Production umstellen.
- Der Service sollte jetzt mit gültigem Zertifikat erreichbar sein.



# Updates und Probes

#### Updates

Die Entwickler haben sich fleißig hingesetzt und eine neue Version der Anwendung entwickelt. Jetzt wollen wir diese
Version unterbrechungsfrei ausrollen. `kubectl destroy` fällt also deswegen aus. 

K8s bietet `rolling updates`, wir können also alle Pods einer Anwendung durch neue Versionen ersetzen. Dabei wird ein
neuer Pod gestartet, in den Service aufgenommen und ein alter Pod entfernt. Das ganze so lange, bis nur noch die neue
Version läuft.

##### Aufgabe 4.3: 
- Die Anwendung aus der letzten Aufgabe sollte noch laufen (Deployment, Service, Ingress)
- Das Kommando lautet: `kubectl set image deployments/app-<username> helloworld=mbeham/hello_world:3.0.0`
- Ihr könnt dem Ganzen im Browser zuschauen. (Häufig aktualisieren) 

- Jetzt gab es nochmal ein Update: `hello_world:3.0.0-slow`. Bitte genauso ausrollen.
- In die Anwendung wurde ein fancy Framework mit viel Magie auf Basis der JVM eingebaut. Leider 
braucht die Anwendung jetzt deutlich länger zum Starten.

- Wieder im Browser zuschauen. Warum genau geht das jetzt schief?

#### Probes

Die Lösung in K8s sind Probes. Damit kann K8s feststellen, ob ein Pod bereit ist.
Es gibt zwei verschiedene Probes, wir verwenden jetzt erstmal nur `readiness`-Probes. 

##### Aufgabe 4.4:
- Eine Readiness-Probe zum Deployment hinzufügen. Die Anwendung braucht ca. 1 Minute zum Starten. 
- Hinweis 1: Die `initalDelaySeconds` sollte nicht gesetzt werden müssen.
- Hinweis 2: Da es ein HTTP-Service ist, sollte auch eine HTTP-Probe verwendet werden.
- Test: Beim Wechsel zwischen `hello_world:3.0.0` und `hello_world:3.0.0-slow` sollten keine 500-Fehler auftreten.


# ConfigMaps und Secrets

Die Anwendung braucht jetzt noch zusätzliche Konfiguration oder auch Secrets (z.B. ein Datenbankpasswort).
Dafür gibt es in K8s die Ressourcen `secret` und `configMap`. Beide funktionieren ähnlich.

Die Doku dazu gibt es hier: https://kubernetes.io/docs/concepts/configuration/secret/
##### Aufgabe 4.5: 
- Fügt ein neues Secret ein. Name: `<username>-secret`. Der Inhalt kann frei gewählt werden.
- Mounte das Secret als Umgebungsvariablen in die Pods. 
- Das Image `mbeham/hello:env` zeigt alle gesetzten Umgebungsvariablen an.

##### Aufgabe 4.6:
- Mounte das Secret als Datei in die Pods.
- Das Image `mbeham/hello:file` zeigt den Inhalt der Datei `/secret` an.
- Alles wieder aufräumen. 


# Helm

Alle bisherigen Deployments wurden direkt in YAML-Dateien geschrieben und ausgeführt. Das klappt solange gut, bis es
komplexere Deployments mit Abhängigkeiten werden, oder ein Deployment in unterschiedlichen Versionen parallel passieren soll.

Und genau dafür wurde Helm entwickelt. 

##### Aufgabe 5.1:
- Wechsel in den Ordner `aufgabe_5`
- Das ist ein komplettes Helm-Chart.
- Installieren des Helm-Charts mit `helm install <username> .`
- Aktualisieren des installierten Charts mit der Version `3.0.0` des Images mbeham/hello_world  
    Doku ist hier: https://helm.sh/docs/helm/helm_install/  
    Hinweis: Eine Änderung ist nicht nötig. 
- Helm-Chart wieder entfernen

