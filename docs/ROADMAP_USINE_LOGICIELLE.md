# Roadmap usine logicielle

Cette roadmap implemente la chaine suivante pour l application Java web:

1. Planifier: Jira
2. Coder: GitHub
3. Compiler: Maven + JDK 11
4. Tester: JUnit + Maven Surefire
5. Qualite: SonarQube
6. Packaging: Docker
7. Securite: Trivy
8. Orchestration CI/CD: Jenkins
9. Deploiement: Ansible + Kubernetes/K3s
10. Supervision: ELK ou Datadog

## Architecture cible

- VM1: noeud de controle avec Docker, Ansible, Jenkins et Trivy.
- VM2: noeud controleur Kubernetes/K3s.
- VM3: noeud worker Kubernetes/K3s.
- GitHub: depot source et declenchement Jenkins par commit/push.

Le premier deploiement evite un registry Docker externe: Jenkins construit l image sur VM1,
la sauvegarde en archive Docker, puis Ansible importe cette image dans containerd K3s sur VM2 et VM3.

## Etape 1 - Verifier le build local

Depuis la racine du projet:

```bash
cd "/Users/ezzatsaoud/Desktop/SupDeVinci/M1 DEVOPS/Usine Logicielle/java-demo-appweb-main"
JAVA_HOME=$(/usr/libexec/java_home -v 11) mvn clean package
```

Resultat attendu:

- `BUILD SUCCESS`
- `webapp/target/webapp.war`

## Etape 2 - Versionner dans GitHub

```bash
git status
git add .
git commit -m "Ajout roadmap usine logicielle Jenkins Ansible Kubernetes"
git push
```

Si le depot distant n existe pas encore:

```bash
git remote add origin <url-du-repo-github>
git push -u origin main
```

## Etape 3 - Installer les outils sur VM1

Se connecter a VM1:

```bash
vagrant ssh VM1
```

Verifier les outils:

```bash
docker version
ansible --version
mvn -version
java -version
```

Installer Trivy si necessaire:

```bash
sudo apt-get update
sudo apt-get install -y wget gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo gpg --dearmor -o /usr/share/keyrings/trivy.gpg
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy
```

## Etape 4 - Installer Kubernetes/K3s sur VM2 et VM3

Depuis VM1:

```bash
ansible-playbook /vagrant/ansible/playbooks/k3s-deploiement.yaml
```

Verifier:

```bash
ssh vagrant@192.168.56.12 "kubectl get nodes -o wide"
```

## Etape 5 - Tester la chaine sans Jenkins

Depuis la racine du projet sur VM1 ou sur une workspace Jenkins:

```bash
mvn clean package
docker build -t webapp-java:manual .
mkdir -p build
docker save webapp-java:manual -o build/webapp-java-manual.tar
trivy image --severity HIGH,CRITICAL webapp-java:manual
ansible-playbook -i /vagrant/ansible/inventaire ansible/deploy-k8s.yaml \
  -e app_image=webapp-java:manual \
  -e image_archive="$PWD/build/webapp-java-manual.tar"
```

Verifier le deploiement:

```bash
ssh vagrant@192.168.56.12 "kubectl -n devops get pods,svc -o wide"
```

## Etape 6 - Configurer Jenkins

Dans Jenkins:

1. Installer les plugins recommandes: Pipeline, Git, JUnit.
2. Donner l acces Docker a l utilisateur Jenkins.
3. Creer un job `Pipeline`.
4. Choisir `Pipeline script from SCM`.
5. Renseigner l URL GitHub du depot.
6. Indiquer le chemin du Jenkinsfile: `Jenkinsfile`.
7. Configurer un webhook GitHub vers Jenkins si Jenkins est accessible depuis GitHub.

Le `Jenkinsfile` execute les etapes:

- compilation Maven
- tests JUnit/Surefire
- analyse SonarQube si `SONAR_HOST_URL` est defini
- construction Docker
- scan Trivy si Trivy est installe
- deploiement Ansible vers K3s

## Etape 7 - Supervision

Deux options sont prevues:

- ELK pour centraliser les logs applicatifs et systeme.
- Datadog pour les metriques, dashboards et alertes.

Pour le rapport, indiquer que la supervision est branchee apres le deploiement afin de surveiller:

- disponibilite de l application
- logs applicatifs
- consommation CPU/RAM
- etat des pods Kubernetes
- alertes en cas d indisponibilite

## Commandes utiles

Arreter les VM:

```bash
vagrant halt
```

Redemarrer les VM:

```bash
vagrant up
```

Verifier Kubernetes:

```bash
ssh vagrant@192.168.56.12 "kubectl get nodes"
ssh vagrant@192.168.56.12 "kubectl -n devops get all"
```

Supprimer l application Kubernetes:

```bash
ssh vagrant@192.168.56.12 "kubectl delete namespace devops"
```
