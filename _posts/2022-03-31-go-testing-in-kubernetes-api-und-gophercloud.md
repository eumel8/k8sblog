---
layout: post
tag: de
title: "Go Testing in Kubernetes API und Gophercloud"
subtitle: "Ein Go Programm zu schreiben, ist auf dem Hello-World-Level genauso einfach wie in anderen Programmiersprachen. Wie sieht es mit Code Testing aus? Go hält auch hier ein Bordmittel bereit, um Code zu formatieren und zu testen. Lesen wir an zwei Beispielen wie dies zu bewerkstelligen ist."
date: 2022-03-31
background: '/images/gophercloud.png'
twitter: '/images/2021-12-31-2.png'
author: eumel8
---

# Go Format

Mit `go fmt` bekommt man seinen Quellcode schonmal automatisch formatiert. Also die Zeilenabstaende sind richtig eingerueckt,ueberfluessige Leerzeichen werden entfernt usw. Also schon mal nicht schlecht.

# Go Lint

Go Linter sind extra Programme, die kein Bestandteil von Go sind. Sie werden etwa installiert mit

```shell
curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.45.2
```

<a href="https://github.com/golangci/golangci-lint">Dieses Programm</a> enthaelt mehrere Linter mit unterschiedlichen Aufgaben. Die Ausgabe bei unserem <a href="https://github.com/eumel8/overlaytest/">overlaytest</a> sieht etwa so aus:

```shell
$ golangci-lint run
overlay.go:125:5: S1002: should omit comparison to bool constant, can be simplified to `!*reuse` (gosimple)
        if *reuse != true {
           ^
overlay.go:78:2: printf: `fmt.Println` arg list ends with redundant newline (govet)
        fmt.Println("Welcome to the overlaytest.\n\n")
        ^
overlay.go:147:5: printf: fmt.Println call has possible formatting directive %v (govet)
                                fmt.Println("Error getting daemonset: %v", err)
                                ^
overlay.go:182:2: printf: `fmt.Println` arg list ends with redundant newline (govet)
        fmt.Println("all pods have network\n")
        ^
overlay.go:185:2: printf: `fmt.Println` arg list ends with redundant newline (govet)
        fmt.Println("=> Start network overlay test\n")
        ^
overlay.go:207:5: printf: fmt.Println call has possible formatting directive %v (govet)
                                fmt.Println("error while creating Executor: %v", err)
                                ^
overlay.go:187:2: SA4006: this value of `err` is never used (staticcheck)
        pods, err = clientset.CoreV1().Pods(namespace).List(context.TODO(), meta.ListOptions{LabelSelector: "app=overlaytest"})
        ^
```

Da wird also schon etwas genauer hingesehen, ob Variablen Sinn machen oder ueberhaupt nach der Defintion verwendet werden. Sehr hilfreich.

# Go Testing

Was wir aber jetzt genau wissen wollen: Funktioniert denn jetzt mein Programm? Oder meine Funktion? Oder der Funktionsaufruf? Dazu gibt es Unittests und das <a href="https://pkg.go.dev/testing">Go Paket Testing</a>. Im Paket gibt es auch Beispiele:

```go
func TestAbs(t *testing.T) {
    got := Abs(-1)
    if got != 1 {
        t.Errorf("Abs(-1) = %d; want 1", got)
    }
}
```

Es wird also die Ausgabe einer Funktion mit einem zu erwartenden Wert verglichen und wenn der okay ist, wurde der Test bestanden. Die Anzahl der mit diesem Test abgedeckten Code-Zeilen heisst Coverage und praesentiert somit eine Art Qualitaetssiegel fuer das Programm.

# Kubernetes API Test

Das Kernstueck unserers overlaytest Programms ist ein DaemonSet, was im zu testenden Kubernetes-Cluster deployt wird. Im Kubernetes Projekt gibt es den Fake Client. Dieser kann saemtliche API-Endpunkte und Resourcen nachbilden und antwortet mit entsprechenden Rueckgabewerten, ohne dass man einen Kubernetescluster oder andere Resourcen brauch. Wir koennen zum Beispel einen Pod erstellen und anschliessend abfragen, ob er existieren wuerde:

```go
package pod

import (
        "context"
        "testing"

        core "k8s.io/api/core/v1"
        meta "k8s.io/apimachinery/pkg/apis/meta/v1"
        "k8s.io/client-go/kubernetes/fake"
)

func TestPod(t *testing.T) {
        client := fake.NewSimpleClientset()
        p := &core.Pod{ObjectMeta: meta.ObjectMeta{Name: "my-pod"}}
        result, err := client.CoreV1().Pods("test-ns").Create(context.TODO(), p, meta.CreateOptions{})
        if err != nil {
                t.Fatalf("error injecting pod add: %v", err)
        }

        t.Logf("Got pod from manifest: %v", p.ObjectMeta.Name)
        t.Logf("Got pod from result: %v", result.ObjectMeta.Name)
}
```

```shell
$ go test pod_test.go -v
=== RUN   TestPod
    pod_test.go:20: Got pod from manifest: my-pod
    pod_test.go:21: Got pod from result: my-pod
--- PASS: TestPod (0.00s)
PASS
ok      command-line-arguments  0.034s
```

Ziemlich einfach, oder? Unser DaemonSet  koennen wir auch testen:

```go
package daemonset

import (
        "context"
        "testing"

        apps "k8s.io/api/apps/v1"
        core "k8s.io/api/core/v1"
        meta "k8s.io/apimachinery/pkg/apis/meta/v1"
        "k8s.io/client-go/kubernetes/fake"
)

func TestDaemonset(t *testing.T) {

        var (
                app         = string("overlaytest")
                image       = string("mtr.external.otc.telekomcloud.com/mcsps/swiss-army-knife:latest")
                graceperiod = int64(1)
                user        = int64(1000)
                privledged  = bool(true)
                readonly    = bool(true)
        )

        client := fake.NewSimpleClientset()

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
                                                        Image:           image,
                                                        ImagePullPolicy: "IfNotPresent",
                                                        SecurityContext: &core.SecurityContext{
                                                                AllowPrivilegeEscalation: &privledged,
                                                                Privileged:               &privledged,
                                                                ReadOnlyRootFilesystem:   &readonly,
                                                                RunAsGroup:               &user,
                                                                RunAsUser:                &user,
                                                        },
                                                },
                                        },
                                        TerminationGracePeriodSeconds: &graceperiod,
                                        Tolerations: []core.Toleration{{
                                                Operator: "Exists",
                                        }},
                                        SecurityContext: &core.PodSecurityContext{
                                                FSGroup: &user,
                                        },
                                },
                        },
                },
        }

        result, err := client.AppsV1().DaemonSets("kube-system").Create(context.TODO(), daemonset, meta.CreateOptions{})
        if err != nil {
                t.Fatalf("error injecting pod add: %v", err)
        }

        if daemonset.ObjectMeta.Name != result.ObjectMeta.Name {
                t.Logf("Got from manifest: %v", daemonset.ObjectMeta.Name)
                t.Logf("Got from result: %v", result.ObjectMeta.Name)
                t.Fatalf("result and manifest are not the same")
        }
}
```

```shell
$ go test daemon_test.go -v
=== RUN   TestDaemonset
--- PASS: TestDaemonset (0.00s)
PASS
ok      command-line-arguments  0.020s
```

# Gophercloud Testing

<a href="https://github.com/gophercloud/gophercloud"> Gophercloud</a> ist ein Go Framework zur Verbindungsaufnahme zu einer OpenStack API. Die Testsuite in diesem Framework bildet nun diese API durch initiieren eigener HTTP-Server nach und hinterlegt die zu erwartenden Antworten.

Schauen wir uns dazu diesen <a href="https://github.com/opentelekomcloud/gophertelekomcloud/commit/45e5435f15218efa296e1a041d1a8536e0b09170#diff-fcfb7ce237e0c8ddf07ef5998d86096e40dfd8159df60fc86988a28bb8cd0b27">Commit</a> an. Es geht um den Restore eines Backups einer MySQL Datenbank in der OpenTelekomCloud. Die OpenTelekomCloud basiert auf OpenStack und unterhaelt dazu den eigenen Fork des Gophercloud SDK. 
Zurueck zum Beispiel:

```go
func TestRestoreRequestPITR(t *testing.T) {
	th.SetupHTTP()
	t.Cleanup(func() {
		th.TeardownHTTP()
	})
	th.Mux.HandleFunc("/instances/recovery", func(w http.ResponseWriter, r *http.Request) {
		th.TestMethod(t, r, "POST")
		th.TestHeader(t, r, "X-Auth-Token", client.TokenID)

		w.WriteHeader(http.StatusAccepted)
		_, _ = fmt.Fprint(w, expectedPITRResponse)
	})

	opts := exampleRestorePITROpts()
	backup, err := backups.RestorePITR(client.ServiceClient(), opts).Extract()
	th.AssertNoErr(t, err)
	tools.PrintResource(t, backup)
}
```

Diese Funktion testet die PointInTimeRecovery Funktion (PITR). Vom Testhelper (th) wird der Webserver gestartet. Auf die URI `/instances/recovery` wird `exampleRestorePITROpts` geposted. Diese enthaelt die Instanz-ID und Restore-Zeitpunkt, was durch diese Funktion zurueckgegeben wird:

```go
func exampleRestorePITROpts() backups.RestorePITROpts {
	return backups.RestorePITROpts{
		Source: backups.Source{
			InstanceID:  "d8e6ca5a624745bcb546a227aa3ae1cfin01",
			RestoreTime: 1532001446987,
			Type:        "timestamp",
		},
		Target: backups.Target{
			InstanceID: "d8e6ca5a624745bcb546a227aa3ae1cfin01",
		},
	}

}
```

Die Antwort steht in dieser const und beinhaltet einfach eine JobID:

```go
const expectedPITRResponse = `
{
  "job_id": "4c56c0dc-d867-400f-bf3e-d025e4fee686"
}
`
```

Sind Anfragen und dazugehoerige Antworten gleich, ist der Test bestanden und die darin enthaltene Funktion `backups.RestorePITR` ausreichend getestet.

# Mocking

Das Nachahmen solcher API-Funktionalitaeten nennt man auch Mocking, das Verteilen verschiedener Anfragen Muxing. Eine Funktion die beides kann, waere also ein MockMuxer von <a href="https://github.com/eumel8/otc-rds-client/blob/master/rds_test.go">rds_test.go</a>:

```go
func MockMuxer() {
	mux := http.NewServeMux()

	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		switch r.Method {
		case "GET":
			w.Header().Add("Content-Type", "application/json")
			w.WriteHeader(http.StatusOK)
			_, _ = fmt.Fprint(w, ProviderGetResponse)
		case "POST":
			w.Header().Add("X-Subject-Token", "dG9rZW46IDEyMzQK")
			w.Header().Add("Content-Type", "application/json")
			w.WriteHeader(http.StatusCreated)
			_, _ = fmt.Fprint(w, ProviderPostResponse)
		}
	})

// ...
	mux.HandleFunc("/jobs", func(w http.ResponseWriter, r *http.Request) {
		switch r.Method {
		case "GET":
			w.Header().Add("Content-Type", "application/json")
			w.WriteHeader(http.StatusOK)

			_, _ = fmt.Fprint(w, RdsJobResponse)
		}
	})

	fmt.Println("Listening...")

	var retries int = 3

	for retries > 0 {
		err := http.ListenAndServe("127.0.0.1:50000", mux)
		if err != nil {
			fmt.Println("Restart http server ... ", err)
			retries -= 1
		} else {
			break
		}
	}

}
```

Die Antworten zu den verschiedenen GET und POST Anfragen befinden sich wieder in const, hier etwa die Antwort auf eine ProviderGet Anfrage:

```go
const ProviderGetResponse = `
{
	"version": {
		"media-types": [{
			"type": "application/vnd.openstack.identity-v3+json",
			"base": "application/json"
		}],
		"links": [{
			"rel": "self",
			"href": "http://127.0.0.1:50000/v3/"
		}],
		"id": "v3.6",
		"updated": "2016-04-04T00:00:00Z",
		"status": "stable"
	}
}
`
```

Diese liefert also die Adresse unserer Pseudo OpenStack API zurueck.

Testen kann man die Authentifizierung unseres OpenStack clients so:

```go
func Test_getProvider(t *testing.T) {
	go MockMuxer()

	err := os.Setenv("OS_USERNAME", "test")
	th.AssertNoErr(t, err)
	err = os.Setenv("OS_USER_DOMAIN_NAME", "test")
	th.AssertNoErr(t, err)
	err = os.Setenv("OS_PASSWORD", "test")
	th.AssertNoErr(t, err)
	err = os.Setenv("OS_IDENTITY_API_VERSION", "3")
	th.AssertNoErr(t, err)
	err = os.Setenv("OS_AUTH_URL", "http://127.0.0.1:50000/v3")
	th.AssertNoErr(t, err)

	provider := getProvider()
	defer getProvider()

	p := &golangsdk.ProviderClient{
		UserID: "91dca41cc55e4614aaca83b78af8ddc5",
	}
	th.CheckDeepEquals(t, p.UserID, provider.UserID)
	fmt.Println("IdentityEndpoint: ", provider.IdentityEndpoint)
	return
}
```

Wie man sieht, koennen solche Tests sehr langwierig werden im Code. Deswegen ist es wichtig zu erkennen, welche Tests das Framework schon bereitstellt. Oder selber ein Testframework zu erstellen, um dieses auch wiederverwenden zu koennen. Und alles nur, um den Code pseudomatisch zu ueberpruefen. 
Der naechste Schritt waeren <a href="https://github.com/opentelekomcloud/gophertelekomcloud/blob/devel/acceptance/README.md">Acceptance Tests</a>. Ab hier werden Code oder Funktionen am "lebenenden" Objekt getestet, es bedarf also einer echten OpenStack, bzw. OpenTelekomCloud API, um etwa ECS zu erstellen oder wie oben, ein Backup einer RDS Instanz wiederherzustellen. 

Fazit: Go Testing stellt einen deutlichen Qualitaetssprung in der Softwareprogrammierung dar. Nicht nur, dass man Code besser versteht, man kann ihn auch im Trockendock oder auf hoher See ausprobieren und sehen was er verspricht. Ein tiefgreifendes Verstaendnis kommt hinzu, genau wie Transparenz.

Viel Spass beim Testen
