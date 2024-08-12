+++
title = 'Mastering Helm : A Journey to Dynamic Secret Generation'
date = 2024-08-12T22:23:13+02:00
draft = false
tags = ['kubernetes', 'secret', 'helm']
+++

In the world of Kubernetes and Helm, managing secrets securely and efficiently is crucial, especially when dealing with production databases. This article chronicles my journey to implement a dynamic secret generation mechanism for a StatefulSet's database container using Helm.

My objective was to generate a password for the database container of a StatefulSet using Helm. The requirements were to create the secret during the initial installation, to reuse the same secret when upgrading the release and to ensure the secret persists even if the release is deleted

Let's walk through the evolution of my solution, examining each attempt and its flaws before arriving at the final (and optimal ?) approach.

## The first attempt

My initial attempt looked like this:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Values.secrets.myapp.secretName }}
  annotations:
    "helm.sh/resource-policy": "keep"
type: Opaque
data:
  {{- $secretObj := (lookup "v1" "Secret" .Release.Namespace .Values.secrets.myapp.secretName) | default dict }}
  {{- $secretData := (get $secretObj "data") | default dict }}
  {{- $mssqlPasswordKey := .Values.secrets.myapp.mssqlPasswordKey }}
  {{- $dbSecret := (get $secretData $mssqlPasswordKey) | default (randAlphaNum 16) }}
  {{ $mssqlPasswordKey }}: {{ $dbSecret | b64enc | quote }}
  ```

### What went wrong

This implementation had a critical flaw: the `randAlphaNum 16` function was called on every Helm operation, not just during the initial installation. This led to:

1. A new random string being generated and appended to the existing password on each upgrade.
2. The secret becoming longer and wrongly encoded due to repeated base64 encoding.

## The second attempt: conditional logic

I then tried a more sophisticated approach:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Values.secrets.myapp.secretName }}
  annotations:
    "helm.sh/resource-policy": "keep"
type: Opaque
data:
  {{- $secretObj := (lookup "v1" "Secret" .Release.Namespace .Values.secrets.myapp.secretName) | default dict }}
  {{- $secretData := (get $secretObj "data") | default dict }}
  {{- $mssqlPasswordKey := .Values.secrets.myapp.mssqlPasswordKey }}
  {{- if not (hasKey $secretData $mssqlPasswordKey) }}
  {{- $dbSecret := (randAlphaNum 16) }}
  {{ $mssqlPasswordKey }}: {{ $dbSecret | b64enc | quote }}
  {{- else }}
  {{- $dbSecret := get $secretData $mssqlPasswordKey }}
  {{ $mssqlPasswordKey }}: {{ $dbSecret | b64enc | quote }}
  {{- end}}
```

### What went wrong

While this version was closer to my goal, it still had several issues:

1. In the `else` branch, I was retrieving the base64-encoded value from the existing secret.
2. I then applied `b64enc` again, resulting in double encoding.
3. The values in `$secretData` were already strings, so `get $secretData $mssqlPasswordKey` returned a base64-encoded string, not the raw value.

## The third attempt: simplified but flawed

My third attempt aimed for simplicity:

```yaml
{{- $secret := (lookup "v1" "Secret" .Release.Namespace .Values.secrets.myapp.secretName) -}}
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Values.secrets.myapp.secretName }}
  annotations:
    "helm.sh/resource-policy": "keep"
type: Opaque
data:
  {{ $mssqlPasswordKey := .Values.secrets.myapp.mssqlPasswordKey -}}
  {{ if $secret -}}
  {{ $mssqlPasswordKey }}: {{ get $secret $mssqlPasswordKey }}
  {{ else -}}
  {{ $mssqlPasswordKey }}: {{ randAlphaNum 16 | b64enc | quote }}
  {{ end -}}
```
### What went wrong

This approach had a fundamental misunderstanding of how secret data is stored:

1. I tried to access secret data directly from the `$secret` object, which doesn't work as expected.
2. The correct way would have been `get $secret.data $mssqlPasswordKey`, but this would still return a base64-encoded value.
3. I didn't apply `b64enc | quote` to the existing value, potentially causing formatting issues.

## The final solution

After my journey of trial and error, I arrived at a simple but yet effective solution:

```yaml
{{- if .Values.secrets.mpleo.create -}}
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Values.secrets.mpleo.secretName }}
  annotations:
    "helm.sh/resource-policy": "keep"
type: Opaque
data:
  {{ .Values.secrets.mpleo.mssqlPasswordKey }}: {{ randAlphaNum 16 | b64enc | quote }}
{{- end -}}
```
### Why this works

This solution is elegant and effective for several reasons:

1. **Controlled Creation**: The `if .Values.secrets.mpleo.create` condition allows us to control when the secret is created, typically only during the initial installation.
2. **Persistence**: The `helm.sh/resource-policy: keep` annotation ensures that the secret isn't deleted when the release is uninstalled, preserving it for future upgrades.
3. **One-time Password Generation**: `randAlphaNum 16` is only called when the secret is first created, ensuring the password remains consistent across upgrades.
4. **Proper Encoding**: The generated password is correctly base64 encoded and quoted, ready for use as a Kubernetes secret.
5. **Simplicity**: By avoiding complex logic for checking existing secrets, we reduce the chances of errors and make the template easier to maintain.

## Good ol' KISS

Well, this exercise teached me a lesson : I tend to over-engineer things. Not later than today a friend of mine told me about Elon Musk's engineering framework in which he states that one should try to think in a "deletion" mindset when designin features of a product. I think I should embrace that way of thinking in my enineering, trying to delete inessential logic and code when I can.
