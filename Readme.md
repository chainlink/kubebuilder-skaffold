# Kubebuilder Dev
Adding skaffold and docker build kit for speedy kube builder dev

Versions:
- Kustomize 3.1.0
- Skaffold 0.3.5
- Kubebuilder 2.0.0-beta.0
- Docker 19.03 (18+ will do)

Base repo was set up using kubebuilder by running the following commands from kubebuilder book:

```
kubebuilder init --domain my.domain
kubebuilder create api --group webapp --version v1 --kind Guestbook
```

## Kustomize fix
Kubebuilder 2.0-beta-0 and Kustomize 3.1 don't play well together just yet. [See here](https://github.com/kubernetes-sigs/kustomize/issues/1429)

The fix:
```
-patches:
+patchesStrategicMerge:

-bases:
+resources:
```

in
- `config/crd/kustomization.yaml`
- `config/default/kustomization.yaml`

This will allow `make deploy` to run successfully

## Skaffold
Kubebuilder and Skaffold don't play well together either, ([see here](https://github.com/GoogleContainerTools/skaffold/issues/1737)) as skaffold messes with CRDs, but we can work around that.

Setting up skaffold:
1.  Add the `skaffold.yaml` file
2.  Change `image:` in `skaffold.yaml` to point to your favourite docker repository
3. Change `image:` in `config/default/manager_image_patch.yaml` to point to the same repository
4. Run `make deploy` _then_ comment out the following section before running Skaffold
```
resources:
# - ../crd
```
in `config/default/kustomization.yaml` (This is a workaround for skaffold messing with the CRDs.)

To run skaffold, run:
```
skaffold dev
```

## Fast Go Builds with Docker buildkit
The go build cache doesn't play well with docker. With a source change, all future layers are invalidated (including the build cache)
To get around this, we can mount in the build cache using the new 'buildkit' feature in docker.

Add this first line to your `Dockerfile`:
```
# syntax = docker/dockerfile:experimental
```
Change the go build line in your `Dockerfile` to the following:
```
RUN --mount=type=cache,target=/go/pkg/mod --mount=type=cache,target=/root/.cache/go-build CGO_ENABLED=0 GOOS=linux GOARCH=amd64 GO111MODULE=on go build -o manager main.go
```
Here, we're mounting in the go module cache _and_ the go build cache, we're also removing the `-a` flag from the build, so it doesn't force rebuilds

You also can comment out go mod download (go build will handle this for you)
```
# RUN go mod download
```

To use in skaffold, make sure this is set in your `skaffold.yaml`:
```
build:
  local:
    useBuildkit: true
```

This speeds up local go builds in docker from 40s to around 5s.

Lots more info [here](https://github.com/golang/go/issues/27719) I tried every method in this thread, but only buildkit would do what I wanted.


## Further reading / Inspiration:
- https://rubygems.registry.github.com/Sanhajio/hot-kubebuilder
