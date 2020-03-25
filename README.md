# Istio et OpenCensus 101 - Démo Rapide

Voici le code utilisé dans le cadre des présentation Istio 101 et OpenCensus par Sandeep Dinesh.  Jetez-y un coup d'oeil!  J'assume une certaine connaissance de Kubernetes mais ce n'est pas totalement nécessaire.

Lien vidéo:
[![Talk YouTube Link](https://i.ytimg.com/vi/8OjOGJKM98o/maxresdefault.jpg)](https://www.youtube.com/watch?v=8OjOGJKM98o)


Prêt pour démarrer en un seul click? Ouvrez ce repo dans Cloud Shell.  C'est gratuit et tout est inclus pour démarrer.

[![Ouvrir dans Cloud Shell](http://gstatic.com/cloudssh/images/open-btn.svg)](https://console.cloud.google.com/cloudshell/editor?cloudshell_git_repo=https%3A%2F%2Fgithub.com%2Fmatbilodeau%2FIstio101&cloudshell_tutorial=README.md)

# TL;DR - Passer la mise en place

Roulez ceci:

`make create-cluster deploy-istio build push deploy-stuff`

Roulez ceci dans une autre fenêtre de terminal:

`make start-monitoring-services`

## Mise en place du Cluster
Vous aurez besoin d'un _cluster_ Kubernetes 1.14.10 ou plus récent.

Vous aurez aussi besoin de Docker et kubectl 1.9.x ou plus récent installés sur votre machine, ainsi que Google Cloud SDK. Vous pouvez installer Google Cloud SDK (et kubectl) [ici](https://cloud.google.com/sdk).

Pour créer le _cluster_ avec Google Kubernetes Engine, roulez cette commande:

`make create-cluster`

Ceci va créer un _cluster_ nommé "my-istio-cluster" avec 4 _nodes_ dans la zone us-east1-b.  Celui-ci sera déployé dans le projet courrant sélectionné dans votre Google Cloud SDK. Vous pouvez changer ceci en paramétrant des valeurs personnalisées pour l'ID de projet et/ou Zone.

`make create-cluster PROJECT_ID=votre-project-id-ici ZONE=votre-zone-ici`

## Mise en place Istio 
Ce projet assume que vous roulez sur Linux x64. Si vous êtes sur une autre plateforme, il est fortement recommandé d'utiliser [Google Cloud Shell](https://cloud.google.com/shell) pour avoir un environnement Linux x64 gratuitement.

Pour déployer Istio dans le _cluster_, roulez

`make deploy-istio`

Ceci va déployer les services Istio et le _control plane_ dans votre _cluster_ Kubernetes. Istio va créer son propre _namespace_ Kubernetes avec ses Services et _Deployments_. De plus, cette commande installe les services assistants. Jaeger pour les traces, Prometheus pour la surveillance, Servicegraph pour visualiser vos microservices, et Grafana pour visualiser les métriques.

## Démarrer les services assistants

Roulez cette commande dans une autre fenêtre de terminal:

`make start-monitoring-services`

Ceci va créer des tunnels vers votre _cluster_ Kubernetes pour [Jaeger](http://localhost:16686), [Servicegraph](http://localhost:8088), et [Grafana](http://localhost:3001). Cette commande ne termine pas puisqu'elle garde une connection ouverte.
Si vous faites ce demo à partir de Cloud Shell, vous devrez utiliser le _Web Preview_ à partir de cette fenêtre de terminal, des instructions sont fournies plus loin.

## Créer les conteneurs Docker

Pour construire et pousser les conteneurs, roulez:

`make build push`

Ceci va créer les conteneurs Docker et les publier vers votre Google Container Registry. 

Encore une fois, vous pouvez utiliser un Project ID personnalisé en vous assurant qu'il soit identique au précédent:

`make build push PROJECT_ID=votre-project-id-ici`

*Note:* ceci va construire trois conteneurs, un pour la démo "vanille" Istio et deux pour celle d'OpenCensus.

## Déployer les Services Kubernetes

Ceci va créer trois _Deployments_ et les trois Services.

`make deploy-stuff`

Ici aussi, vous pouvez utiliser un Project ID personnalisé en vous assurant qu'il soit identique au précédent:

`make deploy-stuff PROJECT_ID=votre-project-id-ici`

# Le Code

La commande ci-dessus a déployé trois microservices roulant tous le même code, avec une configuration différente pour chacun.  Le [code](./code/code-only-istio/index.js) est super simple, il ne fait qu'envoyer une requête à un service en aval(_downstream_), prend le résultat et le concatène avec son propre nom, des informations sur la latence et l'URL en aval.

C'est une superbe application pour la démo Istio car vous pouvez en enchaîner un nombre "infini" pour créer des arbres de services en profondeur simulant de vrais déploiements de microservices.

# Utilisation Istio

Allons voir les ressources Kubernetes:

`make get-stuff`

Vous devriez voir quelque chose comme ça:
```
kubectl get pods && kubectl get svc && kubectl get ingress
NAME                                 READY     STATUS    RESTARTS   AGE
backend-prod-1666293437-dcrnp        2/2       Running   0          21m
frontend-prod-3237543857-g8fpp       2/2       Running   0          22m
middleware-canary-2932750245-cj8l6   2/2       Running   0          21m
middleware-prod-1206955183-4rbpt     2/2       Running   0          21m
NAME         CLUSTER-IP    EXTERNAL-IP       PORT(S)        AGE
backend      10.3.252.16   <none>            80/TCP         22m
frontend     10.3.248.79   <none>            80/TCP         22m
kubernetes   10.3.240.1    <none>            443/TCP        23m
middleware   10.3.251.46   <none>            80/TCP         22m
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                                                                                                     AGE
istio-ingressgateway   LoadBalancer   10.3.246.226   35.XXX.XXX.XXX  80:31380/TCP,443:31390/TCP,31400:31400/TCP,15011:31754/TCP,8060:32056/TCP,15030:31770/TCP,15031:32112/TCP   2m
```

## Où est Istio?

à part du "istio-ingressgateway", vous devriez remarquer qu'il n'y a aucune trace d'Istio ici. C'est parce que le _control plane_ d'Istio est démarré dans son propre _namespace_ Kubernetes. Vous pouvez voir les ressources Istio avec cette commande:

`kubectl get pods --namespace=istio-system`

Nous avons démarré Istio en mode "auto inject". Après que le Makefile ait déployé Istio, le Makefile a roulé cette commande:

`kubectl label namespace default istio-injection=enabled`

Ceci signifie que tous les _Pods_ créés par Kubernetes dans le _namespace_ par défaut vont automatiquement se retrouver avec un sidecar proxy Istio rattaché. Ce proxy va mettre en application les politiques Istio sans requérir aucune autre action de la part de l'application! 
Vous pouvez aussi rouler Istio en mode normal et ajouter le proxy manuellement dans les fichiers YAML de Kubernetes.  Ici aussi, aucun changement n'est apporté à l'applicaton mais le _Deployment_ Kubernetes est _patché_  en mode manuel. C'est utile si vous voulez que certains services contournent (_bypass_) Istio.

## Essayer le tout

Vous avez surement remarqué qu'aucun de nos services n'ont d'adresse IP externe.  C'est parce que nous voulons que tout le trafic entrant soit géré par Istio. Pour ce faire, Istio utilise un [Ingress Gateway](https://istio.io/docs/tasks/traffic-management/ingress/#configuring-ingress-using-an-istio-gateway).

Naviguez vers l'adresse IP externe du "istio-ingressgateway". À ce point, vous devriez obtenir une erreur!

**Pas de panique! Tout va bien!**

Pour l'instant, le _gateway_ n'est pas configuré donc il rejette tout le trafic à la périphérie (_edge_). Configurons le avec cette commande:

`kubectl create -f ./configs/istio/ingress.yaml`

Ce fichier contient deux objets. Le premier est un _gateway_, nous permettant de se rattacher au "istio-ingressgateway" existant sur le _cluster_. Le second objet est un VirtualService, nous permettant d'appliquer des règles de routage. Comme nous utilisons un métacaractère (_wildcard_), soit l'astérisque (*), pour l'hôte avec une seule règle de route, tout le trafic de ce _gateway_ ira au service frontend.

Naviguez vers l'adresse IP et ça devrait maintenant fonctionner!

```
frontend-prod - 0.287secs
http://middleware/ -> middleware-canary - 0.241secs
http://backend/ -> backend-prod - 0.174secs
http://time.jsontest.com/ -> StatusCodeError: 404 - ""
```
Vous voyez ici que le service frontend demande le service middleware, qui à son tour demande le service backend, qui envoie une requête à time.jsontest.com

## Corriger le 404

Vous avez remarqué que time.jsontest.com renvoie un 404. C'est parce que par défaut, Istio bloque tout le trafic sortant (_Egress_) du _cluster_. C'est une bonne pratique en sécurité car ceci empêche que du code malicieux ne se rapporte (_call home_) ou que votre code ne communique avec des services tiers non vérifiés.

Autorisons time.jsontest.com en créant un [ServiceEntry](https://istio.io/docs/tasks/traffic-management/egress/).

Vous pouvez voir le ServiceEntry que nous venons tout juste de créer [ici](./configs/istio/egress.yaml), et remarquez qu'il autorise à la fois l'accès HTTP et HTTPS vers time.jsontest.com

Pour appliquer cette règle, roulez:

`kubectl apply -f ./configs/istio/egress.yaml`

Tous nos services sont maintenant fonctionnels!

```
frontend-prod - 0.172secs
http://middleware/ -> middleware-canary - 0.154secs
http://backend/ -> backend-prod - 0.142secs
http://time.jsontest.com/ -> {
   "time": "12:43:09 AM",
   "milliseconds_since_epoch": 1515026589163,
   "date": "01-04-2018"
}
```

## Corriger le Flip Flop

En rafraîchissant la pages quelques fois, vous remarquerez un petit changement d'un chargement à l'autre. Parfois, le service middleware est nommé `middleware-canary` et d'autres fois c'est `middleware-prod`

C'est parce qu'il n'y a qu'un seul service Kubernetes nommé middleware qui envoie le trafic vers deux _deployments_ (nommés middleware-prod et middleware-canary). C'est une fonctionnalité puissante permettant de faire des choses comme des déploiements Blue-Green et du _Canary testing_. Avec Servicegraph, on peut en avoir une représentation visuelle.

Ouvrir Servicegraph: http://localhost:8088/dotviz
Pour les utilisateurs Cloud Shell, démarrez un _Web Preview_ à partir du terminal ayant servi à rouler `make start-monitoring-services` puis modifiez l'URL comme suit :
https://8088-dot-1111111-dot-devshell.appspot.com/dotviz   (changez le port à 8088 et ajoutez le suffixe /dotviz)

Vous devriez voir quelque chose comme ça:
![Servicegraph](./servicegraph.png)

Vous devriez voir que le trafic part du frontend (prod) et peut ensuite aller vers l'un ou l'autre des middleware, (prod) ou (canary), pour finalement aller vers backend (prod)

Avec Istio, vous pouvez contrôler où le trafic est dirigé en utilisant un [VirtualService](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#VirtualService). Par exemple, vous pourriez envoyer 80% de tout le trafic destiné au service middleware vers prod et le 20% restant vers canary. Dans notre cas, nous enverrons 100% du trafic vers prod.

Vous pouvez voir le VirtualService [ici](./configs/istio/routing-1.yaml).

Quelques points intéressants dans ce fichier.

Vous aurez probablement remarqué que le VirtualService frontend est différent des autres. C'est parce que nous relions le service frontend au _gateway_.

Ensuite, nous avons la partie où nous dirigeons tout le trafic vers le _deployment_ "prod". La section clé pour le VirtualService middleware est ici:
```yaml
  http:
  - route:
    - destination:
        host: middleware
        subset: prod
```

Vous vous demandez probablement où le "subset" est défini? Comment fait Istio pour savoir ce qu'est "prod"?

Vous pouvez définir ces _subsets_ dans ce qu'on appelle une [DestinationRule](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#DestinationRule).

Nos DestinationRules sont définies [ici](./configs/istio/destinationrules.yaml). Allons voir la partie importante de la règle du middleware:

```yaml
spec:
  host: middleware
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - name: prod
    labels:
      version: prod
  - name: canary
    labels:
      version: canary
```

Premièrement, nous établissons un trafficPolicy qui s'assure que tout le trafic est chiffré par TLS Mutuel (_Mutual TLS_). Nous avons donc des communications chiffrées et pouvons restreindre quels services ont accès à quels autres services à l'intérieur du _mesh_. Même si un intrus avait accès à notre _cluster_, il ne pourrait pas imiter un service légitime si nous utilisons mTLS, ce qui limite le dommage qu'il pourrait faire.

Ensuite, vous pouvez voir comment les _subsets_ sont définis. Nous nommons chaque _subset_ et ensuite utilisons des étiquettes (_labels_) Kubernetes pour selectionner les _pods_ dans chaque _subset_. Dans notre cas, nous créons un _subset_ prod et un _subset_ canary.

Créons les DestinationRules et les VirtualServices:

`kubectl apply -f ./configs/istio/destinationrules.yaml`

`kubectl apply -f ./configs/istio/routing-1.yaml`

Maintenant tout le trafic est dirigé vers le service middleware-prod.

## Créer de l'instabilité

Dans le vrai monde, les services plantent (_fail_) tout le temps. Dans un monde de microservices, ça veut dire que vous pouvez avoir des milliers de cas de plantage en aval et votre application doit être concue pour gérer chacun d'entre-eux. Le Service Mesh d'Istio peut être configuré pour gérer automatiquement plusieurs de ces cas pour que votre application n'ait pas à s'en soucier!

_Note: Istio a une fonctionnalité appelée [Fault Injection](https://istio.io/docs/tasks/traffic-management/fault-injection/) qui peut simuler des erreurs. Vous n'avez donc pas à écrire de mauvais code pour tester si votre application peut gérer les cas de plantage._

Bien que le code déployé ne soit très stable, une fonction cachée peut faire retourner un code 500 de manière aléatoire!

Ce code est activé lors de la réception d'un en-tête HTTP (_header_) nommée "fail" ayant une valeur entre 0 et 1, 0 étant 0% de probabilité de plantage et 1 100% de probabilité.

Nous utiliserons [Postman](https://www.getpostman.com/) pour envoyer les en-têtes et voir le résultat.

![Default Route Rules](./no_retry.gif)

Avec 30% de plantage, vous pouvez voir que l'application plante beaucoup mais que ça fonctionne parfois. Si on choisi 50%, l'application plante presque toujours! C'est parce que chaque service transfère les en-têtes aux services en aval et l'erreur se reproduit en cascade!

## Corriger avec Istio

 Comme la requête peut fonctionner ou non, on appelle ca une erreur _flaky_. Ce type d'erreur est difficile à déboguer car elles sont difficiles à identifier! Si le pourcentage est bas, l'errerur ne se produira peut-être qu'un fois parmi un million. Par contre, il peut s'agir d'une requête importante qui pourrait potentiellement tout briser. Le correctif rapide pour ce type d'erreur _flaky_ est de simplement retenter la requête! Ceci fonctionne jusqu'à un certain point mais peut facilement contourner les erreurs de ce type ayant une faible probabilité.

 Normalement, vous auriez besoin d'écrire cette logique de rétentative dans chacune de vos microservices individuels pour chaque requête réseau individuelle. Heureusement, Istio nous la fourni directement et vous n'avez donc pas besoin de modifier votre code du tout!

 Modifions le VirtualServices pour ajouter un peu de logique de retentative. Vous trouverez la mise-à-jour de la règle [Ici](./configs/istio/routing-2.yaml). Une section a été ajoutée pour chacun des services qui ressemble à ceci:

 ```yaml
    retries:
      attempts: 3
      perTryTimeout: 2s
```

Istio va donc retenter la requête trois fois avant d'abandonner et va attendre 2 secondes entre chaque tentative (au cas ou le service en aval soit gelé). Votre application ne le verra que comme une seulre requête, toute la complexité de ces retentatives est donc abstraite.

Appliquez la règle:

`kubectl apply -f ./configs/istio/routing-2.yaml`

Et raffraîchissez large page:

![No more 500 errors with Retry](./working.gif)

Maintenant nous n'avons plus les erreurs!

## Qu'est-ce qui se passe avec le middleware Canary?

Si vous vous souvenez, nous avons deux services middleware. Prod et Canary. Pour l'instant, les règles envoient tout le trafic vers Prod. C'est bon parce qu'on ne veut pas que les utilisateurs normaux aient accès au _build_ Canary.

Toutefois, nous avonsbesoin que notre équipe de développeurs et testeurs de confiance puisse y accéder. Heureusement, Istio nous facilite la tâche ici aussi!

Les règles de routage peuvent contenir des conditions basées sur des éléments comme les en-têtes, témoins (_cookies_), etc. Nous pourrions vérifier si un utilisateur est membre d'un groupe de confiance et lui donner un témoin qui l'autorise l'accès au service canary. Pour simplifier, nous utiliserons seulement un en-tête nommé "x-dev-user" et vérifierons si la valeur correspond à "super-secret".

Vous pouvez voir la nouvelle règle dans [ce fichier](./configs/istio/routing-3.yaml)

```yaml
  http:
  - match:
    - headers:
        x-dev-user:
          exact: super-secret
    route:
      - destination:
          host: middleware
          subset: canary
    retries:
      attempts: 3
      perTryTimeout: 2s
  - route:
    - destination:
        host: middleware
        subset: prod
    retries:
      attempts: 3
      perTryTimeout: 2s
```

Vous voyez que nous vérifions si l'en-tête "x-dev-user" correspond à "super-secret" et que nous dirigeons le trafic vers le _subset_ canary si c'est le cas. Sinon, le trafic est envoyé vers prod.

Appliquez la règle:

`kubectl apply -f ./configs/istio/routing-3.yaml`

À partir de maintenant, si vous avez le bon en-tête, Istio vous dirige automatiquement vers le bon service.

![Route to Canary](./x-dev-user.gif)

En travaillant avec Istio, il est important que tous vos services transfèrent tous les en-têtes dont ont besoin vos services en aval. Nous recommandons de standardiser avec quelques en-têtes et de s'assurer qu'ils sont toujours transférés lorsque vous communiquez avec un service en aval. Plusieurs librairies sont disponibles pour faire ça automatiquement.

# Surveillance et Traces

Un superbe avantage d'Istio est qu'il ajoute automatiquement la prise en charge de trace et de surveillance à vos applications. Alors que la surveillance est ajoutée sans intervention, le traçage nécessite que vous transfériez les en-têtes de traces insérées automatiquement par le _Ingress controller_ d'Istio pour qu'Istio puisse raccorder les requêtes entre-elles. Vous devez transférer les en-têtes suivantes dans votre code:

```
[
    'x-request-id',
    'x-b3-traceid',
    'x-b3-spanid',
    'x-b3-parentspanid',
    'x-b3-sampled',
    'x-b3-flags',
    'x-ot-span-context',
]
```

Le microservice que nous avons déployé le fait déjà.

## Visualiser les Traces

Vous pouvez mainenant ouvrir [Jaeger](http://localhost:16686/), sélectionnez le service "frontend" et cliquez "Find Traces". Istio échantillonne vos requêtes alors elles ne seront pas journalisées individuellement.
Pour les utilisateurs Cloud Shell, démarrez un _Web Preview_ à partir du terminal ayant servi à rouler `make start-monitoring-services` puis modifiez l'URL comme suit :
https://16686-dot-1111111-dot-devshell.appspot.com/   (changez le port à 16686)

Cliquez sur une Trace et vous pourrez voir les requêtes en cascade. Comme nous avons utilisé l'en-tête "fail" , nous pouvons même voir le mécanisme de retentative automatisée d'Istio en action!

![Jaeger Trace](./jaeger.gif)

## Visualiser les Métriques

Pour visualiser les métriques, ouvrez [Grafana](http://localhost:3001/dashboard/db/istio-dashboard)
Pour les utilisateurs Cloud Shell, démarrez un _Web Preview_ à partir du terminal ayant servi à rouler `make start-monitoring-services` puis modifiez l'URL comme suit :
https://3001-dot-1111111-dot-devshell.appspot.com/d/1/istio-mesh-dashboard?refresh=5s%3ForgId%3D1&orgId=1   (changez le port à 3001 et ajoutez le suffixe /d/1/istio-mesh-dashboard?refresh=5s%3ForgId%3D1&orgId=1)

Vous pouvez voir plusieurs métriques intéressantes dans le tableau de bord Istio par défaut ou le personnaliser selon vos besoins!

![Grafana Dashboard](./grafana.png)

# OpenCensus

Bien qu'Istio soit merveilleux pour les traces au niveau réseau et les métriques, il est aussi important de recueillir des informations sur ce qui se passe à l'*intérieur* de votre application. C'est ici qu'OpenCensus entre en jeu. OpenCensus est une librairie, non liée à un fournisseur, pour les traces et métriques concue pour plusieurs langages populaires.

Ajoutons OpenCensus à notre application pour voir comment ça peut aider!

## Transfert automatique des en-têtes

Si vous vous souvenez de la section trace de ce README, nous devions transférer manuellement tous ces en-têtes `x-blah-blah` pour chaque requête entrante ou sortante.

Un aspect intéressant d'OpenCensus est qu'il peut automatiquement transférer ces en-têtes pour la majorité des _frameworks_ serveur populaires.

Le code entier est disponible [ici](./code/code-opencensus-simple/index.js) mais la partie clé est au tout début:

```javascript
const tracing = require('@opencensus/nodejs');
const propagation = require('@opencensus/propagation-b3');
const b3 = new propagation.B3Format();

tracing.start({
	propagation: b3,
	samplingRate: 1.0
});
```

Juste à ajouter ce bout de code par dessus le vôtre et les en-têtes de trace seront ajoutés automatiquement à chaque requête individuelle! Vous pouvez modifier le `samplingRate` si vous ne voulez pas que chaque requête soit tracée (ce qui peut être accablant pour les grands systèmes).

Bien que notre application de test n'ait qu'une route et un appel en aval, vous remarquerez combien ça peut être utile pour un vrai serveur ayant plusieurs routes et appels en aval. Autrement, vous devriez ajouter le transfert d'en-têtes pour chaque route individuelle et chaque appel en aval vous-même!

Déployons ce code:

`make deploy-opencensus-code`

Vous ne devriez pas voir grand chose de différent (à part le texte indiquant "opencensus simple"), mais la grosse différence est que notre code n'a plus besion de tranférer manuellement ces en-têtes.

## Trace intégrée à l'application

Bien que de tracer les requête réseau ne soit fantastique, OpenCensus peut aussi tracer les appels de fonctions à l'intérieur de votre applicationn et les rattacher aux traces réseau! Ceci vous donne encore plus de visibilité sur ce qui se passe réellement dans vos microservices.

Le code entier est disponible [ici](./code/code-opencensus-full/index.js).

Dans le _setup_, nous devons indiquer à OpenCensus comment se connecter à  l'instance Jaeger qui roule dans le _namespace_ d'Istio.

```javascript
// Set up jaeger
const jaeger = require('@opencensus/exporter-jaeger')

const jaeger_host = process.env.JAEGER_HOST || 'localhost'
const jaeger_port = process.env.JAEGER_PORT || '6832'

const exporter = new jaeger.JaegerTraceExporter({
	host: jaeger_host,
    port: jaeger_port,
	serviceName: service_name,
});

tracing.start({
	propagation: b3,
	samplingRate: 1.0,
    exporter: exporter
});
```

Les variables d'environnement utilisées par notre code se retrouvent dans la [ConfigMap](./configs/opencensus/config.yaml) Kubernetes.
Dans la fonction `tracing.start` , nous ne faisons que passer l'exporteur créé avec les détails de Jaeger et c'est tout! Nous avons tout en place!

Vous pouvez maintenant créer des _spans_ personnalisés pour tracer ce que vous voulez. Comme OpenCensus crée automatiquement des "rootSpans" pour toute requête web entrante, vous pouvez commencer en créant facilement des "childSpans" à l'intérieur des routes:

```javascript
  const childSpan = tracing.tracer.startChildSpan('Child Span!')
  // Do Stuff
  childSpan.end()
```

Si vous voulez créer des "petits-enfants", assurez-vous de régler le "parentSpanId" à l'id du premier _childSpan_.

```javascript
  const childSpan = tracing.tracer.startChildSpan('Child Span!')
    // Do Child Stuff
    const grandchildSpan = tracing.tracer.startChildSpan('Grandchild Span!')
    grandchildSpan.parentSpanId = childSpan.id
    // Do Granchild Stuff
    grandchildSpan.end()
  childSpan.end()
```

Dans le code, nous calculons un nombre de Fibonacci aléatoire et attendons un peu. C'est un exemple un peu idiot mais ça démontre bien le traçage!

![Jaeger OpenCensus](./jaeger-opencensus.png)

Vous pouvez voir toutes les traces imbriquées et ce qui est génial c'est que vous pouvez voir les traces pour tous les trois microservices en même temps! C'est parce qu' OpenCensus et Istio utilisent les mêmes IDs à travers la pile (_stack_).

# Métriques personnalisées

Bien qu'Istio soit en mesure de fournir plusieurs métriques de niveau réseau, encore une fois il ne peut pas vous donner de métriques spécifiques à l'application que vous pourriez avoir besoin. Comme avec les traces applicatives, OpenCensus peut également vous aider avec des métriques de niveau applicatif.

**À venir!**

# Éteindre le cluster de test

`make delete-cluster`

ou

`make delete-cluster PROJECT_ID=votre-project-id-ici ZONE=votre-zone-ici`


# Avancé

## Personnalisez votre Deployment Istio

Si vous voulez personnaliser les composantes d'Istio qui seront installées, vous pouvez le faire avec Helm! Premièrement, vous devez télécharger la version 1.0 d'Istio qui contient les _charts_ Helm dont vous aurez besoin.

`make download-istio`

Ensuite, vous pouvez générer le YAML pour Istio qui a été utilisé pour cette démo avec la commande suivante:

`helm template istio-1.0.0/install/kubernetes/helm/istio --name istio --namespace istio-system --set global.mtls.enabled=true --set tracing.enabled=true --set servicegraph.enabled=true --set grafana.enabled=true > istio.yaml`

Soyez à l'aise de [modifier cette commande](https://istio.io/docs/reference/config/installation-options/) selon vos besoins mais prenez en considération que cette démo ne fonctionnera pas sans activer toutes ces composantes!

NOTE: Ceci n'est pas un produit officiel de Google
