---
layout: post
tag: en
title: "Go Testing in Kubernetes API and Gophercloud"
subtitle: "Writing a Go program is easy like in other programming languages on the Hello-World-Level. How about Code Testing? Go delivers some on-board tools to format and test code. Let's use 2 examples to show how this can be done."
date: 2022-03-31
background: '/images/gophercloud.png'
twitter: '/images/2021-12-31-2.png'
author: eumel8
---

# Go Format

With `go fmt` we can format our source code automatically. Spaces and tabs are in the correct order, redundant spaces are removed etcd. Not too bad for the first shot.

# Go Lint

Go Linter are extra programs, which are not part of Go. To install like this:

```shell
curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.45.2
```

<a href="https://github.com/golangci/golangci-lint">This program</a> includes multiple linter with different task. The output of our <a href="https://github.com/eumel8/overlaytest/">overlaytest</a> looks like this:

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

There is a closer look for sense on variables or the usage after definition. Very helpful.

# Go Testing

What we want to know now: Is my program working? Or my function? Or the call of a function? For this there are unit tests and the <a href="https://pkg.go.dev/testing">Go Package Testing</a>. In the package are examples included:

```go
func TestAbs(t *testing.T) {
    got := Abs(-1)
    if got != 1 {
        t.Errorf("Abs(-1) = %d; want 1", got)
    }
}
```

The output of a function will compare with an expected value and if this is okay, the test will pass. The count of lines of code, which is with this test covered, is named Coverage and present a seal of quality for this program.

# Kubernetes API Test

The main part of the overlaytest program is a DaemonSet, which must be deployed on the Kubernetes target cluster. The community project has a fake client. This can replicate all API endpoints and resources and replies with corresponding values without any real cluster or other real resources. For example we create a POD and request the API after that if the POD exists:

<script src="https://gist.github.com/eumel8/6a2a0d50b073f30cee24822aced9a5aa.js"></script>

```shell
$ go test pod_test.go -v
=== RUN   TestPod
    pod_test.go:20: Got pod from manifest: my-pod
    pod_test.go:21: Got pod from result: my-pod
--- PASS: TestPod (0.00s)
PASS
ok      command-line-arguments  0.034s
```

Pretty easy, isn't it? We can also test our DaemonSet:

<script src="https://gist.github.com/eumel8/d35368b3fedf2450673fa67edd26de13.js"></script>

```shell
$ go test daemon_test.go -v
=== RUN   TestDaemonset
--- PASS: TestDaemonset (0.00s)
PASS
ok      command-line-arguments  0.020s
```

# Gophercloud Testing

<a href="https://github.com/gophercloud/gophercloud"> Gophercloud</a> is a Go framework for connection to an OpenStack API. This test suite in this framework initiate an own HTTP server to replicate the expected values after request the test server.

For example look at this <a href="https://github.com/opentelekomcloud/gophertelekomcloud/commit/45e5435f15218efa296e1a041d1a8536e0b09170#diff-fcfb7ce237e0c8ddf07ef5998d86096e40dfd8159df60fc86988a28bb8cd0b27">Commit</a>. Context is a restore of a backup of a MySQL database in OpenTelekomCloud. The OpenTelekomCloud is based on OpenStack and maintains it's own fork of Gophercloud SDK. 
Back to the example:

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

This function will test the PointInTimeRecovery function (PITR). The Testhelper (th) starts the web server. To the URI `/instances/recovery` we post `exampleRestorePITROpts`. This contains the InstanceID and RestoreTime, what the function returns:

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

The answer of this request is this const and contains simply the JobID:

```go
const expectedPITRResponse = `
{
  "job_id": "4c56c0dc-d867-400f-bf3e-d025e4fee686"
}
`
```

Are requests and corresponding answers similar, the test is okay and the function `backups.RestorePITR` tested.

# Mocking

To emulate API functionalities is called Mocking, to distribute different requests is called Muxing. A function that can do both, is a MockMuxer from <a href="https://github.com/eumel8/otc-rds-client/blob/master/rds_test.go">rds_test.go</a>:

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

The answers to different GET and POST requests contain in const, i.e. the answer of ProviderGet request:

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

This delivers the address our Pseudo OpenStack API.

To test the authentication of our OpenStack client:

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

How to see, such kind of tests can be very long on the code. For this reason, it's important to know, which tests are already provided by a framework. Or create an own test framework to re-use. And only to test code pseudomatically.
The next step are <a href="https://github.com/opentelekomcloud/gophertelekomcloud/blob/devel/acceptance/README.md">Acceptance Tests</a>. Code and function are tested on "live" environments, a real OpenStack or OpenTelekomCloud API is required, to create an ECS for example, or, like above, to restore the backup of a RDS instance.

Summary: Go Testing represents a clear leap in quality in software programming. Not only will you understand code better, you can try it out in dry dock or at sea and see what it promises. A profound understanding is added, as is transparency.

Have fun with testing
