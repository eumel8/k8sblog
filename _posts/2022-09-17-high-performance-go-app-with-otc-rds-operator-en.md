---
layout: post
tag: en
title: "High Performance Go Lang Apps with OTC RDS Operator"
subtitle: "IT products that require explanation or crazy technologies become a little clearer in their function if they can be tested in practice. A demo app is helpful for this"
date: 2022-09-17
background: '/images/high-performance-golang.png'
twitter: '/images/2022-05-27-1.png'
author: eumel8
---

Demonstration of how to use Kubernetes incl. Container, Pod, Deployment, Service, and Ingress can be found for example <a href="https://github.com/mcsps/use-cases/blob/master/demoapp.yaml">here</a>, with external volume also <a href="https://github.com/mcsps/use-cases/blob/master/demoapp_volume.yaml">here</a>. The result can be viewed in browser from every place on the Internet.

The <a href="https://github.com/eumel8/otc-rds-operator">OTC RDS Operator</a> can deploy OTC RDS Databases from a Kubernetes Cluster. With Autopilot function the instance scaled independent. But how can I include this resource in an application? For this case the operator has it's own <a href="https://github.com/eumel8/oro-demoapp">demo app</a>.  This app based on a model with a web application and MySQL backend, but is developed in Go with some extra features:

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

The demo app is running in the cluster, has automatically access to the Kubernetes API. In the function for database connect, is this access called and is looking for RDS resources with the corresponding instance in the same namespace, waits until the status is 'ACTIVE', unless the app is new deployed, get credentials which are defined in the RDS object and knows hostname/IP-address, which the OTC places for the RDS instance and henceforth in the status of the RDS object in the Kubernetes cluster is. Behold, connection is sucessful, the app used the database.

What works in the laboratory does not necessarily have to work in production. There is a <a href="https://github.com/ddosify/ddosify/releases">Go App Ddosify</a>, with which load scenarios for web applications can be simulated. The program is also available as an online service, but is chargeable after a short test phase. The program runs in many locations around the world from which you can test its service. We can also download the binary and test our demo app from our work station:

```bash
$ ddosify -t https://oro-demoapp.mcsps.telekomcloud.com/insert -m POST -h "Content-Type: application/x-www-form-urlencoded" -b "name=value1&city=value2" -l incremental -n 200000 -d 1200 -T 1

# single curl
curl -X POST https://oro-demoapp.mcsps.telekomcloud.com/insert
   -H "Content-Type: application/x-www-form-urlencoded" 
   -d "name=value1&city=value2"
```

200.000 new database entries in 20 minutes (therefor the Autopilot of the RDS Operate starts after 15 min).
That's too much for our app:

First error: MySQL - too many connections.
Can be changed in a normal database with variables:


```bash
mysql> set global max_connections = 2000;
```

But it doesn't work in the OTC and has to be set in the parameter group. The default is 800 in the MySQL default parameter group. It's a bit cumbersome to set in the OTC web console. Neither the operator nor the openstack client currently offer a suitable option. Then immediately reboot the database.

Next error: Oomkilled POD. The low limits of the test app is reached. This can be increased in the Statefulset:

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

Next error: too many open files. Probably in the container, because the directory for storing the images is always reloaded with each access. So it is better to scale the application in width:

```bash
$ kubectl -n rds1 scale --replicas=5 statefulsets.apps demoapp
```

With attached storage, this only works with multi-attach-capable or RWX storage such as NFS clients.
For database performance testing, we can forgo the permanent image storage and place the directory in the POD's RAM memory by replacing the PVC entry with

```yaml
      volumes:
      - emptyDir:
          medium: Memory
        name: demoapp-volume
```

The demo app is already running quite well, the RDS may actually be scaled

```yaml
Events:
  Type    Reason  Age                From              Message
  ----    ------  ----               ----              -------
  Normal  Update  34m (x52 over 9h)  otc-rds-operator  This instance set autopilot.
  Normal  Update  29m (x2 over 29m)  otc-rds-operator  This instance is scaling to rds.mysql.s1.medium.ha
```
However, there are still enough adjustments to be made to the components involved in our demo app: load balancers, ingress, worker nodes. Ddosify counts the error types and gives a report at the end of its work.

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

With this one can then examine the installation and improve the quality of the service

Happy Testing
