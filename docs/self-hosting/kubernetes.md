# Deploy with Kubernetes

Kubernetes may be overkill for a static site, but it can be a great fit if you already standardize on Helm + GitOps.

> [!IMPORTANT]
> **Required Headers for Office File Conversion**
> 
> LibreOffice-based tools (Word, Excel, PowerPoint conversion) require these HTTP headers for `SharedArrayBuffer` support:
> - `Cross-Origin-Opener-Policy: same-origin`
> - `Cross-Origin-Embedder-Policy: require-corp`
> 
> The official PDFusion nginx images include these headers. In Kubernetes, **Ingress/Gateway controllers are also reverse proxies**, so ensure these headers are preserved (or add them at the edge).

## Prereqs

- Kubernetes cluster
- Helm v3
- A PDFusion nginx image (e.g. `ghcr.io/nikhilsundriya/PDFusion:<tag>`) that serves on **port 8080**

## Deploy with Helm

### Install from this repo (local chart)

```bash
kubectl create namespace PDFusion

helm upgrade --install PDFusion /path/to/PDFusion/chart \
  --namespace PDFusion \
  --set image.repository=ghcr.io/nikhilsundriya/PDFusion \
  --set image.tag=latest
```

### Install from GHCR (OCI chart)

If the chart is published to GHCR as an OCI artifact:

```bash
export GHCR_USERNAME="<github-org-or-user>"

helm upgrade --install PDFusion oci://ghcr.io/$GHCR_USERNAME/charts/PDFusion \
  --namespace PDFusion \
  --create-namespace \
  --version 0.1.0 \
  --set image.repository=ghcr.io/nikhilsundriya/PDFusion \
  --set image.tag=latest
```

## Expose it

### Port-forward (quick test)

```bash
kubectl -n PDFusion port-forward deploy/PDFusion 8080:8080
```

### Ingress (optional)

Enable Ingress (example for nginx-ingress):

```yaml
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: pdf.example.com
      paths:
        - path: /
          pathType: Prefix
```

### Gateway API (optional)

This chart supports Gateway API `Gateway` + `HTTPRoute`.

Example (Cloudflare Gateway API operator):

```yaml
gateway:
  enabled: true
  name: PDFusion-tunnel
  namespace: PDFusion
  gatewayClassName: cloudflare

httpRoute:
  enabled: true
  parentRefs:
    - name: PDFusion-tunnel
      namespace: PDFusion
      sectionName: http
  hostnames:
    - pdfs.example.com
```

## Ensuring the SharedArrayBuffer headers still work (Ingress/Gateway)

### What "should" happen

PDFusion’s nginx config sets the required response headers. Most Ingress/Gateway controllers **pass upstream response headers through unchanged**.

### What can break it

- A controller/edge policy that **overrides** or **strips** response headers
- A "security headers" middleware that sets different COOP/COEP values

### How to verify

Run this against your public endpoint:

```bash
curl -I https://pdf.example.com/ | egrep -i 'cross-origin-opener-policy|cross-origin-embedder-policy'
```

You should see:

- `Cross-Origin-Opener-Policy: same-origin`
- `Cross-Origin-Embedder-Policy: require-corp`

### If your Ingress controller does not preserve them

Add the headers at the edge (controller-specific). Example for **nginx-ingress**:

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header Cross-Origin-Opener-Policy "same-origin" always;
      add_header Cross-Origin-Embedder-Policy "require-corp" always;
```

### If you’re using Gateway API and want to force-add headers

Gateway API supports a `ResponseHeaderModifier` filter. You can attach it in `httpRoute.rules[*].filters`:

```yaml
httpRoute:
  enabled: true
  hostnames: [pdf.example.com]
  parentRefs:
    - name: PDFusion-tunnel
      namespace: misc
      sectionName: http
  rules:
    - matches:
        - path: { type: PathPrefix, value: / }
      filters:
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            set:
              - name: Cross-Origin-Opener-Policy
                value: same-origin
              - name: Cross-Origin-Embedder-Policy
                value: require-corp
```

Support for specific filters depends on your Gateway controller; if a filter is ignored, add headers at the edge/controller layer instead.
