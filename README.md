# admission-sidecar

Side car meant to run along side of Styra (or other things like that) which
allow for proxying through it to 'real' k8s webhooks. Reconciles configurations
and makes them available for easier calling from other places.

Exposes webhooks via:
```
http://<address of this webhook>/[admit|mutate]/<name-of-the-k8s-webhook>
```

When injected as a sidecar container, by default the URL is:
```
http://localhost:8088/[admit|mutate]/name-of-the-k8s-webhook
```

# Controlling level of enforcement

By default the proxy requires the namespace of the resource to be labeled with
the following label. If label is not there, proxy will return Allowed
admission response.
```
proxy.chainguard.dev/include=true
```

This behaviour can be changed with environment variable `REQUIRE_LABEL` when
starting proxy. Setting it to "false" will mean that proxy will not require the
label, and resources in all namespaces are handled by the proxy.

# Styra Integration

To patch this into a running OPA system, we add our container into the mix like so. Change this to your own container that you built. TODO(vaikas): Change this to our releases.

```
kubectl patch statefulset opa -n styra-system --type "json" -p '[{"op":"replace","path":"/spec/template/spec/containers/2","value": {"env":[{"name":"SYSTEM_NAMESPACE","valueFrom":{"fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}},{"name":"POD_IP","valueFrom":{"fieldRef":{"apiVersion":"v1","fieldPath":"status.podIP"}}},{"name":"CONFIG_LOGGING_NAME","value":"config-logging"},{"name":"CONFIG_OBSERVABILITY_NAME","value":"config-observability"},{"name":"METRICS_DOMAIN","value":"chainguard.dev/admission-sidecar"}],"image":"gcr.io/vaikas-chainguard/cmd@sha256:b1ac30e0763e5f36c0e40efeb294c1021e6ab08c8e22f1cae3122396eb894e55","imagePullPolicy":"IfNotPresent","livenessProbe":{"failureThreshold":50,"httpGet":{"httpHeaders":[{"name":"k-kubelet-probe","value":"admission-sidecar"}],"path":"/","port":8443,"scheme":"HTTP"},"periodSeconds":1,"successThreshold":1,"timeoutSeconds":1},"name":"controller","ports":[{"containerPort":8443,"name":"http-webhook","protocol":"TCP"}],"readinessProbe":{"failureThreshold":3,"httpGet":{"httpHeaders":[{"name":"k-kubelet-probe","value":"admission-sidecar"}],"path":"/","port":8443,"scheme":"HTTP"},"periodSeconds":1,"successThreshold":1,"timeoutSeconds":1},"resources":{"limits":{"cpu":"1","memory":"1000Mi"},"requests":{"cpu":"50m","memory":"50Mi"}},"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["all"]},"readOnlyRootFilesystem":true,"runAsNonRoot":true},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File"}}]'

```


# Then for the Styra you can add a Validating Rule that looks like this:

```
package policy["com.styra.kubernetes.validating"].rules.rules
enforce[decision] {
  #title: chainguard-proxy
  input.request.kind.kind == "Pod"
  admissionresponse := http.send({"raise_error": true, "url": "http://localhost:8088/admit/<webhook you want to hit>", "body": input, "method": "POST", "force_json_decode": true})
  response := json.unmarshal(admissionresponse.raw_body)
  msg := sprintf("Enforce Response: (%v)", [response.response])
  decision := {
    "allowed": response.response.allowed == true,
    "message": msg
  }
}
```

# Open questions

Because validatingwebhookconfiguration has a name, and each webhook has a name,
we might need a two level naming scheme in the above, something like:

```
http://localhost:8088/admit/<validatingwebhookconfiguration name>/<webhookname>
```

However, empirical evidence suggests that you can't (at least on GKE that I'm
running) have multiple webhooks even though it accepts multiple, it only keeps
one.
