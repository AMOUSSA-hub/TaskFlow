
## 1. Choix techniques

### Choix 1 — Image de base du backend
*   **Choix final** : `node:20-alpine`.
*   **Justification** : L'image Alpine est extrêmement légère (taille finale ~130MB), ce qui accélère les temps de build et de déploiement. Contrairement à l'image complète qui amène l'image a une taille de 1.08 Go.

### Choix 2 — Politique de redémarrage (Docker Compose)
*   **Choix final** : `unless-stopped`.
*   **Justification** : Cette politique garantit que les services redémarrent automatiquement après un crash ou un redémarrage du serveur, tout en respectant l'arrêt manuel effectué par un administrateur.

### Choix 3 — Stratégie de déploiement Kubernetes
*   **Options évaluées** : `maxUnavailable: 0 / maxSurge: 1` vs `maxUnavailable: 1 / maxSurge: 0`.
*   **Choix final** : `maxUnavailable: 0 / maxSurge: 1`.
*   **Justification** : Choix du **Zero Downtime**. Pendant la mise à jour, Kubernetes crée d'abord un nouveau pod avant de supprimer l'ancien, garantissant une disponibilité constante du service, même en production.

### Choix 4 — Nombre de replicas
*   **Options évaluées** : 1 staging / 1 production, 1 staging / 3 production.
*   **Choix final** : **1 en staging / 3 en production**.
*   **Justification** : En staging, un seul pod suffit pour la validation (économie de ressources). En production, 3 réplicas permettent de supporter une charge plus importante et garantissent la haute disponibilité : même si un pod tombe, deux autres continuent de servir le trafic.

## 2. Sécurité et Scan Trivy

Le scan Trivy est intégré dans le pipeline CI/CD et bloque le déploiement en cas de vulnérabilité **CRITICAL**.
*   **Résultats** : Quelques vulnérabilités ont été détectées dans les packages système de l'image Alpine.
*   **Gestion** : Utilisation d'une image multi-stage pour ne pas inclure les outils de build dans l'image finale, ce qui a réduit le nombre de CVE détectées.

## 3. Difficulté rencontrée et résolution

**Problème** : Erreur de connexion `dial tcp [::1]:8080: connect: connection refused` lors de l'exécution de `kubectl apply` dans GitHub Actions.
**Résolution** : Mise en place de **Kind (Kubernetes in Docker)** dans le pipeline pour simuler un cluster Kubernetes réel. Cela a permis de valider le déploiement et les smoke tests sans avoir besoin d'un cluster externe distant, certifiant ainsi la validité des manifests YAML.

## 4. Points avancés implémentés
- **Multi-arch build** : Images compatibles `amd64` (serveurs) et `arm64` (Mac M1/M2).
- **Auto-scaling** : Mise en place de `HorizontalPodAutoscaler` (HPA) en production.
- **Isolations réseau** : NetworkPolicy pour protéger le cache Redis.
- **Budget de disponibilité** : PodDisruptionBudget pour garantir au moins un pod lors des maintenances.
