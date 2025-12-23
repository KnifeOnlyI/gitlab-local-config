# Gitlab

## Configuration réseau

Permettre d'accéder à gitlab et à son container registry en utilisant `gitlab.local` et `registry.gitlab.local`

```bash
echo '127.0.0.1 gitlab.local' | sudo tee -a /etc/hosts
echo '127.0.0.1 registry.gitlab.local' | sudo tee -a /etc/hosts
```

Modifier le contenu du fichier `/etc/docker/daemon.json` pour accepter le pull d'images depuis un docker registry en http.
Ce sera utile à la fin, pour vérifier que le container registry de l'instance gitlab est fonctionnelle et accessible.

```json
{
  "insecure-registries": [
    "registry.gitlab.local:5000"
  ] 
}
```

Redémarrer le service docker

```bash
sudo systemctl restart docker
```

## Démarrer les containers

À la racine du projet : 

```bash
docker compose up -d && docker logs -f gitlab
```

Ceci démarrera tous les containers : 

- Gitlab (Pour gérer les repository de code sources)
- Gitlab runner (pour gérer l'exécution des jobs de pipeline gitlab)
- Minio (Pour gérer le cache des runners)

## Création bucket minio pour cache des runners gitlab

1. Se rendre à l'adresse suivante : http://localhost:9001/login
2. Se connecter avec le couple login `minioadmin` et le mot de passe `minioadmin`
3. Créer un bucket nommé : `gitlab-runners-cache`

## Configurer gitlab

### Configuration de base + config utilisateur root

1. Récupérer le mot de passe par défaut de l'utilisateur `root` de gitlab : `docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password`
2. Se rendre à l'adress suivante : http://gitlab.local:8888
3. Se connecter avec le login `root` et le mot de passe récupéré à l'étape 1
4. Changer le mot de passe de l'utilisateur `root` : http://gitlab.local:8888/-/user_settings/password/edit
5. Désactiver la création de compte autorisée pour tous (décocher `Sign-up enabled`) : http://gitlab.local:8888/admin/application_settings/general#js-signup-settings
6. Sauvegarder
7. Changer la branche par défaut pour `master` ici : http://gitlab.local:8888/admin/application_settings/repository#js-default-branch-name
8. Sauvegarder

### Enregistrer les runners gitlab

Une fois gitlab démarré

1. Se rendre à l'adress suivante : http://gitlab.local:8888/admin/runners/new
2. Configurer comme souhaité. Par exemple : Tout laisser vide et cocher `Run untagged jobs`
3. Récupérer le token affiché sur la page
4. Répéter l'opération encore 3 fois pour obtenir au total 4 tokens
5. Copier le fichier de config du runner : `cp volumes/gitlab-runner/config/config.toml.bak volumes/gitlab-runner/config/config.toml`
6. Renseigner les token récupérés à l'étape 4 dans le fichier : `volumes/gitlab-runner/config/config.toml`
7. Redémarrer le container `gitlab-runner` : `docker restart gitlab-runner`
8. Vérifier qu'il y a bien 4 runners "Online" ici : http://gitlab.local:8888/admin/runners

### Tester le fonctionnement des runners

En étant toujours connecté avec l'utilisateur root.

1. Créer un projet (sans README.md)
2. Push le contenu du dossier `test-project` sur la branche `master` de ce nouveau projet gitlab (configuration SSH requise en amont)
3. Constater la présence d'une pipeline, vérifier qu'il y a 3 stages : `lint`, `compile` et `build`. Le dernier stage contient 4 jobs, chacun publie une image dans le container registry du projet. Ces jobs doivent s'exécuter en parralèle

### Tester le fonctionnement du cache partagé des runners

1. Analyser les jobs des stages `lint` et `compile`. Constater que `lint` a fait un `npm install` complet, a caché le résultat et que `lint` n'a pas eu besoin de re-télécharger les dépendances node
2. Aller voir la présence du cache dans le minio : http://localhost:9001/browser/gitlab-runners-cache

## Tester la connexion au container registry gitlab

On va maintenant tenter de télécharger une des 4 images qui ont été publiés par la pipeline.

```bash
docker login registry.gitlab.local:8888

# Renseigner le login : root
# Renseigner le mot de passe de l'utilisateur `root`

# Téléchargement de l'image depuis le container registry de l'instance gitlab locale
docker pull registry.gitlab.local/root/<project_name>/react-app:job-1
```

## Conclusion

Si tout s'est bien passé, vous avez installé une instance gitlab locale contenant les éléments suivants : 

- 4 runners gitlab (sur un seul container)
- Une gestion du cache partagé entre ces runners (en utilisant minio)
- Un container registry