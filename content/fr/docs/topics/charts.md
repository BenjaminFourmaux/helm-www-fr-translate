---
title: "Charts"
description: "Explique le format des charts et fournit des conseils de base pour créer des charts avec Helm."
weight: 1
---

Helm utilise un format de packaging appelé _charts_. Un chart est une collection de fichiers qui décrivent un ensemble de ressources Kubernetes liées. Un seul chart peut être utilisé pour déployer quelque chose de simple, comme un pod memcached, ou quelque chose de complexe, comme une pile d'application web complète avec serveurs HTTP, bases de données, caches, etc.

Les charts sont créés sous forme de fichiers organisés dans un arbre de répertoires particulier. Ils peuvent être emballés dans des archives versionnées pour être déployés.

Si vous souhaitez télécharger et consulter les fichiers d'un chart publié sans l'installer, vous pouvez le faire avec la commande `helm pull chartrepo/chartname`.

Ce document explique le format des charts et fournit des conseils de base pour créer des charts avec Helm.

## La structure des fichiers d'un chart

Un chart est organisé comme une collection de fichiers dans un répertoire. Le nom du répertoire est le nom du chart (sans les informations de version). Ainsi, un chart décrivant WordPress serait stocké dans un répertoire `wordpress/`.

À l'intérieur de ce répertoire, Helm s'attend à une structure qui correspond à ceci :

```text
wordpress/
  Chart.yaml          # Un fichier YAML qui contient les informations sur le charts
  LICENSE             # OPTIONNEL : Un fichier en texte brut contenant la licence du chart
  README.md           # OPTIONNEL : Un fichier README lisible par l'homme
  values.yaml         # Les valeurs de configuration par défaut pour ce chart
  values.schema.json  # OPTIONNEL : Un schéma JSON pour imposer une structure au fichier values.yaml
  charts/             # Un répertoire contenant les charts dont ce chart dépend.
  crds/               # Custom Resource Definitions
  templates/          # Un répertoire de templates qui, combinés avec les valeurs, généreront des fichiers manifest Kubernetes valides.
  templates/NOTES.txt # OPTIONNEL : Un fichier en texte brut contenant de courtes notes d'utilisation
```

Helm réserve l'utilisation des répertoires `charts/`, `crds/` et `templates/`, ainsi que des noms de fichiers listés. Les autres fichiers seront laissés tels quels.

## Le fichier Chart.yaml

Le fichier `Chart.yaml` est requis pour un chart. Il contient les champs suivants :

```yaml
apiVersion: La version API du chart (requis)
name: Le nom du chart (requis)
version: Une version SemVer 2 (requis)
kubeVersion: Une plage de versions Kubernetes compatibles en SemVer (optionnel)
description: Une description en une phrase de ce projet (optionnel)
type: Le type du chart (optionnel)
keywords:
  - Une liste de mots-clés concernant ce projet (optionnel)
home: L'URL de la page d'accueil de ce projet (optionnel)
sources:
  - Une liste d'URLs vers le code source de ce projet (optionnel)
dependencies: # Une liste des dépendances du chart (optionnel)
  - name: Le nom du chart (nginx)
    version: La version du chart ("1.2.3")
    repository: (optionnel) L'URL du dépôt ("https://example.com/charts") ou un alias ("@repo-name")
    condition: (optionnel) Un chemin YAML qui résout en booléen, utilisé pour activer/désactiver des charts (ex. : subchart1.enabled)
    tags: # (optionnel)
      - Les tags peuvent être utilisés pour grouper les charts pour les activer/désactiver ensemble
    import-values: # (optionnel)
      - ImportValues contient le mappage des valeurs source vers la clé parente à importer. Chaque élément peut être une chaîne ou une paire d'éléments enfants/parents.
    alias: (optionnel) Alias à utiliser pour le chart. Utile lorsque vous devez ajouter le même chart plusieurs fois
maintainers: # (optionnel)
  - name: Le nom du mainteneur (requis pour chaque mainteneur)
    email: L'email du mainteneur (optionnel pour chaque mainteneur)
    url: Une URL pour le mainteneur (optionnel pour chaque mainteneur)
icon: Une URL vers une image SVG ou PNG à utiliser comme icône (optionnel)
appVersion: La version de l'application que ce chart contient (optionnel). N'a pas besoin d'être en SemVer. Les guillemets sont recommandés.
deprecated: Indique si ce chart est obsolète (optionnel, booléen)
annotations:
  example: Une liste d'annotations avec des clés par nom (optionnel).
```

Depuis [v3.3.2](https://github.com/helm/helm/releases/tag/v3.3.2), les champs supplémentaires ne sont pas autorisés. L'approche recommandée est d'ajouter des métadonnées personnalisées dans `annotations`.

### Charts et versionnage

Chaque chart doit avoir un numéro de version. Une version doit suivre le standard [SemVer 2](https://semver.org/spec/v2.0.0.html). Contrairement à Helm Classic, Helm v2 et les versions ultérieures utilisent les numéros de version comme marqueurs de release. Les packages dans les dépôts sont identifiés par leur nom ainsi que leur version.

Par exemple, un chart `nginx` dont le champ de version est défini sur `version: 1.2.3` sera nommé :

```text
nginx-1.2.3.tgz
```

Des noms SemVer 2 plus complexes sont également pris en charge, tels que `version: 1.2.3-alpha.1+ef365`. Cependant, les noms non conformes à SemVer sont explicitement interdits par le système.

**REMARQUE :** Alors que Helm Classic et Deployment Manager étaient très orientés GitHub pour les charts, Helm v2 et les versions ultérieures ne dépendent ni de GitHub ni de Git. Par conséquent, il n'utilise pas du tout les SHAs Git pour le versionnage.

Le champ `version` à l'intérieur du fichier `Chart.yaml` est utilisé par de nombreux outils Helm, y compris la CLI. Lors de la génération d'un package, la commande `helm package` utilise la version trouvée dans le `Chart.yaml` comme jeton dans le nom du package. Le système suppose que le numéro de version dans le nom du package du chart correspond au numéro de version dans le `Chart.yaml`. Le non-respect de cette hypothèse entraînera une erreur.

### Le champ `apiVersion`

Le champ `apiVersion` doit être `v2` pour les charts Helm qui nécessitent au moins Helm 3. Les charts compatibles avec les versions précédentes de Helm ont un `apiVersion` défini sur `v1` et sont toujours installables par Helm 3.

Changements de `v1` à `v2` :

- Un champ `dependencies` définissant les dépendances du chart, qui étaient situées dans un fichier `requirements.yaml` séparé pour les charts `v1` (voir [Dépendances des Charts](#chart-dependencies)).
- Le champ `type`, discriminant les charts d'application et les charts de bibliothèque (voir [Types de Charts](#chart-types)).

### Le champ `appVersion`

Notez que le champ `appVersion` n'est pas lié au champ `version`. Il sert à spécifier la version de l'application. Par exemple, le chart `drupal` peut avoir un `appVersion: "8.2.1"`, ce qui indique que la version de Drupal incluse dans le chart (par défaut) est `8.2.1`. Ce champ est informatif et n'a aucun impact sur les calculs de version du chart. Il est fortement recommandé d'encapsuler la version entre guillemets. Cela force le parseur YAML à traiter le numéro de version comme une chaîne. Le fait de laisser la version sans guillemets peut entraîner des problèmes de parsing dans certains cas. Par exemple, YAML interprète `1.0` comme une valeur flottante, et un SHA de commit git comme `1234e10` comme une notation scientifique.

Depuis Helm v3.5.0, `helm create` entoure le champ `appVersion` par défaut de guillemets.

### Le champ `kubeVersion` 

Le champ optionnel `kubeVersion` peut définir des contraintes semver sur les versions Kubernetes prises en charge. Helm validera les contraintes de version lors de l'installation du chart et échouera si le cluster utilise une version de Kubernetes non supportée.

Les contraintes de version peuvent comprendre des comparaisons AND séparées par des espaces, telles que
```
>= 1.13.0 < 1.15.0
```
qui peuvent elles-mêmes être combinées avec l'opérateur OR `||`, comme dans l'exemple suivant
```
>= 1.13.0 < 1.14.0 || >= 1.14.1 < 1.15.0
```
Dans cet exemple, la version `1.14.0` est exclue, ce qui peut être pertinent si un bug dans certaines versions est connu pour empêcher le bon fonctionnement du chart.

Outre les contraintes de version utilisant les opérateurs `=`, `!=`, `>`, `<`, `>=`, `<=`, les notations abrégées suivantes sont prises en charge :

* Plages par tiret pour les intervalles fermés, où `1.1 - 2.3.4` est équivalent à `>= 1.1 <= 2.3.4`.
* Caractères génériques `x`, `X` et `*`, où `1.2.x` est équivalent à `>= 1.2.0 < 1.3.0`.
* Plages avec tilde (changements de version de correctif autorisés), où `~1.2.3` est équivalent à `>= 1.2.3 < 1.3.0`.
* Plages avec caret (changements de version mineure autorisés), où `^1.2.3` est équivalent à `>= 1.2.3 < 2.0.0`.

Pour une explication détaillée des contraintes semver prises en charge, consultez [Masterminds/semver](https://github.com/Masterminds/semver).

### Dépréciation des Charts

Lors de la gestion des charts dans un dépôt de charts, il est parfois nécessaire de déprécier un chart. Le champ optionnel `deprecated` dans `Chart.yaml` peut être utilisé pour marquer un chart comme déprécié. Si la **dernière** version d'un chart dans le dépôt est marquée comme dépréciée, alors le chart dans son ensemble est considéré comme déprécié. Le nom du chart peut être réutilisé ultérieurement en publiant une version plus récente qui n'est pas marquée comme dépréciée. Le flux de travail pour la dépréciation des charts est le suivant :

1. Mettez à jour le `Chart.yaml` du chart pour marquer le chart comme déprécié, en augmentant la version.
2. Publiez la nouvelle version du chart dans le dépôt de charts.
3. Supprimez le chart du dépôt source (par exemple, git).

### Types de Charts

Le champ `type` définit le type de chart. Il y a deux types : `application` et `library`. `application` est le type par défaut et c'est le chart standard qui peut être entièrement utilisé. Le [chart de bibliothèque]({{< ref "/docs/topics/library_charts.md" >}}) fournit des utilitaires ou des fonctions pour le constructeur de charts. Un chart de bibliothèque diffère d'un chart d'application car il n'est pas installable et ne contient généralement aucun objet de ressource.

**Remarque :** Un chart d'application peut être utilisé comme un chart de bibliothèque. Cela est possible en définissant le type sur `library`. Le chart sera alors rendu comme un chart de bibliothèque où toutes les utilitaires et fonctions peuvent être utilisées. Tous les objets de ressource du chart ne seront pas rendus.

## Licence, README et NOTES du Chart

Les charts peuvent également contenir des fichiers qui décrivent l'installation, la configuration, l'utilisation et la licence d'un chart.

Un fichier LICENSE est un fichier texte brut contenant la [licence](https://en.wikipedia.org/wiki/Software_license) du chart. Le chart peut contenir une licence car il peut avoir une logique de programmation dans les templates et ne serait donc pas uniquement une configuration. Il peut également y avoir des licences séparées pour l'application installée par le chart, si nécessaire.

Un README pour un chart devrait être formaté en Markdown (README.md) et devrait généralement contenir :

- Une description de l'application ou du service fourni par le chart
- Les prérequis ou les exigences pour exécuter le chart
- Descriptions des options dans `values.yaml` et des valeurs par défaut
- Toute autre information pertinente pour l'installation ou la configuration du chart

Lorsque les hubs et autres interfaces utilisateur affichent des détails sur un chart, ces détails sont tirés du contenu du fichier `README.md`.

Le chart peut également contenir un fichier `templates/NOTES.txt` en texte brut qui sera imprimé après l'installation et lors de la consultation du statut d'une release. Ce fichier est évalué comme un [template](#templates-and-values) et peut être utilisé pour afficher des notes d'utilisation, des étapes suivantes ou toute autre information pertinente pour une release du chart. Par exemple, des instructions pourraient être fournies pour se connecter à une base de données ou accéder à une interface web. Étant donné que ce fichier est imprimé sur STDOUT lors de l'exécution de `helm install` ou `helm status`, il est recommandé de garder le contenu bref et de renvoyer au README pour plus de détails.

## Chart Dependencies

In Helm, one chart may depend on any number of other charts. These dependencies
can be dynamically linked using the `dependencies` field in `Chart.yaml` or
brought in to the `charts/` directory and managed manually.

### Managing Dependencies with the `dependencies` field

The charts required by the current chart are defined as a list in the
`dependencies` field.

```yaml
dependencies:
  - name: apache
    version: 1.2.3
    repository: https://example.com/charts
  - name: mysql
    version: 3.2.1
    repository: https://another.example.com/charts
```

- The `name` field is the name of the chart you want.
- The `version` field is the version of the chart you want.
- The `repository` field is the full URL to the chart repository. Note that you
  must also use `helm repo add` to add that repo locally.
- You might use the name of the repo instead of URL

```console
$ helm repo add fantastic-charts https://charts.helm.sh/incubator
```

```yaml
dependencies:
  - name: awesomeness
    version: 1.0.0
    repository: "@fantastic-charts"
```

Once you have defined dependencies, you can run `helm dependency update` and it
will use your dependency file to download all the specified charts into your
`charts/` directory for you.

```console
$ helm dep up foochart
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "local" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "example" chart repository
...Successfully got an update from the "another" chart repository
Update Complete. Happy Helming!
Saving 2 charts
Downloading apache from repo https://example.com/charts
Downloading mysql from repo https://another.example.com/charts
```

When `helm dependency update` retrieves charts, it will store them as chart
archives in the `charts/` directory. So for the example above, one would expect
to see the following files in the charts directory:

```text
charts/
  apache-1.2.3.tgz
  mysql-3.2.1.tgz
```

#### Alias field in dependencies

In addition to the other fields above, each requirements entry may contain the
optional field `alias`.

Adding an alias for a dependency chart would put a chart in dependencies using
alias as name of new dependency.

One can use `alias` in cases where they need to access a chart with other
name(s).

```yaml
# parentchart/Chart.yaml

dependencies:
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-1
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-2
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
```

In the above example we will get 3 dependencies in all for `parentchart`:

```text
subchart
new-subchart-1
new-subchart-2
```

The manual way of achieving this is by copy/pasting the same chart in the
`charts/` directory multiple times with different names.

#### Tags and Condition fields in dependencies

In addition to the other fields above, each requirements entry may contain the
optional fields `tags` and `condition`.

All charts are loaded by default. If `tags` or `condition` fields are present,
they will be evaluated and used to control loading for the chart(s) they are
applied to.

Condition - The condition field holds one or more YAML paths (delimited by
commas). If this path exists in the top parent's values and resolves to a
boolean value, the chart will be enabled or disabled based on that boolean
value.  Only the first valid path found in the list is evaluated and if no paths
exist then the condition has no effect.

Tags - The tags field is a YAML list of labels to associate with this chart. In
the top parent's values, all charts with tags can be enabled or disabled by
specifying the tag and a boolean value.

```yaml
# parentchart/Chart.yaml

dependencies:
  - name: subchart1
    repository: http://localhost:10191
    version: 0.1.0
    condition: subchart1.enabled,global.subchart1.enabled
    tags:
      - front-end
      - subchart1
  - name: subchart2
    repository: http://localhost:10191
    version: 0.1.0
    condition: subchart2.enabled,global.subchart2.enabled
    tags:
      - back-end
      - subchart2
```

```yaml
# parentchart/values.yaml

subchart1:
  enabled: true
tags:
  front-end: false
  back-end: true
```

In the above example all charts with the tag `front-end` would be disabled but
since the `subchart1.enabled` path evaluates to 'true' in the parent's values,
the condition will override the `front-end` tag and `subchart1` will be enabled.

Since `subchart2` is tagged with `back-end` and that tag evaluates to `true`,
`subchart2` will be enabled. Also note that although `subchart2` has a condition
specified, there is no corresponding path and value in the parent's values so
that condition has no effect.

##### Using the CLI with Tags and Conditions

The `--set` parameter can be used as usual to alter tag and condition values.

```console
helm install --set tags.front-end=true --set subchart2.enabled=false
```

##### Tags and Condition Resolution

- **Conditions (when set in values) always override tags.** The first condition
  path that exists wins and subsequent ones for that chart are ignored.
- Tags are evaluated as 'if any of the chart's tags are true then enable the
  chart'.
- Tags and conditions values must be set in the top parent's values.
- The `tags:` key in values must be a top level key. Globals and nested `tags:`
  tables are not currently supported.

#### Importing Child Values via dependencies

In some cases it is desirable to allow a child chart's values to propagate to
the parent chart and be shared as common defaults. An additional benefit of
using the `exports` format is that it will enable future tooling to introspect
user-settable values.

The keys containing the values to be imported can be specified in the parent
chart's `dependencies` in the field `import-values` using a YAML list. Each item
in the list is a key which is imported from the child chart's `exports` field.

To import values not contained in the `exports` key, use the
[child-parent](#using-the-child-parent-format) format. Examples of both formats
are described below.

##### Using the exports format

If a child chart's `values.yaml` file contains an `exports` field at the root,
its contents may be imported directly into the parent's values by specifying the
keys to import as in the example below:

```yaml
# parent's Chart.yaml file

dependencies:
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    import-values:
      - data
```

```yaml
# child's values.yaml file

exports:
  data:
    myint: 99
```

Since we are specifying the key `data` in our import list, Helm looks in the
`exports` field of the child chart for `data` key and imports its contents.

The final parent values would contain our exported field:

```yaml
# parent's values

myint: 99
```

Please note the parent key `data` is not contained in the parent's final values.
If you need to specify the parent key, use the 'child-parent' format.

##### Using the child-parent format

To access values that are not contained in the `exports` key of the child
chart's values, you will need to specify the source key of the values to be
imported (`child`) and the destination path in the parent chart's values
(`parent`).

The `import-values` in the example below instructs Helm to take any values found
at `child:` path and copy them to the parent's values at the path specified in
`parent:`

```yaml
# parent's Chart.yaml file

dependencies:
  - name: subchart1
    repository: http://localhost:10191
    version: 0.1.0
    ...
    import-values:
      - child: default.data
        parent: myimports
```

In the above example, values found at `default.data` in the subchart1's values
will be imported to the `myimports` key in the parent chart's values as detailed
below:

```yaml
# parent's values.yaml file

myimports:
  myint: 0
  mybool: false
  mystring: "helm rocks!"
```

```yaml
# subchart1's values.yaml file

default:
  data:
    myint: 999
    mybool: true
```

The parent chart's resulting values would be:

```yaml
# parent's final values

myimports:
  myint: 999
  mybool: true
  mystring: "helm rocks!"
```

The parent's final values now contains the `myint` and `mybool` fields imported
from subchart1.

### Managing Dependencies manually via the `charts/` directory

If more control over dependencies is desired, these dependencies can be
expressed explicitly by copying the dependency charts into the `charts/`
directory.

A dependency should be an unpacked chart directory but its name cannot start 
with `_` or `.`. Such files are ignored by the chart loader.

For example, if the WordPress chart depends on the Apache chart, the Apache
chart (of the correct version) is supplied in the WordPress chart's `charts/`
directory:

```yaml
wordpress:
  Chart.yaml
  # ...
  charts/
    apache/
      Chart.yaml
      # ...
    mysql/
      Chart.yaml
      # ...
```

The example above shows how the WordPress chart expresses its dependency on
Apache and MySQL by including those charts inside of its `charts/` directory.

**TIP:** _To drop a dependency into your `charts/` directory, use the `helm
pull` command_

### Operational aspects of using dependencies

The above sections explain how to specify chart dependencies, but how does this
affect chart installation using `helm install` and `helm upgrade`?

Suppose that a chart named "A" creates the following Kubernetes objects

- namespace "A-Namespace"
- statefulset "A-StatefulSet"
- service "A-Service"

Furthermore, A is dependent on chart B that creates objects

- namespace "B-Namespace"
- replicaset "B-ReplicaSet"
- service "B-Service"

After installation/upgrade of chart A a single Helm release is created/modified.
The release will create/update all of the above Kubernetes objects in the
following order:

- A-Namespace
- B-Namespace
- A-Service
- B-Service
- B-ReplicaSet
- A-StatefulSet

This is because when Helm installs/upgrades charts, the Kubernetes objects from
the charts and all its dependencies are

- aggregated into a single set; then
- sorted by type followed by name; and then
- created/updated in that order.

Hence a single release is created with all the objects for the chart and its
dependencies.

The install order of Kubernetes types is given by the enumeration InstallOrder
in kind_sorter.go (see [the Helm source
file](https://github.com/helm/helm/blob/484d43913f97292648c867b56768775a55e4bba6/pkg/releaseutil/kind_sorter.go)).

## Templates and Values

Helm Chart templates are written in the [Go template
language](https://golang.org/pkg/text/template/), with the addition of 50 or so
add-on template functions [from the Sprig
library](https://github.com/Masterminds/sprig) and a few other [specialized
functions]({{< ref "/docs/howto/charts_tips_and_tricks.md" >}}).

All template files are stored in a chart's `templates/` folder. When Helm
renders the charts, it will pass every file in that directory through the
template engine.

Values for the templates are supplied two ways:

- Chart developers may supply a file called `values.yaml` inside of a chart.
  This file can contain default values.
- Chart users may supply a YAML file that contains values. This can be provided
  on the command line with `helm install`.

When a user supplies custom values, these values will override the values in the
chart's `values.yaml` file.

### Template Files

Template files follow the standard conventions for writing Go templates (see
[the text/template Go package
documentation](https://golang.org/pkg/text/template/) for details). An example
template file might look something like this:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{ .Values.imageRegistry }}/postgres:{{ .Values.dockerTag }}
          imagePullPolicy: {{ .Values.pullPolicy }}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{ default "minio" .Values.storage }}
```

The above example, based loosely on
[https://github.com/deis/charts](https://github.com/deis/charts), is a template
for a Kubernetes replication controller. It can use the following four template
values (usually defined in a `values.yaml` file):

- `imageRegistry`: The source registry for the Docker image.
- `dockerTag`: The tag for the docker image.
- `pullPolicy`: The Kubernetes pull policy.
- `storage`: The storage backend, whose default is set to `"minio"`

All of these values are defined by the template author. Helm does not require or
dictate parameters.

To see many working charts, check out the CNCF [Artifact
Hub](https://artifacthub.io/packages/search?kind=0).

### Predefined Values

Values that are supplied via a `values.yaml` file (or via the `--set` flag) are
accessible from the `.Values` object in a template. But there are other
pre-defined pieces of data you can access in your templates.

The following values are pre-defined, are available to every template, and
cannot be overridden. As with all values, the names are _case sensitive_.

- `Release.Name`: The name of the release (not the chart)
- `Release.Namespace`: The namespace the chart was released to.
- `Release.Service`: The service that conducted the release.
- `Release.IsUpgrade`: This is set to true if the current operation is an
  upgrade or rollback.
- `Release.IsInstall`: This is set to true if the current operation is an
  install.
- `Chart`: The contents of the `Chart.yaml`. Thus, the chart version is
  obtainable as `Chart.Version` and the maintainers are in `Chart.Maintainers`.
- `Files`: A map-like object containing all non-special files in the chart. This
  will not give you access to templates, but will give you access to additional
  files that are present (unless they are excluded using `.helmignore`). Files
  can be accessed using `{{ index .Files "file.name" }}` or using the
  `{{.Files.Get name }}` function. You can also access the contents of the file
  as `[]byte` using `{{ .Files.GetBytes }}`
- `Capabilities`: A map-like object that contains information about the versions
  of Kubernetes (`{{ .Capabilities.KubeVersion }}`) and the supported Kubernetes
  API versions (`{{ .Capabilities.APIVersions.Has "batch/v1" }}`)

**NOTE:** Any unknown `Chart.yaml` fields will be dropped. They will not be
accessible inside of the `Chart` object. Thus, `Chart.yaml` cannot be used to
pass arbitrarily structured data into the template. The values file can be used
for that, though.

### Values files

Considering the template in the previous section, a `values.yaml` file that
supplies the necessary values would look like this:

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "s3"
```

A values file is formatted in YAML. A chart may include a default `values.yaml`
file. The Helm install command allows a user to override values by supplying
additional YAML values:

```console
$ helm install --generate-name --values=myvals.yaml wordpress
```

When values are passed in this way, they will be merged into the default values
file. For example, consider a `myvals.yaml` file that looks like this:

```yaml
storage: "gcs"
```

When this is merged with the `values.yaml` in the chart, the resulting generated
content will be:

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "gcs"
```

Note that only the last field was overridden.

**NOTE:** The default values file included inside of a chart _must_ be named
`values.yaml`. But files specified on the command line can be named anything.

**NOTE:** If the `--set` flag is used on `helm install` or `helm upgrade`, those
values are simply converted to YAML on the client side.

**NOTE:** If any required entries in the values file exist, they can be declared
as required in the chart template by using the ['required' function]({{< ref
"/docs/howto/charts_tips_and_tricks.md" >}})

Any of these values are then accessible inside of templates using the `.Values`
object:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{ .Values.imageRegistry }}/postgres:{{ .Values.dockerTag }}
          imagePullPolicy: {{ .Values.pullPolicy }}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{ default "minio" .Values.storage }}
```

### Scope, Dependencies, and Values

Values files can declare values for the top-level chart, as well as for any of
the charts that are included in that chart's `charts/` directory. Or, to phrase
it differently, a values file can supply values to the chart as well as to any
of its dependencies. For example, the demonstration WordPress chart above has
both `mysql` and `apache` as dependencies. The values file could supply values
to all of these components:

```yaml
title: "My WordPress Site" # Sent to the WordPress template

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

Charts at a higher level have access to all of the variables defined beneath. So
the WordPress chart can access the MySQL password as `.Values.mysql.password`.
But lower level charts cannot access things in parent charts, so MySQL will not
be able to access the `title` property. Nor, for that matter, can it access
`apache.port`.

Values are namespaced, but namespaces are pruned. So for the WordPress chart, it
can access the MySQL password field as `.Values.mysql.password`. But for the
MySQL chart, the scope of the values has been reduced and the namespace prefix
removed, so it will see the password field simply as `.Values.password`.

#### Global Values

As of 2.0.0-Alpha.2, Helm supports special "global" value. Consider this
modified version of the previous example:

```yaml
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

The above adds a `global` section with the value `app: MyWordPress`. This value
is available to _all_ charts as `.Values.global.app`.

For example, the `mysql` templates may access `app` as `{{
.Values.global.app}}`, and so can the `apache` chart. Effectively, the values
file above is regenerated like this:

```yaml
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  global:
    app: MyWordPress
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  global:
    app: MyWordPress
  port: 8080 # Passed to Apache
```

This provides a way of sharing one top-level variable with all subcharts, which
is useful for things like setting `metadata` properties like labels.

If a subchart declares a global variable, that global will be passed _downward_
(to the subchart's subcharts), but not _upward_ to the parent chart. There is no
way for a subchart to influence the values of the parent chart.

Also, global variables of parent charts take precedence over the global
variables from subcharts.

### Schema Files

Sometimes, a chart maintainer might want to define a structure on their values.
This can be done by defining a schema in the `values.schema.json` file. A schema
is represented as a [JSON Schema](https://json-schema.org/). It might look
something like this:

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "properties": {
    "image": {
      "description": "Container Image",
      "properties": {
        "repo": {
          "type": "string"
        },
        "tag": {
          "type": "string"
        }
      },
      "type": "object"
    },
    "name": {
      "description": "Service name",
      "type": "string"
    },
    "port": {
      "description": "Port",
      "minimum": 0,
      "type": "integer"
    },
    "protocol": {
      "type": "string"
    }
  },
  "required": [
    "protocol",
    "port"
  ],
  "title": "Values",
  "type": "object"
}
```

This schema will be applied to the values to validate it. Validation occurs when
any of the following commands are invoked:

- `helm install`
- `helm upgrade`
- `helm lint`
- `helm template`

An example of a `values.yaml` file that meets the requirements of this schema
might look something like this:

```yaml
name: frontend
protocol: https
port: 443
```

Note that the schema is applied to the final `.Values` object, and not just to
the `values.yaml` file. This means that the following `yaml` file is valid,
given that the chart is installed with the appropriate `--set` option shown
below.

```yaml
name: frontend
protocol: https
```

```console
helm install --set port=443
```

Furthermore, the final `.Values` object is checked against *all* subchart
schemas. This means that restrictions on a subchart can't be circumvented by a
parent chart. This also works backwards - if a subchart has a requirement that
is not met in the subchart's `values.yaml` file, the parent chart *must* satisfy
those restrictions in order to be valid.

### References

When it comes to writing templates, values, and schema files, there are several
standard references that will help you out.

- [Go templates](https://godoc.org/text/template)
- [Extra template functions](https://godoc.org/github.com/Masterminds/sprig)
- [The YAML format](https://yaml.org/spec/)
- [JSON Schema](https://json-schema.org/)

## Custom Resource Definitions (CRDs)

Kubernetes provides a mechanism for declaring new types of Kubernetes objects.
Using CustomResourceDefinitions (CRDs), Kubernetes developers can declare custom
resource types.

In Helm 3, CRDs are treated as a special kind of object. They are installed
before the rest of the chart, and are subject to some limitations.

CRD YAML files should be placed in the `crds/` directory inside of a chart.
Multiple CRDs (separated by YAML start and end markers) may be placed in the
same file. Helm will attempt to load _all_ of the files in the CRD directory
into Kubernetes.

CRD files _cannot be templated_. They must be plain YAML documents.

When Helm installs a new chart, it will upload the CRDs, pause until the CRDs
are made available by the API server, and then start the template engine, render
the rest of the chart, and upload it to Kubernetes. Because of this ordering,
CRD information is available in the `.Capabilities` object in Helm templates,
and Helm templates may create new instances of objects that were declared in
CRDs.

For example, if your chart had a CRD for `CronTab` in the `crds/` directory, you
may create instances of the `CronTab` kind in the `templates/` directory:

```text
crontabs/
  Chart.yaml
  crds/
    crontab.yaml
  templates/
    mycrontab.yaml
```

The `crontab.yaml` file must contain the CRD with no template directives:

```yaml
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
```

Then the template `mycrontab.yaml` may create a new `CronTab` (using templates
as usual):

```yaml
apiVersion: stable.example.com
kind: CronTab
metadata:
  name: {{ .Values.name }}
spec:
   # ...
```

Helm will make sure that the `CronTab` kind has been installed and is available
from the Kubernetes API server before it proceeds installing the things in
`templates/`.

### Limitations on CRDs

Unlike most objects in Kubernetes, CRDs are installed globally. For that reason,
Helm takes a very cautious approach in managing CRDs. CRDs are subject to the
following limitations:

- CRDs are never reinstalled. If Helm determines that the CRDs in the `crds/`
  directory are already present (regardless of version), Helm will not attempt
  to install or upgrade.
- CRDs are never installed on upgrade or rollback. Helm will only create CRDs on
  installation operations.
- CRDs are never deleted. Deleting a CRD automatically deletes all of the CRD's
  contents across all namespaces in the cluster. Consequently, Helm will not
  delete CRDs.

Operators who want to upgrade or delete CRDs are encouraged to do this manually
and with great care.

## Using Helm to Manage Charts

The `helm` tool has several commands for working with charts.

It can create a new chart for you:

```console
$ helm create mychart
Created mychart/
```

Once you have edited a chart, `helm` can package it into a chart archive for
you:

```console
$ helm package mychart
Archived mychart-0.1.-.tgz
```

You can also use `helm` to help you find issues with your chart's formatting or
information:

```console
$ helm lint mychart
No issues found
```

## Chart Repositories

A _chart repository_ is an HTTP server that houses one or more packaged charts.
While `helm` can be used to manage local chart directories, when it comes to
sharing charts, the preferred mechanism is a chart repository.

Any HTTP server that can serve YAML files and tar files and can answer GET
requests can be used as a repository server. The Helm team has tested some
servers, including Google Cloud Storage with website mode enabled, and S3 with
website mode enabled.

A repository is characterized primarily by the presence of a special file called
`index.yaml` that has a list of all of the packages supplied by the repository,
together with metadata that allows retrieving and verifying those packages.

On the client side, repositories are managed with the `helm repo` commands.
However, Helm does not provide tools for uploading charts to remote repository
servers. This is because doing so would add substantial requirements to an
implementing server, and thus raise the barrier for setting up a repository.

## Chart Starter Packs

The `helm create` command takes an optional `--starter` option that lets you
specify a "starter chart". Also, the starter option has a short alias `-p`.

Examples of usage:

```console
helm create my-chart --starter starter-name
helm create my-chart -p starter-name
helm create my-chart -p /absolute/path/to/starter-name
```

Starters are just regular charts, but are located in
`$XDG_DATA_HOME/helm/starters`. As a chart developer, you may author charts that
are specifically designed to be used as starters. Such charts should be designed
with the following considerations in mind:

- The `Chart.yaml` will be overwritten by the generator.
- Users will expect to modify such a chart's contents, so documentation should
  indicate how users can do so.
- All occurrences of `<CHARTNAME>` will be replaced with the specified chart name so that starter charts can be used as templates, except for some variable files. For example, if you use custom files in the `vars` directory or certain `README.md` files, `<CHARTNAME>` will NOT override inside them. Additionally, the chart description is not inherited.

Currently the only way to add a chart to `$XDG_DATA_HOME/helm/starters` is to
manually copy it there. In your chart's documentation, you may want to explain
that process.
