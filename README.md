```markdown
# Multi-Tenant kagent Setup with Solo Enterprise agentgateway - Code Examples

As a Kubernetes-focused AI Architect, I continue to emphasize that **Kgateway** (the open-source foundation) and its **agentgateway** extension from Solo.io represent the most innovative solution for managing AI traffic in Kubernetes — especially with native support for MCP, A2A, LLM proxying, and AI-aware policies.

Below are detailed, ready-to-use code examples for the three topics we discussed.

## 1. Cross-Namespace ModelConfig Sharing via agentgateway (Centralized LLM Proxy)

**Prerequisites**  
- agentgateway deployed in a central namespace (e.g., `gloo-system`).  
- LLM providers configured centrally in agentgateway (via Helm values or `LLMProvider` CRs).

**Example Helm values snippet for agentgateway LLM proxy**
```yaml
# values.yaml for agentgateway Helm chart
llmProxy:
  enabled: true
  providers:
    - name: openai
      baseURL: https://api.openai.com/v1
      apiKeySecretRef:
        name: openai-key
        key: api-key
```

**Tenant ModelConfig (apply in each tenant namespace)**
```yaml
apiVersion: kagent.dev/v1alpha1
kind: ModelConfig
metadata:
  name: shared-llm-config
  namespace: tenant-a  # Repeat in tenant-b, tenant-c, etc.
spec:
  provider: openai
  baseURL: http://agentgateway.gloo-system.svc.cluster.local:8080/v1  # Internal gateway endpoint
  models:
    - name: gpt-4o
      maxTokens: 4096
  # No apiKey needed – handled centrally by agentgateway
```

**Agent using the shared ModelConfig**
```yaml
apiVersion: kagent.dev/v1alpha1
kind: Agent
metadata:
  name: tenant-agent
  namespace: tenant-a
spec:
  modelConfigName: shared-llm-config
  # ... rest of agent configuration
```

## 2. Agent-to-Agent Integration Routed Through agentgateway (A2A via Gateway)

**Agent B (exposes A2A endpoint)**
```yaml
apiVersion: kagent.dev/v1alpha1
kind: Agent
metadata:
  name: agent-b
  namespace: agent-b-ns
spec:
  a2aConfig:
    enabled: true
    port: 8443
  # ... tools and prompts for Agent B
```

**HTTPRoute in agentgateway to route A2A traffic**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: agent-b-a2a-route
  namespace: gloo-system
spec:
  parentRefs:
    - name: agentgateway
      namespace: gloo-system
  hostnames:
    - agent-b.a2a.internal
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: kagent-controller
          namespace: agent-b-ns
          port: 8443
          kind: Service
      appProtocol: kgateway.dev/a2a  # Critical for native A2A handling
```

**Agent A (calls Agent B via gateway)**
```yaml
apiVersion: kagent.dev/v1alpha1
kind: Agent
metadata:
  name: agent-a
  namespace: agent-a-ns
spec:
  tools:
    - type: remoteAgent
      remoteServer:
        url: http://agentgateway.gloo-system.svc.cluster.local/agent-b.a2a.internal
        protocol: a2a
  # ... rest of Agent A config
```

## 3. Multi-Tenant AuthN/AuthZ with JWT + CEL RBAC

**TrafficPolicy for tenant-scoped tool access**
```yaml
apiVersion: security.policy.gloo.solo.io/v2
kind: TrafficPolicy
metadata:
  name: multi-tenant-rbac
  namespace: gloo-system
spec:
  applyToRoutes:
    - route:
        labels:
          tenant: all  # Or target specific HTTPRoutes
  policy:
    jwt:
      providers:
        - name: keycloak
          issuer: https://keycloak.example.com/realms/my-realm
          audiences:
            - agentgateway
          jwks:
            remote:
              url: https://keycloak.example.com/realms/my-realm/protocol/openid-connect/certs
    celAuthorization:
      parsedExpressions:
        - expression: |
            jwt.claims.tenant == request.headers['x-tenant'] &&
            mcp.tool.name in ['allowed-tool-1', 'allowed-tool-2', 'safe-search']
          message: "Access denied: invalid tenant or unauthorized tool"
```

**Agent with JWT propagation (optional)**
```yaml
apiVersion: kagent.dev/v1alpha1
kind: Agent
metadata:
  name: tenant-agent
  namespace: tenant-a
spec:
  auth:
    jwtSecretName: tenant-a-jwt  # Secret containing bearer token with tenant claim
  # ... other spec
```

These examples give you a complete, production-ready pattern leveraging the unique strengths of **agentgateway** — centralization, observability, and AI-protocol-aware security.

Feel free to copy-paste and adapt. If you hit any apply errors or need testing commands (e.g., curl against the gateway), just let me know your exact versions and I'll refine further!
```

You can save the content above as `kgateway-multi-tenant-examples.md` for easy reference.
