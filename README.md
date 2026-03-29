# KermitKernel

It's not easy being secure and efficient. Your AI agent fleet : local or cloud, hardened, orchestrated.

## Project

This project aims at craeting an application for deploying fleets of AI agents levarging kubernetes. We wan't to create sandboxes to control with great precision the scope of each agent, the data they are exposed to and the tasks they need to perform.

Being a kubrenetes app, we are using all kubernetes concept to harden sandboxes in pods, and make this application deployables on-prem or on any cloud and be scalable.

~~~ bash
kermit-kernel/
├── api/                    # Définitions des CRDs (YAML & Go structs)
│   ├── v1alpha1/           # Versions des APIs (Agent, AgentFleet, Gateway)
├── charts/                 # Helm charts pour installer KermitKernel
│   ├── kermit-operator/
│   ├── frogfort-runtime/
│   └── kermit-dash/
├── cmd/                    # Points d'entrée des binaires
│   ├── operator/           # Le Kermit Commander (Go)
│   ├── gateway/            # Kermit Gateway (Envoy Control Plane)
│   └── sidecar/            # Proxy de sécurité (Wasm/Go)
├── config/                 # Configuration Kubernetes (Kustomize)
│   ├── crd/
│   ├── rbac/
│   └── samples/            # Exemples de manifests pour les utilisateurs
├── internal/               # Logique privée (non exportable)
│   ├── controller/         # Logique de réconciliation de l'opérateur
│   ├── graph/              # Interface avec ArcadeDB / FalkorDB
│   ├── session/            # Gestionnaire de sessions (Postgres/JuiceFS)
│   └── inference/          # Adaptateurs vLLM, Ollama, LiteLLM
├── images/                 # Dockerfiles & Recettes pour FrogFort
│   ├── agent-base/         # Image durcie (Talos/Mariner)
│   └── sidecar-envoy/      # Image avec filtres Wasm pré-installés
├── pkg/                    # Bibliothèques réutilisables (SDK pour agents)
└── ui/                     # Kermit Dashboard (Next.js)
~~~

## KermitKernel (kAIs) resources

### Control-plane (Kermit Commander)

Is the heart of the soft. Manages the agent lifecycle via :

- the kermit operator: A k8s operator scrutting the agent type resources. When an agent is reqsuested, it creates the pod and all needed resources (secret, netpol, sidecar, ...).

- the kermit graph: A graph based database for managing agent fleet and communication. It stores metadata of the fleet.

- the session manager: It manages persisting the resources. If an Agent goes down its state, operations and temporary file sneeds to be persisted (PVC or database). It also shares sessions beetweens agents that are part of the same workflow. Exemple if agent A read and writes 50 mo and agent B also, rather tha putting data in the agent sand box, maybe putting it in shared workspace also hardened and isolated ?

- Governenance and audit logs: An imutable registre of every decision taken by an agent. Maybe in the angentic graph with each agent registered (but we keep the killed ones for tracability ?)

### Data Plane (Sandboxes = frogfort)

Agent hardened isolation. Deep defense.

- Runtime class: using kata containers and kata-qemu or kata-clh beside runc. Each agent runs in its own light linux kernel. Maybe work on the images (with pre built ones) for performance and security (less services, or specific services installed like rtk maybe or other).

- Securit

- Sidecars: Each agentic pod contains a proxy container intercepting egress / ingress calls making sure the agent doesn't send data to unauthorized domains and incomming data doesn't contains sensitive information.

### Agentic Mesh (Service Mesh)

Made for:

- agents collaboration, managing communication through A2A and MCP. Enabling collaboration strategys such as parallelisme, looping or sequencial workflows. Garantying secure communications between agents.

- Observability with agent sanity (no hallucinating) and agent economy, storing data on token consumption and helping finding the best prompting patterns for better result with less token.

- Rate limiting : no infinit loop

### Kermit Brains (Inference)

- possibility of local inference / cloud inference or hybrid with different strategies (for exemple local first and fall back if needed or sensitive workflow in local and non sensitive in the cloud or light workflow with light open local models)

- different models and medium for different specialities. The goal is performance and specialization of agents.

### Kermit Gateway

A resource managing connection between agents and models, agents and applications / tools, agents and user.

### Client

A dashboard interface for users to manager their fleet and watch progress.

## Techno stack

### KermitKernel (kAIs) stack

kAIs is orchestrated by kubernetes, any kind of cluster cloud (GKE / AKS / AKE) or on prem (k3s, on prem kubeadm cluster, ...)

### Control-plane (Kermit Commander) stack

- Kermit Operator : Go SDK (le standard pour les performances et l'intégration kube-api).

- Kermit Graph + Governance : ArcadeDB (Multi-modèle : Graph + Document, parfait pour l'audit log immuable) ou FalkorDB (Ultra-rapide, basé sur Redis).

- Session Manager : PostgreSQL avec l'extension pgvector.

- Shared Workspace : MinIO (S3-compatible local) pour le stockage d'objets ou JuiceFS pour un système de fichiers partagé (POSIX) ultra-performant sur K8s.

### Data Plane (Sandboxes = frogfort) stack

- Runtime Class : Kata Containers (Micro-VM).

- Images OS : CBL-Mariner (Microsoft) ou Talos OS (très minimalistes, "read-only" par défaut) pour réduire la surface d'attaque.

- Sidecar Security : Envoy Proxy piloté par WebAssembly (Wasm) pour le filtrage de contenu en temps réel.

- Policy Engine : Kyverno (plus simple à maintenir en YAML natif Kubernetes qu'OPA).

### Agentic Mesh (Service Mesh) stack

- Orchestration Mesh : Istio (Ambient Mesh pour réduire l'overhead des sidecars).

- Agents Collaboration : Dapr (Distributed Application Runtime). Il fournit des APIs de communication (Pub/Sub, State Management) parfaites pour le A2A et supporte le protocole MCP (Model Context Protocol).

- Observabilité / Sanity : Arize Phoenix ou LangSmith (Self-hosted) pour le tracing des traces LLM et la détection d'hallucinations.

### Kermit Brains (Inference) stack

- Local Inference : vLLM (le plus rapide pour le débit) ou TGI (Text Generation Inference).

- Multi-Model Management : Ollama (pour la simplicité de gestion des poids) intégré via l'opérateur.

- Hybrid Orchestration : LiteLLM (en mode Proxy/Gateway) pour unifier les appels (OpenAI, Anthropic, vLLM) sous un seul format.

### Kermit Gateway stack

- Inference Gateway : Gloo Gateway (basé sur Envoy) : gère le failover entre le local et le cloud.

- Tooling Gateway : Temporal.io pour gérer les "long-running tasks" et les retries si un outil externe (API tierce) échoue.

### Client stack

- Frontend : Next.js avec shadcn/ui (Dashboard moderne et rapide).

- Visualisation Graph : React Flow pour afficher le Kermit Graph et le déplacement des agents en temps réel.

## Data flow

Voici le parcours d'une requête de bout en bout :

1. Entrée : Le Client soumet un workflow au Kermit Commander via une API Rest/gRPC.

2. Planification : L'Operator consulte le Kermit Graph pour définir quels agents sont nécessaires. Il crée les pods FrogFort (Kata Containers).

3. Isolation & Data : Le Session Manager monte un volume partagé (JuiceFS) dans les pods pour que les agents puissent lire/écrire des fichiers volumineux de manière sécurisée sans sortir de la sandbox.

4. Exécution (Brains) : L'agent envoie une requête de complétion. La Kermit Gateway intercepte :Si donnée sensible $\rightarrow$ Routage vers vLLM (Local).Si besoin puissance $\rightarrow$ Routage vers Cloud LLM (External).

5. Filtrage (Sidecar) : Le proxy Envoy scanne la réponse du LLM. Si un "leak" de donnée sensible est détecté, il bloque la réponse avant qu'elle n'atteigne l'agent ou le client.

6. Communication Mesh : Les agents collaborent via Dapr/Istio. Chaque échange est logué de façon immuable dans ArcadeDB pour l'audit.

7. Sortie : Le résultat final est renvoyé au Client, et le Session Manager archive le workspace ou le maintient pour une session future.
