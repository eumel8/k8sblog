---
layout: post
tag: de
title: "Go Projekt: Der Overlay Network Test"
subtitle: "Vor einem knappen Jahr begann ich einen Onlinekurs zum Erlernen der Programmiersprache Go. Seitdem habe ich mir etliche Programme im Quellcode angeguckt, den einen oder anderen Patch geschrieben, was aber nie ueber einen Einzeiler hinausgeht. Zeit, das erste eigene Programm zu schreiben."
date: 2021-12-31
background: '/images/gopher-over-sea.png'
twitter: '/images/2021-12-31-2.png'
author: eumel8
---

# Vorwort

Eines gleich vorweg: Ich bin kein Programmierer, und werde es wahrscheinlich auch nicht mehr werden. Mein ganzes Arbeitsleben mache ich Betrieb. Deswegen habe ich auch nur begrenztes Wissen zur Softwareprogrammierung und betrachte es immer aus Sicht des Betriebs, letztlich hier der Anwender einer solchen Software.  Deswegen fehlen hier im Projekt vielleicht ein paar Grundfesten der Softwareprogrammierung und Optimierung. Andererseits, was solls: Es ist mein erstes Go-Programm!

# Warum GO

<img src="/images/2021-12-31-1.png" />

Go gibt es noch gar nicht so lange als Programmiersprache, also genaugenommen genau jetzt 10 Jahre. 2011 ist es entstanden und 2012 erblickte das erste Release das Licht der Welt.
Entwickelt wurde es von Mitarbeitern von Google aus Unzufriedenheit ueber die bestehende Landschaft an Programmiersprachen. Egal ob C oder Java, man hat immer Unmengen von Abhaengigkeiten zu Bibliotheken, die man mit installieren muss, ehe das eigentliche Programm startet. Wenn alles von libc abhaengt, hat man auch noch Abhaengigkeit zu bestimmten Betriebssystem-. oder Kernelversionen. Zum Schluss hat man einen ganzen Wust von Dateien installiert, nur um dann ein Programm mit einer Funktion am Laufen zu haben.
Go hat diese alten Zoepfe alle abgeschnitten. Es gibt EIN Programm mit EINER Datei. Dennoch ist dieses Binary rank und schlank ... naja, es frisst jetzt keine Gigabytes an Speicher. Dennoch muss man nicht auf Bibliotheken, Go Packages, verzichten.
Diese werden aber zur Entwicklungszeit eingebunden und entsprechend der genutzen Funktionen fertig im Programm ausgeliefert. 
Zum Schluss hat man eine auf sehr unterschiedlichen Plattformen lauffaehige Version, wie wir am Ende sehen werden.

# Das Projekt Overlay Network Test

Der Overlay Network Test ist ein Programm, welches die Netzwerkverbindung des Overlay Network (CNI) im Kubernetes Cluster ueberprueft. Mitunter kann es dort zu Stoerungen kommen und bevor eine stunden- oder tagelange Fehlersuche beginnt, kann man einfach ueberpruefen, ob sich die Knoten eines Clusters "gegenseitig sehen", aber nicht auf dem Netzwerk der Knoten sondern dem der Container/PODs. Dazu installiert man ein DaemonSet, ein Pod, der auf jedem Knoten eines Clusters laeuft, und pingt von dort alle anderen Knoten und sich selbst an. Bei einem funktionierenden Overlay Network sollte das problemlos funktionieren. Geht das auf einem oder allen Knoten nicht, hat man ein Problem und kann das zielgerichtet untersuchen.

<strong>Programmierung</strong>

Quelle: <a href="https://github.com/eumel8/overlaytest/tree/0.0.3">https://github.com/eumel8/overlaytest</a>

Die Welt beginn mit 

<code>package main</code>

Nur mit "main" bekommt man zum Schluss ein ausfuehrbares Binary gebaut. Man koennte auch "package overlay" schreiben. Dann waere das ein Package, was Teil einer anderen Applikation ist. Aber wir wollten ja alles in einer Datei schreiben.

Nun folgt die Liste anderer Packages, die wir fuer unsere Programmlogik benoetigen:

```go
import (
	"context"
	"flag"
	"fmt"
	apps "k8s.io/api/apps/v1"
	core "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	meta "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/tools/remotecommand"
	"k8s.io/client-go/util/homedir"
	"os"
	"path/filepath"
	"time"
)
```

Eine ganze Menge. Einige kennen wir schon vom Go-Kurs, wie etwa das "fmt" fuer formatierte Textausgabe. Die anderen alle sind Packages vom Kubernetes Projekt, die wir verwenden. Angesprochen werden diese im Programm mit "kubernetes", "scheme", "clientcmd", "remotecommand" usw..
Hat man mehrere mit demselben Namen oder moechte man den langen Namen abkuerzen, kann man ein Alias setzen wie bei "core/v1" oder "meta/v1". Man hat vielleicht eine Vorstellung.
Dokumentiert sind die Packages allesamt wie etwa das <a href="https://pkg.go.dev/k8s.io/api/core/v1">Kubernetes API Core v1</a> und so kann man bei Go erstmal eine ganze Menge lesen. Auch die <a href="https://go.dev/">Go Homepage</a> gibt eine ganze Menge her.
Sehr hilfreich ist auch die Suche bei Github. Wenn man die Anwendung einer bestimmten Funktion sucht, dies auf "code" einschraenkt und dann die Programmiersprache "go", findet man viele nuetzliche Beispiele. Ohne diese Unterstuetzung waere ich nicht so weit gekommen, also alles sehr hilfreich.

Weiter gehts in unserem Programm:

```go
func main() {
	var kubeconfig *string
	namespace := "kube-system"
	app := "overlaytest"
```

Die Funktion `main` ist der Startpunkt des Programms. Hier auch gleich nochmal der Hinweis auf meine minderbemittelten Programmierkenntnisse. Normalerweise würde man jede Logik in eine andere Funktion packen und diese von der "main" aufrufen. 
In dieser Version ist alles in einer Funktion enthalten. Geht auch, jedoch verbaut man sich den Weg der Wiederverwendbarkeit und auch die Uebersichtlichkeit wird leiden, ist aber in dieser Phase noch von Vorteil. 
Am Anfang deklarieren wir also 3 Variablen. Die erste `kubeconfig` ist ein Pointer auf einen String, die anderen zwei sind in der verkuerzten Schreibweise `:=` Festlegungen von Namespace Name und App Name (die Namen kann man also spaeter nicht mehr aendern).

Es folgt die Einbindung der Credentials von kube-config, damit wir uns zum Kubernetes Cluster verbinden koennen. Der Code ist gnadenlos kopiert von <a href="https://github.com/kubernetes/client-go/tree/master/examples/out-of-cluster-client-configuration">Beispiel Ordner</a>, weswegen der hier nicht weiter Beachtung findet. Nach den paar Zeilen haben wir eine Verbindung zu Kubernetes, was schon mal ein kleiner Erfolg ist. Fuer alle anderen Schritte koennen wir die Variable "clientset" verwenden, um mit dem Cluster "zu sprechen":

```go
	daemonsetsClient := clientset.AppsV1().DaemonSets(namespace)
	daemonset := &apps.DaemonSet{
		ObjectMeta: meta.ObjectMeta{
			Name: app,
		},
		Spec: apps.DaemonSetSpec{
			Selector: &meta.LabelSelector{
				MatchLabels: map[string]string{
					"app": app,
				},
			},
			Template: core.PodTemplateSpec{
				ObjectMeta: meta.ObjectMeta{
					Labels: map[string]string{
						"app": app,
					},
				},
				Spec: core.PodSpec{
					Containers: []core.Container{
						{
							Args:            []string{"tail -f /dev/null"},
							Command:         []string{"sh", "-c"},
							Name:            app,
							Image:           "mtr.external.otc.telekomcloud.com/mcsps/swiss-army-knife:latest",
							ImagePullPolicy: "Always",
						},
					},
				},
			},
		},
	}

```

Puh, wir erstellen ein Objekt "DaemonsetClient", welches so ein bischen wie eine Kubernetes Resource wie ein DaemonSet aussieht Da gibts ein Image Namen, eine ImagePullPolicy ... und wir verwenden die `app` Variable fuer den Namen.

Jetzt wird tatsaechlich das DaemonSet im Cluster erstellt:

```go
	fmt.Println("Creating daemonset...")
	result, err := daemonsetsClient.Create(context.TODO(), daemonset, meta.CreateOptions{})
```

Das klappt dann genau einmal, denn wenn wir das Programm nochmal aufrufen, gibts das DaemonSet natuerlich schon. 
Also muss man diesen Fehler abfangen und eine Aktion daraus ableiten. Entweder Programmabbruch oder, wie hier, das alte DaemonSet loeschen:

```go
	if errors.IsAlreadyExists(err) {
		fmt.Println("daemonset already exists, deleting ... & exit")
		deletePolicy := meta.DeletePropagationForeground
		if err := daemonsetsClient.Delete(context.TODO(), app, meta.DeleteOptions{
			PropagationPolicy: &deletePolicy,
		}); err != nil {
			panic(err)
		}
		os.Exit(1)
	} else if err != nil {
		panic(err)
	}
	fmt.Printf("Created daemonset %q.\n", result.GetObjectMeta().GetName())
```

Alle anderen Fehler fuehren tatsaechlich zum Programmabbruch. Die Fehlermeldung ist uebrigens auch eine <a href="https://pkg.go.dev/k8s.io/apimachinery/pkg/api/errors#IsAlreadyExists">Funktion aus einem Package</a>, was wir hier nutzen.

Bevor wir mit dem eigentlichen Programm weitermachen koennen, muessen wir warten, bis das DaemonSet tatssaechlich deployt ist. Das kann etwas dauern. Man behilft sich hier mit einem "sleep" oder, eleganter, "wait", was in einer extra Funktion aufgerufen wird.

Wir machen das direkt im Anschluss:

```go
	for {
		obj, err := clientset.AppsV1().DaemonSets(namespace).Get(context.TODO(), "overlaytest", meta.GetOptions{})
		if err != nil {
			fmt.Println("Error getting daemonset: %v", err)
			panic(err.Error())
		}
		if obj.Status.NumberReady != 0 {
			fmt.Println("all pods ready")
			break
		}
		time.Sleep(2 * time.Second)
	}
```

Hier ist auch schon eine Schwachstelle. Das `for {}` wuerde ewig laufen, wenn das DaemonSet nicht `ready`  wird. Andererseits, ohne dem DaemonSet kommen wir auch nicht weiter. Nuetzt also nicht, das einfach zu uebergehen.

Jetzt kann es noch passieren, dass die PODs noch gar keine IP-Adresse haben. Das sind zwar nur Millisekunden, aber unser Programm wuerde gnadenlos weiterlaufen und ohne IP-Adresse kann ich auch keinen POD anpingen bzw. wuerde der Test fehlschlagen. Also ueberpruefen wir nochmal, ob alle PODs eine IP-Adresse haben:

```go
	pods, err := clientset.CoreV1().Pods(namespace).List(context.TODO(), meta.ListOptions{LabelSelector: "app=overlaytest"})
	if err != nil {
		panic(err.Error())
	}
	fmt.Printf("There are %d nodes in the cluster\n", len(pods.Items))

	// wait here again if all PODs become an ip-address
	fmt.Println("checking pod network...")
	for _, pod := range pods.Items {
		for {
			if pod.Status.PodIP != "" {
				fmt.Println(pod.ObjectMeta.Name, "ready")
				break
			}
		}
	}
	fmt.Println("all pods have network\n")
```

Erst jetzt kanns weitergehen und wir kommen zum Hauptteil des Programms:

```go
	fmt.Println("=> Start network overlay test\n")
	for _, upod := range pods.Items {
		for _, pod := range pods.Items {
			cmd := []string{
				"sh",
				"-c",
				"ping -c 2 " + upod.Status.PodIP + " > /dev/null 2>&1",
			}
			req := clientset.CoreV1().RESTClient().Post().Resource("pods").Name(pod.ObjectMeta.Name).Namespace(namespace).SubResource("exec").VersionedParams(&core.PodExecOptions{
				Command: cmd,
				Stdin:   true,
				Stdout:  true,
				Stderr:  true,
			}, scheme.ParameterCodec)

			exec, err := remotecommand.NewSPDYExecutor(config, "POST", req.URL())
			if err != nil {
				fmt.Println("error while creating Executor: %v", err)
			}

			err = exec.Stream(remotecommand.StreamOptions{
				Stdin:  os.Stdin,
				Stdout: os.Stdout,
				Stderr: os.Stderr,
				Tty:    false,
			})
			if err != nil {
				fmt.Println(upod.Spec.NodeName, " can NOT rach ", pod.Spec.NodeName)
			} else {
				fmt.Println(upod.Spec.NodeName, " can reach ", pod.Spec.NodeName)
			}

		}
	}
	fmt.Println("=> End network overlay test\n")
	fmt.Println("Call me again to remove installed cluster resources\n")
```

Wir haben zwei `for{}` Schleifen, die um die Nodes bzw. PODs laufen und im Container das "ping" Kommando absetzen.  Gibt es dabei einen Fehler, ist diese Verbindung als nicht erfolgreich deklariert und die entsprechende Meldung wird ausgegeben. Bei Erfolg erfolgt auch eine Meldung.
Zu Beachten ist hier, dass wir den REST Client verwenden, um ueber die API den Exec-Befehl in dem POD zu realisieren. Auf der API selber gibt es dafuer keine Resource. Man wird in der API-Dokumentation auf ein "ExecAction" stossen, aber damit ist das Exec IM Container eines Deployments gemeint, also etwas fuer eine LIvenessprobe als Healthcheck Kommando.

Einige hilfreiche Kommandos zum Bauen des Programms. Damit wir diese ausfuehren koennen, brauchen wir das <a href="https://go.dev/dl/go1.17.5.linux-amd64.tar.gz">Programm Go</a>, was aber selber nur ein Binary ist, was man leicht herunterladen und an beliebiger Stelle installieren kann.

```bash
go fmt overlay.go
```

Damit wird der Quellcode ordentlich formatiert und syntaktisch ueberprueft. Unordentlicher Code wird also ordentlich gemacht, importierte Packages werden auf ihre Notwendigkeit und Vorhandensein geprueft.

```bash
go mod init overlay.go
go mod tidy
```

Damit werden alle abhaengigen Packages heruntergeladen und die Dateien go.mod und go.sum erstellt.

```bash
go build -o overlay -v overlay.go
```

Der eigentliche Bau des Programms. Dies sollte ohne Fehler ausgefuehrt werden und das Binary "overlay" erstellen.

Programm starten und testen:

```bash
./overlay
Welcome to the overlaytest.


Creating daemonset...
Created daemonset "overlaytest".
all pods ready
There are 2 nodes in the cluster
checking pod network...
overlaytest-2x2bh ready
overlaytest-kvq8p ready
all pods have network

=> Start network overlay test

k3s-test-server-1  can reach  k3s-test-server-1
k3s-test-server-1  can reach  k3s-test-server-2
k3s-test-server-2  can reach  k3s-test-server-1
k3s-test-server-2  can reach  k3s-test-server-2
=> End network overlay test

Call me again to remove installed cluster resources

```

# Release Day (31.12.2021)

Wenn das Programm lokal kompiliert und lauffaehig ist, ist es an der Zeit, es auf die Menschheit loszulassen. Der Code ist eh schon im public Github Repo, jetzt will es vielleicht jemand herunterladen und anwenden.
<a href="https://docs.github.com/en/actions">Github Action</a> ueberzeugt mich ja schon lange und hat auch meine Pipelines bei Travis abgeloest oder Container Build bei Docker Hub. Dauert viel zu lange, ist viel zu limiert und geht bei Github Action viel schneller. Aber ich will auch nicht zu viel Werbung machen, damit nicht zu viele Leute wechseln. Das alles kostenlos zur Verfuegung steht, ist sicherlich nicht fuer lange.
Nie war der Release Prozess so einfach wie bei Github. Wir suchen uns im Marktplatz ein Github Action <a href="https://github.com/marketplace/actions/go-release-binaries">zum Bauen von Go Programmen und Speichern von Artefacts</a>, erstellen ueber die Weboberflaeche ein Release Draft mit einem neuen Tag, der triggert das Action und wenn es erfolgreich laeuft, kann man unter <a href="https://github.com/eumel8/overlaytest/releases">Releases</a> die Software fertig herunterladen, fuer Linux, Windows und Arm (Raspberry).

Ist das nicht toll?

<img src="/images/2021-12-31-3.png" />
