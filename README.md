# 🔀 Kubernetes Ingress Rewrite Target Configuration

A **comprehensive, hands-on guide** to mastering NGINX Ingress Controller's `rewrite-target` annotation for intelligent URL path manipulation and backend service routing using real-world examples.

---

## 📋 Table of Contents

* 📌 [Overview](#overview)
* ⚙️ [Prerequisites](#prerequisites)
* 🎯 [The Problem](#the-problem)
* 💡 [Understanding rewrite-target](#understanding-rewrite-target)
* 🔧 [Basic Configuration Examples](#basic-configuration-examples)
* 🚀 [Advanced Regex Patterns](#advanced-regex-patterns)
* 🎬 [Real-World Use Cases](#real-world-use-cases)
* ✅ [Best Practices](#best-practices)
* 📖 [Common Patterns Reference](#common-patterns-reference)
* 🛠️ [Troubleshooting](#troubleshooting)
* 🤝 [Contributing](#contributing)
* 📄 [License](#license)
* 👤 [Author](#author)

---

## 📌 Overview

This repository demonstrates **practical Kubernetes Ingress URL rewriting techniques** with a strong focus on:

* Understanding path-based routing challenges
* Implementing URL rewrite rules
* Using regex capture groups for complex scenarios
* Avoiding common 404 errors in multi-service architectures

All examples use **NGINX Ingress Controller** with **real-world application scenarios**.

---

## ⚙️ Prerequisites

Before getting started, ensure you have the following:

* ✅ Kubernetes cluster (**v1.19+**)
* ✅ NGINX Ingress Controller installed
* ✅ `kubectl` CLI installed and configured
* ✅ Basic understanding of Kubernetes Ingress resources
* ✅ Familiarity with regular expressions (for advanced examples)

---

## 🎯 The Problem

Different ingress controllers have different options that can be used to customize the way they work. NGINX Ingress controller has many options that can be seen in the [official documentation](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/).

### 🏗️ Example Scenario

Consider two microservices in your cluster:

* **Watch App**: Video streaming service available at `http://<watch-service>:<port>/`
* **Wear App**: Apparel e-commerce service available at `http://<wear-service>:<port>/`

### 🎯 Desired Behavior

We want to configure Ingress to achieve the following URL mapping:

| User Visits | Should Forward To |
|------------|-------------------|
| `http://<ingress-service>:<ingress-port>/watch` | `http://<watch-service>:<port>/` |
| `http://<ingress-service>:<ingress-port>/wear` | `http://<wear-service>:<port>/` |

> **Note**: The `/watch` and `/wear` URL paths are configured on the ingress controller to route users to the appropriate backend application. The applications themselves don't have these URL paths configured in their routing logic.

---

### ❌ What Happens Without `rewrite-target`

Without the `rewrite-target` annotation, the ingress controller passes the **full path** to the backend service:

| User Visits | Actually Forwards To | Result |
|------------|---------------------|--------|
| `http://<ingress-service>:<ingress-port>/watch` | `http://<watch-service>:<port>/watch` | ❌ **404 Error** |
| `http://<ingress-service>:<ingress-port>/wear` | `http://<wear-service>:<port>/wear` | ❌ **404 Error** |

**Why does this fail?**

The target applications are built specifically for their purpose and don't expect `/watch` or `/wear` in their URL paths. They're looking for routes like `/`, `/videos`, `/products`, etc., not `/watch/videos` or `/wear/products`.

---

## 💡 Understanding rewrite-target

The `rewrite-target` annotation **rewrites the URL path** before forwarding the request to the backend service. It works like a **search and replace** function:

```
replace(path, rewrite-target)
```

### 🔄 How It Works

**Basic example:**
```
replace("/pay", "/")
```

This removes the `/pay` prefix from the incoming request before forwarding it to the backend.

**Request Flow:**
1. User requests: `http://<ingress>/pay/checkout`
2. Ingress matches path: `/pay`
3. Ingress rewrites to: `/checkout`
4. Backend receives: `http://<service>/checkout`

---

## 🔧 Basic Configuration Examples

### 📝 Example 1: Simple Path Removal

This example strips the `/pay` prefix when forwarding to the backend service:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
  namespace: critical-space
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - http:
        paths:
          - path: /pay
            pathType: Prefix
            backend:
              service:
                name: pay-service
                port:
                  number: 8282
```

**How it works:**

| User Request | Backend Receives |
|--------------|------------------|
| `http://<ingress>/pay` | `http://pay-service:8282/` |
| `http://<ingress>/pay/checkout` | `http://pay-service:8282/checkout` |
| `http://<ingress>/pay/success` | `http://pay-service:8282/success` |

---

### 📝 Example 2: Multi-Service Routing

Configure multiple services with different path prefixes:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-service-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - http:
        paths:
          - path: /watch
            pathType: Prefix
            backend:
              service:
                name: watch-service
                port:
                  number: 8080
          - path: /wear
            pathType: Prefix
            backend:
              service:
                name: wear-service
                port:
                  number: 8080
```

**Request mapping:**

| User Request | Backend Receives |
|--------------|------------------|
| `http://<ingress>/watch/videos` | `http://watch-service:8080/videos` |
| `http://<ingress>/wear/products` | `http://wear-service:8080/products` |

---

## 🚀 Advanced Regex Patterns

### 🎯 Example 3: Regex Capture Groups

This advanced example uses **regex capture groups** for more flexible path rewriting. See the [official example](https://kubernetes.github.io/ingress-nginx/examples/rewrite/) for more details.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - host: rewrite.bar.com
      http:
        paths:
          - path: /something(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: http-svc
                port:
                  number: 80
```

**Understanding the regex pattern:**

```
/something(/|$)(.*)
```

* `/something` - Literal match for this prefix
* `(/|$)` - **First capture group ($1)**: Matches either `/` or end of string
* `(.*)` - **Second capture group ($2)**: Captures everything after

**The rewrite operation:**
```
replace("/something(/|$)(.*)", "/$2")
```

**Request mapping:**

| User Request | Regex Match | $2 Captures | Backend Receives |
|--------------|-------------|-------------|------------------|
| `http://rewrite.bar.com/something` | ✅ | (empty) | `http://http-svc:80/` |
| `http://rewrite.bar.com/something/` | ✅ | (empty) | `http://http-svc:80/` |
| `http://rewrite.bar.com/something/foo` | ✅ | `foo` | `http://http-svc:80/foo` |
| `http://rewrite.bar.com/something/foo/bar` | ✅ | `foo/bar` | `http://http-svc:80/foo/bar` |

---

### 🎯 Example 4: API Versioning

Route API versions to different backends while maintaining clean URLs:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-versioning
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: api-v1-service
                port:
                  number: 8080
          - path: /v2(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: api-v2-service
                port:
                  number: 8080
```

**Request mapping:**

| User Request | Backend Receives |
|--------------|------------------|
| `http://api.example.com/v1/users` | `http://api-v1-service:8080/users` |
| `http://api.example.com/v2/users` | `http://api-v2-service:8080/users` |

---

## 🎬 Real-World Use Cases

### 🏢 Use Case 1: Microservices Architecture

**Scenario:** You have multiple microservices, each serving different functionalities, but you want to expose them under a single domain.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /auth(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: auth-service
                port:
                  number: 3000
          - path: /users(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 3000
          - path: /orders(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 3000
```

---

### 🏢 Use Case 2: Legacy Application Migration

**Scenario:** Migrating from a monolith to microservices while maintaining backward compatibility.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: migration-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - host: legacy.example.com
      http:
        paths:
          - path: /api/v1(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: legacy-api
                port:
                  number: 8080
          - path: /api/v2(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: new-microservice
                port:
                  number: 8080
```

---

## ✅ Best Practices

✔️ **Always test rewrite rules** in a development environment before deploying to production

✔️ **Use specific paths** instead of overly broad regex patterns to avoid unintended matches

✔️ **Document your rewrite rules** in the Ingress annotations or in a separate documentation file

✔️ **Monitor 404 errors** to catch misconfigured rewrite rules early

✔️ **Avoid deep path nesting** — keep path structures simple and maintainable

✔️ **Use capture groups judiciously** — only when simple rewrites don't suffice

✔️ **Test edge cases** — empty paths, trailing slashes, special characters

✔️ **Version your Ingress resources** — keep them in source control with proper versioning

✔️ **Consider using path-based routing** only when necessary — host-based routing is often simpler

---

## 📖 Common Patterns Reference

### Pattern 1: Simple Prefix Removal

```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /
path: /api
```

| Request | Backend |
|---------|---------|
| `/api/users` | `/users` |

---

### Pattern 2: Capture Everything After Prefix

```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /$2
path: /api(/|$)(.*)
```

| Request | Backend |
|---------|---------|
| `/api/v1/users` | `/v1/users` |

---

### Pattern 3: Multiple Capture Groups

```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /api/$1/$2
path: /service/([^/]+)/(.*)
```

| Request | Backend |
|---------|---------|
| `/service/users/list` | `/api/users/list` |

---

### Pattern 4: Conditional Trailing Slash

```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /$2
path: /app(/|$)(.*)
```

| Request | Backend |
|---------|---------|
| `/app` | `/` |
| `/app/` | `/` |
| `/app/home` | `/home` |

---

## 🛠️ Troubleshooting

### 🚨 404 Errors After Applying Rewrite Rules

**Check the actual path being sent to the backend:**

```bash
kubectl logs -n ingress-nginx <ingress-controller-pod> | grep "upstream"
```

**Verify your Ingress configuration:**

```bash
kubectl describe ingress <ingress-name>
```

**Test the rewrite pattern:**

Use a simple curl command to test:

```bash
curl -v http://<ingress-ip>/your-path
```

---

### 🔍 Regex Pattern Not Matching

**Common issues:**

* Missing escape characters in regex
* Incorrect capture group references ($1, $2)
* Path type mismatch (Prefix vs Exact)

**Validate regex patterns** using online tools or testing environments before deployment.

**Debug with verbose logging:**

```bash
kubectl logs -n ingress-nginx <ingress-controller-pod> --tail=100 -f
```

---

### 🔄 Unexpected Path Transformations

**Verify the capture groups:**

```bash
kubectl get ingress <ingress-name> -o yaml
```

**Check for conflicting rules:**

Multiple Ingress resources with overlapping paths can cause unexpected behavior. Review all Ingress resources:

```bash
kubectl get ingress --all-namespaces
```

---

### ⚠️ Backend Service Receiving Wrong Path

**Inspect the rewrite-target annotation:**

```bash
kubectl get ingress <ingress-name> -o jsonpath='{.metadata.annotations}'
```

**Common mistakes:**

* Using `/$1` instead of `/$2` when you have two capture groups
* Forgetting the `(/|$)` pattern for handling trailing slashes
* Using simple `/` when a more complex pattern is needed

---

## 🤝 Contributing

Contributions are welcome!  
Feel free to submit issues or pull requests to improve this guide.

**Ideas for contributions:**
* Additional real-world examples
* More troubleshooting scenarios
* Performance optimization tips
* Security best practices

---

## 📄 License

MIT License — free to use for learning and reference.

---

## 👤 Author

Created as a **practical reference for Kubernetes Ingress rewrite-target configuration** based on **NGINX Ingress Controller** best practices and real-world implementation experience, this articl from **CKAD Course in KodeKloud**.

---

## 🔗 Additional Resources

* [NGINX Ingress Controller Annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/)
* [NGINX Ingress Rewrite Examples](https://kubernetes.github.io/ingress-nginx/examples/rewrite/)
* [Kubernetes Ingress Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)
* [Regular Expression Testing Tool](https://regex101.com/)

---