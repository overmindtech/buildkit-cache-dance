# The BuildKit Cache Dance

Workaround buildkit/buildx's lack of integrated caching solution

```
---
name: Build
on: push

jobs:
  build:
    name: Build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: YOUR_IMAGE
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Go Build Cache for Docker
        uses: actions/cache@v3
        with:
          path: go-build-cache
          key: ${{ runner.os }}-go-build-cache-${{ hashFiles('**/go.sum') }}

      - name: inject/extract go-build-cache into docker
        uses: overmindtech/buildkit-cache-dance@main
        with:
          cache-source: go-build-cache

      - name: Build and push
        uses: docker/build-push-action@v3
        with:
          context: .
          cache-from: type=gha
          cache-to: type=gha,mode=max
          file: build/package/Dockerfile
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          platforms: linux/amd64,linux/arm64
```

Thanks to [Alexander Pravdin](https://github.com/speller) for the basic idea in [this comment](https://github.com/moby/buildkit/issues/1512).
