# Aufgaben

## Fluxcd Bootstrap

* Lege ein eigenes GitRepo in deinem Github Account an. 
* Kopiere alles ab hier in dieses Repo rein. Es soll also der Ordner fluxcd in diesem Repo enthalten sein:
    ```
    cp $(pwd)/fluxcd $PATH_TO_REPO/
    ```
* Die Verzeichnisstruktur richtig sich nach dieser Anleitung: https://fluxcd.io/flux/guides/repository-structure/. 
* Die eigentliche Flux Installation wird über das Verzeichnis fluxcd/clusters/local gesteuert. local steht dabei für die locale Umgebung. Später können mehr folgen. 
* Die Datei azure-key-vault-secret.enc.yaml ist von mir vorbefüllt worden. Hier findet sich ein Service Principal mit dem FluxCD später auf den KeyVault zugreifen kann. 
* Niemals, wirklich !niemals! editiere die Datei flux-components.yaml. Diese wird nämlich generiert - dabei gehen Änderungen verloren
* Öffne eine weiteres Terminal und lege einen ssh-key außerhalb des Git Repositories an. Dieser darf nicht in Git eingecheckt werden.

```
ssh-keygen -C fluxcd_key -f ./identity
ssh-keyscan github.com > ./known_hosts
```
* Damit man nicht diesem Key auf das Git Repo zugreifen kann, soll der Public Key (identity.pub) als Deployment Key hinzugefügt werden. https://docs.github.com/en/developers/overview/managing-deploy-keys#deploy-keys
* Erstelle daraus ein generic k8s secret mit den drei Keys identity, identity.pub und known_hosts mit dem Namen git-pull-secret. Tips: 
    * `kubectl create secret generic -h`
    * Benutzt bei euerem Befehl noch folgende Optionen: `--dry-run=client -o yaml`  damit wird das Yaml ausgegeben. 
Bei dieser Aufgabe kommt in etwas das hier raus:
```
apiVersion: v1
data:
    identity: ......
    identity.pub: ....
    known_hosts: ....
kind: Secret
metadata:
    name: git-pull-secret
```

Wenn ihr die Secret Datei habt, legt diese in das GitRepo unter fluxcd/clusters/local und ersetzt damit git-pull-secret.enc.yaml. Verschlüsselt es mit sops!

* Nun muss die Datei flux-sync.yaml noch angepasst werden. Die Git Url muss für euer Git Repository stimmen.
  git commit und push nicht vergessen. 

* Jetzt könnt ihr `./install.sh` ausführen.

  Ihr solltet sowas sehen:
  ```
  kubectl get pods -n flux-system
  NAME                                       READY   STATUS    RESTARTS   AGE
  helm-controller-56fc8dd99d-f4f78           1/1     Running   0          3h44m
  kustomize-controller-b98f664d9-6rvsq       1/1     Running   0          3h44m
  source-controller-7f66565fb8-r95n5         1/1     Running   0          3h44m
  notification-controller-644f548fb6-bhcvl   1/1     Running   0          3h44m
  ```

  Jetzt prüfen wir noch den Status von GitRepository:

  ```
  kubectl get GitRepository -n flux-system
  NAME          URL                                                           AGE     READY   STATUS
  flux-system   ssh://git@github.com/mschreibjambit/k8s-training-fluxcd.git   3h47m   True    stored artifact for revision 'main/a6c456a6e46d2e6675feaa082922dd07b9a5eee6'
  ```
  Wie ihr seht, hat der Source Controller die Git Revision a6c456a6e46d2e6675feaa082922dd07b9a5eee6 (GitHash) im Main Branch gefunden. Wollt ihr den lokalen Stand sehen,
  könnt ihr diesen mit `git rev-parse main` oder `git rev-parse HEAD` prüfen.

  Die Kustomization muss noch geprüft werden:

  ```
  kubectl get Kustomization -n flux-system
  NAME             AGE     READY   STATUS
  flux-system      3h51m   True    Applied revision: main/a6c456a6e46d2e6675feaa082922dd07b9a5eee6
  ```

  Wenn das so aussieht, kann FluxCD sich selbst korrekt installieren und damit funktioniert alles. 

  Tipp: Wenn ihr mich mal seht, wie ich `kubectl get ks -A` oder `kubectl get gitrepo -A` tipp - ich habe die Shortnames benutzt. Probiert mal `kubectl api-resources` aus.

* Wie führt man jetzt ein update von fluxcd aus?

  * Schaut mal welche Version gerade installiert ist. 
  * Runterladen und installieren der aktuellsten flux cli Version (https://github.com/fluxcd/flux2/releases)
  * wechselt in fluxcd/clusters/local und führt folgendes aus:

  ```
  flux install --export > flux-components.yaml
  ```

  * git add & commit und push - schon wird flux sich selbst updaten.

## Aufgaben Infrastructure

* Nun soll die Nginx Ingress Controller Installation mit FluxCD automatisiert werden. Nehmt hierzu das Skript aus `Kubernetes_Basics/08_ingress/install-ingress.sh` als Vorlage. 
  Wie das nun geht, findet ihr hier https://fluxcd.io/flux/use-cases/helm/#getting-started. An die Create Befehle solltet ihr unbedingt ein `--export` ran hängen.

  Ihr findet auch hier Infos: https://fluxcd.io/flux/components/source/helmrepositories/ und https://fluxcd.io/flux/components/helm/helmreleases/

* Jetzt suchen wir noch für das nginx ingress Helm Chart die aktuelle Version heraus und pinnt diese. Wo würdet ihr das Pinning hinschreiben: base/nginx-ingress oder local/nginx-ingress? Warum?

* im scripte Ordner gibt es das Script `validate.sh`. Ihr solltet also im infrastructure Ordner mal `validate.sh . && echo OK` ausführen. Wenn zum Schluß OK steht, war die Validierung in Ordnung.

* Teste deine Installation. 

  Das HelmRelease ist eine Custom Resource von FluxCD und zeigt an, dass es erfolgreich war, oder einen Fehler gibt. 

  ```
  kubectl get helmrelease -n nginx-ingress
  NAME            AGE     READY   STATUS
  ingress-nginx   6h36m   True    Release reconciliation succeeded
  ```
  
  Folgendes zeigt die Liste der Helm Releases im Namespace nginx-ingress an. Jedoch handelt es sich nicht um die Custom Resource von FluxCD, sondern einer Funktion von helm.
  ```
  helm list -n nginx-ingress
  NAME         	NAMESPACE    	REVISION	UPDATED                                	STATUS  	CHART              	APP VERSION
  ingress-nginx	nginx-ingress	1       	2023-02-06 13:34:20.835568504 +0000 UTC	deployed ingress-nginx-4.4.2	1.5.1
  ```

