Pods are ephemeral. They may be destroyed and recreated, or moved from one node to another.

This means their IP addresses are unstable and unreliable.

Services provide a stable abstraction over a set of Pods.  
A Service gets a stable virtual IP (ClusterIP) and a DNS name.

---

### How Services Select Pods

Services use a selector to identify the backend Pods.

This means that a Service does not directly know which Pods it is working with. Instead, it is given a way to identify them. This allows Pods to be easily removed from or added to a Service by simply removing or adding the identifying label that the Service is selecting for.

---

### Types of Services

- ClusterIP
    
- NodePort
    
- LoadBalancer
    
- Ingress