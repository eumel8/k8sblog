---
layout: post
tag: de
title: "Hochleistung-GoLang-Applikationen mit OTC RDS Operator"
subtitle: "Erklärungsbedürftige IT-Produkte oder abgefahrene Technologien werden in der Funktion etwas anschaulicher, wenn sie in der Praxis erprobt werden können. Hilfreich dazu ist eine Demo App."
date: 2022-09-17
background: '/images/high-performance-golang.png'
twitter: '/images/2022-05-27-1.png'
author: eumel8
---

Demonstration zur Einsatzweise von Kubernetes incl. Container, Pod, Deployment, Service und Ingress findet man zum Beispiel <a href="https://github.com/mcsps/use-cases/blob/master/demoapp.yaml">hier</a>, mit externem Volume auch <a href="https://github.com/mcsps/use-cases/blob/master/demoapp_volume.yaml">hier</a>. Das Ergebnis kann im Browser von überall im Internet besichtigt werden.

Der <a href="https://github.com/eumel8/otc-rds-operator">OTC RDS Operator</a> kann OTC RDS Datenbanken aus einem Kubernetes Cluster deployen. Mit einer Autopilot Funktion skaliert die Instanz eigenständig. Wie kann ich aber so eine Resource in einer Applikation einbinden? Dazu hat der Operator seine eigene <a href="https://github.com/eumel8/oro-demoapp">Demo App</a>.  Die basiert auf den bisherigen Demo App Modellen mit Webapplikation und MySQL-Backend, hat aber, in Go geschrieben, noch einige Besonderheiten:

```golang
import (
	// ...
	"k8s.io/client-go/rest"
	rdsv1alpha1clientset "github.com/eumel8/otc-rds-operator/pkg/rds/v1alpha1/apis/clientset/versioned"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)
// ...
	for {
		log.Println("LOOP: get rds " + rdsname)
	
	rds, err := rdsclientset.McspsV1alpha1().Rdss(namespace).Get(context.TODO(), rdsname, metav1.GetOptions{})
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			log.Println("KRDS: " + err.Error())
			return nil, err
		}
		if rds.Status.Status == "ACTIVE" {
			log.Println("DB ACTIVE")
			for _, i := range *rds.Spec.Users {
				dbUser = i.Name
				dbPass = i.Password
				break
			}
			dbDriver := "mysql"
			dbHost := rds.Status.Ip
			dbPort := rds.Spec.Port
			dbName := rds.Spec.Databases[0]
			log.Println("DB CONNECT " + dbHost)
			db, err = sql.Open(dbDriver, dbUser+":"+dbPass+"@tcp("+dbHost+":"+dbPort+")/"+dbName)
			if err != nil {
				http.Error(w, err.Error(), http.StatusInternalServerError)
				log.Println("DBCONN: " + err.Error())
				return nil, err
			}
		}
		// ...
		time.Sleep(5 * time.Second)
	}
```

Die Demo App läuft im Cluster, hat also automatisch Zugriff auf die Kubernetes-API. In der Funktion zum Herstellen der Verbindung zur Datenbank, ruft sie diese auf und sucht in den RDS Resourcen nach der passenden Instanz im selben Namespace, wartet bis diese im Status "ACTIVE" ist, sofern sie zusammen mit der App gerade deployt wurde und greift sich dort die Credentials ab, die ueber das RDS-Objekt definiert wurden und erfährt Hostname/IP-Adresse, die die OTC für die RDS-Instanz vergeben wurde und fortan im Status des RDS-Objekts im Kubernetes-Cluster stehen. Voila, die Verbindung klappt, die App kann die Datenbank nutzen. 

Was im Labor klappt, muss nicht unbedingt im Produktionseinsatz funktionieren. Es gibt die <a href="https://github.com/ddosify/ddosify/releases">Go App Ddosify</a>, mit der Lastszenarios für Webapplikatonen nachgestellt werden können. Das Programm gibt es auch als Onlinedienst, ist aber nach einer kurzen Testphase kostenpflichtig.  Dafür läuft das Programm in vielen Standorten der Welt, von der man seinen Dienst testen kann. Wir können uns aber auch das Binary herunterladen und von unserer Wörkstation unsere Demo App testen:

```bash
$ ddosify -t https://oro-demoapp.mcsps.telekomcloud.com/insert -m POST -h "Content-Type: application/x-www-form-urlencoded" -b "name=value1&city=value2" -l incremental -n 200000 -d 1200 -T 1

# enspricht
curl -X POST https://oro-demoapp.mcsps.telekomcloud.com/insert
   -H "Content-Type: application/x-www-form-urlencoded" 
   -d "name=value1&city=value2"
```

200.000 neue Datenbankeinträge in 20 Minuten (damit nach 15 Minuten unser Autopilot des RDS Operator anspringt). Das hält unsere Labor-App nicht aus.

Erster Fehler: MySQL - too many connections.
Lässt sich auf der normalen Datenbank über Variablen ändern:

```bash
mysql> set global max_connections = 2000;
```

Klappt in der OTC aber nicht und ist in der Parametergroup einzustellen. Default sind es 800 in der MySQL-Default-Parametergroup. Ist bischen umständlich in der OTC-Web-Console einzustellen. Weder der Operator noch der openstack-Client bieten derzeit eine passende Option. Danach gleich Datenbank rebooten.

Nächster Fehler: Oomkilled POD. Die spartanischen Limits der Labor-App sind erreicht. Diese kann man im Statefulset erhöhen:

```bash
$ kubectl -n rds1 edit statefulsets.apps demoapp
...
       resources:
          limits:
            cpu: "1"
            memory: 1280Mi
          requests:
            cpu: 100m
            memory: 128Mi
```

Nächster Fehler: too many open files. Vermutlich im Container, weil das Verzeichnis zum Ablegen der Bilder immer neu geladen wird bei jedem Zugriff. Besser ist also die Anwendung in die Breite zu skalieren:

```bash
$ kubectl -n rds1 scale --replicas=5 statefulsets.apps demoapp
```

Mit angehängtem Storage klappt das bloss bei Multi-Attach-fähigem oder RWX-Storage wie NFS-Client.
Für Performance-Tests der Datenbank können wir auf den permanenten Bilderspeicher verzichten und das Verzeichnis in den RAM-Speicher des PODs legen, indem wir den PVC Eintrag ersetzen durch:

```yaml
      volumes:
      - emptyDir:
          medium: Memory
        name: demoapp-volume
```

Die Demo App läuft damit schon recht gut, die RDS wird vielleicht tatsächlich skaliert:

```yaml
Events:
  Type    Reason  Age                From              Message
  ----    ------  ----               ----              -------
  Normal  Update  34m (x52 over 9h)  otc-rds-operator  This instance set autopilot.
  Normal  Update  29m (x2 over 29m)  otc-rds-operator  This instance is scaling to rds.mysql.s1.medium.ha
```

Allerdings gibt es noch genug Stellschrauben an den Komponenten, die bei unserer Demo App beteiligt sind: Loadbalancer, Ingress, Worker Nodes. Ddosify zählt die Fehlerarten und gibt am Ende seines Werkens einen Bericht. 

```yaml
RESULT
-------------------------------------
Success Count:    466   (46%)
Failed Count:     534   (54%)

Durations (Avg):
  DNS                  :0.0018s
  Connection           :0.0404s
  TLS                  :0.0912s
  Request Write        :0.0001s
  Server Processing    :1.0341s
  Response Read        :0.4948s
  Total                :1.6625s

Status Code (Message) :Count
  200 (OK)                       :194
  500 (Internal Server Error)    :272

Error Distribution (Count:Reason):
  534     :connection timeout
```

Damit kann man dann die Installation untersuchen und die Qualität seines Dienstes verbessern.

Happy Testing
