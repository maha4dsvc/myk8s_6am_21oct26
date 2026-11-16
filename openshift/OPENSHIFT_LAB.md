# OpenShift Lab – run our booking app on OpenShift

**Goal:** deploy the same booking app we ran on EKS, the OpenShift way:
deploy an image → hit OpenShift's security rule → build a fixed image from Git → expose it with an HTTPS Route
→ scale, rebuild, roll back → bonus: install our Helm chart on OpenShift.

**You need:** a free Red Hat account and a browser. No cluster of your own.

```
openshift/
├── OPENSHIFT_LAB.md                <- this guide
├── OpenShift_for_Kubernetes.pptx
├── Dockerfile                      <- OpenShift-friendly image (fixes file permissions)
└── route.yaml                      <- Route for the Helm chart's Service (step 7)
```

---

## Step 1 – Get a cluster and log in

1. Open **https://developers.redhat.com/developer-sandbox**, log in with a Red Hat account, verify your phone,
   and click **Launch** for OpenShift. The web console opens.
2. Download `oc`: in the console click **?** (top right) → **Command line tools** → download for your OS,
   unzip, and put `oc` / `oc.exe` on your PATH.
3. In the console click **your name** (top right) → **Copy login command** → **Display Token**. Copy the
   `oc login` line and run it:

```bash
oc login --token=sha256~xxxxxxxx --server=https://api.<cluster>.openshiftapps.com:6443
oc whoami
oc project                      # Sandbox gives you one project, e.g. <username>-dev
```

> The token is a password. Don't share it or commit it. When it expires, `oc` says *Unauthorized*: copy a new one.

**No Sandbox?** OpenShift Local runs on your laptop: download `crc` and the pull secret from
console.redhat.com, then `crc setup` and `crc start` (needs 4 vCPU, ~11 GB RAM, 35 GB disk).
Then use `oc login -u developer -p developer https://api.crc.testing:6443` and `oc new-project booking`.

---

## Step 2 – Deploy the Docker Hub image as it is

```bash
oc new-app --name=booking-hub --image=docker.io/maha4iac/myonlinebooking:2
oc expose service/booking-hub
oc get pods
oc get route booking-hub
```

`oc new-app` read the image and created a **Deployment**, a **Service** (port 8080 from the image's `EXPOSE`)
and an **ImageStream**. `oc expose` created a **Route**. Compare with our hand-written `mydeploy.yml` and
`mySVCdeploy.yml`:

```bash
oc get deploy,svc,is,route
```

Open `http://<HOST>/` (Tomcat page) and then `http://<HOST>/mahaLogin/`.

---

## Step 3 – Find the problem: OpenShift does not run containers as root

```bash
oc get pod -l deployment=booking-hub -o jsonpath="{.items[0].spec.containers[0].securityContext}"
oc rsh deploy/booking-hub id                 # uid=1000xxxxxx gid=0(root) – NOT root
oc logs deploy/booking-hub | grep -iE "unable|denied|SEVERE"
```

What you should notice:

| Check | On EKS | On OpenShift |
|---|---|---|
| `id` inside the pod | `uid=0(root)` | `uid=1000xxxxxx` (random), `gid=0` |
| `/mahaLogin/` | Login page | Likely **404**: Tomcat could not unpack `mahaLogin.war` into `/usr/local/tomcat/webapps` |

Our image was built expecting root (`USER` is empty in its config). OpenShift's default
**SCC `restricted-v2`** forces a random non-root UID, so Tomcat can't write to its folders, which are owned by root.

> If `/mahaLogin/` still works for you, look at the logs anyway – the lesson is the UID. Images written for
> OpenShift must not depend on running as root.

Remove the broken app:

```bash
oc delete all -l app=booking-hub
```

---

## Step 4 – Build an OpenShift-friendly image from Git

`openshift/Dockerfile` starts from our image and gives **group 0** the same permissions as the owner, so any
random UID (which is always in group 0) can run Tomcat:

```dockerfile
FROM docker.io/maha4iac/myonlinebooking:2
USER root
RUN chgrp -R 0 /usr/local/tomcat && chmod -R g=u /usr/local/tomcat
USER 1001
```

Let OpenShift clone Git and build it – no Docker on your laptop:

```bash
oc new-build https://github.com/maha4dsvc/myK8s_8am_13oct26.git --context-dir=openshift --strategy=docker --name=booking
oc logs -f bc/booking                        # watch the build, ends with "Push successful"
oc get is booking                            # the new image in the internal registry
```

What was created:

| Object | Job |
|---|---|
| BuildConfig `booking` | Recipe: Git URL + folder + Docker strategy |
| Build `booking-1` | One run of the recipe (runs as a pod) |
| ImageStream `booking` | Points to the built image `booking:latest` |

> This needs `openshift/Dockerfile` pushed to `main` on GitHub. If you forked the repo, use your fork's URL.

---

## Step 5 – Deploy it and expose it with HTTPS

```bash
oc new-app booking
oc create route edge --service=booking
oc get route booking
```

Open **https://<HOST>/mahaLogin/** – the login page loads, over HTTPS, with no LoadBalancer and no NodePort.

Check the pod runs as non-root and the app was unpacked:

```bash
oc rsh deploy/booking id
oc rsh deploy/booking ls /usr/local/tomcat/webapps
```

**Web console:** switch to the **Developer** view → **Topology**. You see `booking` as a circle;
the arrow icon opens the Route, and clicking the circle shows pods, builds and the Route.

---

## Step 6 – Day-2 operations

```bash
oc scale deploy/booking --replicas=3         # or use the up arrow in Topology
oc get pods -w

oc start-build booking --follow              # rebuild from Git…
oc get pods -w                               # …and the Deployment rolls out the new image by itself

oc rollout history deploy/booking
oc rollout undo deploy/booking               # back to the previous version

oc logs -f deploy/booking
```

`oc start-build` → new image in the ImageStream → `oc new-app` added an **image trigger** → new rollout.
That is the OpenShift build-and-deploy loop.

---

## Step 7 (bonus) – Our Helm chart on OpenShift

Helm works on OpenShift unchanged. Two settings differ from EKS:
- **ClusterIP** instead of NodePort (we expose with a Route instead)
- the **fixed image** from the internal registry instead of Docker Hub

```bash
P=$(oc project -q)
helm install booking-helm ../helm/myjavaapp \
  --set service.type=ClusterIP \
  --set image.repository=image-registry.openshift-image-registry.svc:5000/$P/booking \
  --set image.tag=latest
oc apply -f route.yaml                       # run from the openshift/ folder
oc get route booking-helm                    # open https://<HOST>/mahaLogin/
```

The Helm release also shows in the console: **Developer** view → **Helm**.

(PowerShell: replace `$(oc project -q)` with `$P = oc project -q` on its own line, and the `\` line endings with backticks.)

---

## Step 8 – Clean up

```bash
helm uninstall booking-helm
oc delete route booking-helm
oc delete all -l app=booking
oc delete bc,is booking
oc delete is myonlinebooking                 # base image stream created by new-build
oc get all                                   # should be empty
```

The Sandbox also deletes everything when your 30 days end.

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `error: You must be logged in` / `Unauthorized` | Token expired | Copy login command again (step 1) |
| Build fails: `error: build error … no such file Dockerfile` | `openshift/Dockerfile` not on GitHub `main` | Push it, or use `--context-dir` of your fork |
| Build fails pulling `docker.io/...` (`toomanyrequests`) | Docker Hub rate limit | Wait and `oc start-build booking --follow` again |
| Pod `CrashLoopBackOff`, logs say `Permission denied` | Image needs root | Use the `booking` image from step 4, not `booking-hub` |
| Route shows *Application is not available* | No ready pod behind the Service | `oc get pods`, `oc logs deploy/booking` |
| `/mahaLogin/` 404 | War not unpacked (permissions) or wrong path | Step 3 vs step 5; check `oc rsh deploy/booking ls /usr/local/tomcat/webapps` |
| App slow / pods at 0 after hours idle | Sandbox scales idle apps to zero | `oc scale deploy/booking --replicas=1` |
| `oc adm policy add-scc-to-user anyuid …` forbidden | Needs cluster-admin; not allowed in Sandbox | Fix the image instead (step 4) – that is the right way |

---

## Homework

1. In the console, use **+Add → Import from Git** with the same repo and context dir `openshift`, and compare
   what it creates with `oc get all`.
2. Install the **Red Hat OpenShift GitOps** operator on OpenShift Local (needs admin) and point
   `argocd/application-dev.yaml` at it. Which two values must change for OpenShift?
