---
layout: post
tag: en
title: "Go Project: The Overlay Network Test"
subtitle: "One year ago I started to learn the programming language Go. Since them I reviewed multiple source code or wrote one or another patch, which was no more then a one-liner. Time to wrote the first own program."
date: 2021-12-31
background: '/images/gopher-over-sea.png'
twitter: '/images/2021-12-31-2.png'
author: eumel8
---

# Preamble

First of all: I am not a programmer, and probably will not be anymore. I've been in operations my entire working life. That's why I only have limited knowledge of software programming and always look at it from the perspective of the operation team, at the end the user of such software. Therefore, a few basics of software programming and optimization may be missing here in the project. On the other hand, what the heck: It's my first Go program!

# Why GO

<img src="/images/2021-12-31-1.png" />

Go has not been around as a programming language for that long, in fact 10 years now. It was created in 2011 and the first release saw the light of day in 2012.
It was developed by employees of Google out of dissatisfaction with the existing landscape of programming languages. Regardless of whether it is C or Java, you always have tons of dependencies on libraries that you have to install before the actual program starts. If everything depends on libc, you also have a dependency on certain operating systems. or kernel versions. In the end, you installed a whole heap of files, only to have a program with a function running.

Go cut off all of those old tugs. There is ONE program with ONE file. Still, this binary is slim ... well, it doesn't eat up gigabytes of memory now. Still, you don't have to do without libraries, go packages. However, these are integrated at development time and delivered ready-made in the program according to the functions used.
In the end you have a version that can run on very different platforms, as we will see at the end.


# The Project Overlay Network Test

The Overlay Network Test is a program that checks the network connection of the Overlay Network (CNI) in the Kubernetes cluster. Occasionally there can be malfunctions and before an hour or day long troubleshooting begins, you can simply check whether the nodes of a cluster "see each other", but not on the network of the nodes but that of the container / PODs. To do this, you install a DaemonSet, a pod that runs on every node of a cluster, and ping all other nodes and yourself from there. With a functioning overlay network, this should work fine. If this does not work on one or all of the nodes, you have a problem and can investigate it in a targeted manner.

<strong>Programming</strong>

Source: <a href="https://github.com/eumel8/overlaytest/tree/0.0.3">https://github.com/eumel8/overlaytest</a>

The world starts with:

<code>package main</code>

Only with "main" we become an executable binary at the end. We can also write "package overlay". Then we have a package as part of another application. But we want to write all of them in one file.

Now follows the list of packages, which we need for our program logic:

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

A whole lot. We already know some of them from the Go course, such as the "fmt" for formatted text output. The others are all packages from the Kubernetes project that we are using. These are addressed in the program with "kubernetes", "scheme", "clientcmd", "remotecommand" etc..
If you have several with the same name or if you want to abbreviate the long name, you can set an alias like "core/v1" or "meta/v1". Maybe you have an idea. The packages are all documented like the <a href="https://pkg.go.dev/k8s.io/api/core/v1"> Kubernetes API Core v1 </a> and so you can read a lot with Go. The <a href="https://go.dev/"> Go Homepage </a> also has a lot to offer.
The search on Github is also very helpful. If you look for the application of a certain function, restrict it to "code" and then the programming language "go", you will find many useful examples. Without this support I would not have come this far, so everything was very helpful.

Our program continues:

```go
func main() {
	var kubeconfig *string
	namespace := "kube-system"
	app := "overlaytest"
```

The function `main` is the starting point of the program. Here again the reference to my underprivileged programming skills. Normally you would pack every logic in a different function and call this from the "main".
In this version everything is contained in one function. This is also possible, but the way of reusability is blocked and the clarity will also suffer, but is still an advantage in this phase.
So at the beginning we declare 3 variables. The first `kubeconfig` is a pointer to a string, the other two are in the shortened notation `:=` specification of namespace name and app name (the names cannot be changed later).

This is followed by the integration of the kube-config credentials so that we can connect to the Kubernetes cluster. The code is mercilessly copied from <a href="https://github.com/kubernetes/client-go/tree/master/examples/out-of-cluster-client-configuration"> example folder </a>, which is why which is not taken into account here. After the few lines we have a connection to Kubernetes, which is a small success. For all other steps we can use the variable "clientset" to "talk" to the cluster:

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

Phew, we create an object "DaemonsetClient", which looks a bit like a Kubernetes Resource like a DaemonSet. There is an image name, an ImagePullPolicy ... and we use the `app` variable for the name.

Now the DaemonSet is actually created in the cluster:

```go
	fmt.Println("Creating daemonset...")
	result, err := daemonsetsClient.Create(context.TODO(), daemonset, meta.CreateOptions{})
```

That works exactly once, because when we call the program again, the DaemonSet is of course already there.
So you have to catch this error and take action from it. Either terminate the program or, as here, delete the old DaemonSet:

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

All other errors actually lead to the program being terminated. The error message is also a <a href="https://pkg.go.dev/k8s.io/apimachinery/pkg/api/errors#IsAlreadyExists"> function from a package </a>, which we use here.

Before we can continue with the actual program, we have to wait until the DaemonSet is actually deployed. It can take a while. You can make do with a "sleep" or, more elegantly, "wait", which is called in an extra function.

We do that right afterwards:

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

There is also a weak point here. The `for {}` would run forever if the DaemonSet is not `ready`. On the other hand, we won't get any further without the DaemonSet. So there is no point in simply ignoring it.

Now it can still happen that the PODs do not have an IP address at all. These are only milliseconds, but our program would continue to run mercilessly and without an IP address I cannot ping a POD or the test would fail. So let's check again whether all PODs have an IP address:

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

Now can we continue and we come to the main part of the program:

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

We have two `for {}` loops that run around the nodes or PODs and issue the "ping" command in the container. If there is an error, this connection is declared unsuccessful and the corresponding message is output. If this is successful, a message is also issued.
It should be noted here that we are using the REST client to implement the Exec command in the POD via the API. There is no resource for this on the API itself. You will come across an "ExecAction" in the API documentation, but this means the Exec IN THE container of a deployment, i.e. something for a lifecycle test as a health check command.

Some helpful commands for building the program. So that we can execute this, we need the <a href="https://go.dev/dl/go1.17.5.linux-amd64.tar.gz"> program Go </a>, which itself is just a binary is what is easy to download and install anywhere.

```bash
go fmt overlay.go
```

The source code is properly formatted and syntactically checked. Untidy code is made neatly, imported packages are checked for their necessity and existence.

```bash
go mod init overlay.go
go mod tidy
```

This will download all the dependent packages and create the go.mod and go.sum files.

```bash
go build -o overlay -v overlay.go
```

The actual construction of the program. This should be done without errors and create the binary "overlay".

Start and test the program:


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

When the program is compiled and running locally, it is time to let it loose on humanity. The code is already in the public Github Repo, now someone might want to download it and use it.
<a href="https://docs.github.com/en/actions"> Github Action </a> has convinced me for a long time and has also replaced my pipelines at Travis or Container Build at Docker Hub. Takes way too long, is way too limited and is much faster on Github Action. But I also don't want to advertise too much so that too many people don't switch. That everything is available for free is certainly not for long.
The release process has never been so easy as with Github. We are looking for a Github Action <a href="https://github.com/marketplace/actions/go-release-binaries"> to build Go programs and save artefacts </a>, created via the web interface a release draft with a new tag that triggers the action and if it runs successfully, the software can be downloaded from <a href="https://github.com/eumel8/overlaytest/releases">Releases</a> , for Linux, Windows and Arm (Raspberry).

Isn't that great?

<img src="/images/2021-12-31-3.png" />
