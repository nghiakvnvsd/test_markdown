## Deployment Architecture

### Container Architecture

```mermaid
flowchart TB
    subgraph Docker["🐳 Docker Container"]
        JVM[JVM - Java 8]
        App[Spring Boot App<br/>:8080]
        Certs[/usr/src/liberty/trust-store/]
        Templates[/usr/src/liberty/mail-templates/]
    end
   
    subgraph Config["📁 External Config"]
        Props[application-{env}.properties]
        Secrets[Vault Secrets]
    end
   
    subgraph Network["🌐 Network"]
        LB[Load Balancer]
        Ingress[Ingress Controller]
    end
   
    Config --> Docker
    Network --> Docker
```
