One GKE , project creation takes 7 minutes.

single node (first/basic) cluster creation takes 3 minutes.





configuration with gcloud command:
```
[demo@kworkhorse ~]$ gcloud container clusters get-credentials your-first-cluster-1 --zone us-central1-a --project kubernetes-demo-1-219619
ERROR: (gcloud.container.clusters.get-credentials) You do not currently have an active account selected.
Please run:

  $ gcloud auth login

to obtain new credentials, or if you have already logged in with a
different account:

  $ gcloud config set account ACCOUNT

to select an already authenticated account to use.


[demo@kworkhorse ~]$ gcloud auth login
Your browser has been opened to visit:

    https://accounts.google.com/o/oauth2/auth?redirect_uri=http%3A%2F%2Flocalhost%3A8085%2F&prompt=select_account&response_type=code&client_id=32798640559.apps.googleusercontent.com&scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.email+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcloud-platform+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fappengine.admin+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcompute+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Faccounts.reauth&access_type=offline


Created new window in existing browser session.
WARNING: `gcloud auth login` no longer writes application default credentials.
If you need to use ADC, see:
  gcloud auth application-default --help

You are now logged in as [kamranazeem@gmail.com].
Your current project is [None].  You can change this setting by running:
  $ gcloud config set project PROJECT_ID


[demo@kworkhorse ~]$ gcloud config set project kubernetes-demo-1-219619
Updated property [core/project].
[demo@kworkhorse ~]$ 
```

Now, run the gcloud get-credentials command again:

```
[demo@kworkhorse ~]$ gcloud container clusters get-credentials your-first-cluster-1 --zone us-central1-a --project kubernetes-demo-1-219619
Fetching cluster endpoint and auth data.
kubeconfig entry generated for your-first-cluster-1.
[demo@kworkhorse ~]$ 

```



kubectl should work:

```
[demo@kworkhorse ~]$ kubectl get nodes
NAME                                            STATUS    ROLES     AGE       VERSION
gke-your-first-cluster-1-pool-1-0cc95df2-r412   Ready     <none>    21m       v1.9.7-gke.6
[demo@kworkhorse ~]$
```




