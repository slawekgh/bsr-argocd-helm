jak w nazwie repo - omawiamy tu próbę realizacji BuildShipRun dla tandemu argocd+helm 

BSR to w tym wypadku i w tym kontekście :

- rozdział artefaktu generycznego od konfiguracji
- artefakt generyczny ma być pozbawiony konfiguracji
- konfiguracja per środowisko (prod, test, dev) ma być dostarczana wraz z release artefaktu. 




Przedstawione jest 5 podejść do proby realizacji tego via argocd zintegrowane z helm-charts:

- pierwsze rozwiązanie - 1 repo z helm-chart a w nim 3 x values
- drugie rozwiązanie - 3 x repo z helm-chart
- trzecie rozwiązanie - umbrella chart
- czwarte rozwiązanie - zmienne nadpisywane przez argo-aplikacje
- piąte rozwiązanie - 2 osobne repozytoria - jedno na helm-chart drugie na konfig  


  
<br>  
<br>  
<br>  
<br>  
<br>  
<br>  
  
  
  


Zaczynamy 


Oto proste repozytorium helma typu all-in-one - zawiera helm-chart + values:
https://github.com/slawekgh/argo-helm/tree/main/test-chart


sam helm-chart to:
- k8s-deployment , k8s-svc i k8s-cm zrobione w formie templates 
- jest też tester
- z racji tego że to all-in-one jest również values.yaml 


```
├── Chart.yaml
├── templates
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── NOTES.txt
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```


Zadanie polega na podziale frameworku opartego o tandem argocd+helm na środowiska DEV, TEST i PROD i zrobienie emulacji BuildShipRun na tymże tandemie (czyli jeden helm-chart ale wdrażany wielokrotnie , za każdym razem inaczej i bazujący za każdym razem na innych wartościach w helmowym values.yaml) 

Trzeba zatem ten sam helm-chart wdrożyć oddzielnie dla DEV, dla TEST i dla PROD 

Na potrzeby tego LAB będziemy używać jednego GKE i zrobimy podział na Namespaces dev, test i prod - z grubsza odpowiada to normalnemu podziałowi na 3 osobne GKE 

Tzw "różnice między środowiskami DEV, TEST i PROD" będą zaemulowane innymi portami k8s-svc i inną ilością replik (odpowiednio dla DEV : k8s-svc-port=2222 i 2repliki , dla TEST 3333+3repliki i dla PROD 4444+4r) 

W rzeczywistych środowiskach różnice są inne i jest ich znacznie więcej - tzn. diff między helmowym values-dev.yaml a values-prod.yaml może liczyć kilkadziesiąt parametrów - tu dla nas nie ma to znaczenia bo nas interesuje mechanizm a nie zawartość

Jednym słowem dążymy do modelu gdzie helm-chart jest jeden ale ma 3 różne metody wdrożenia - czyli upraszczając 3 różne pliki values (odpowiednio dla DEV, dla TEST i dla PROD)


## pierwsze rozwiązanie - 1 repo z helm-chart a w nim 3 x values

pierwsze intuicyjne (ale bardzo słabe) rozwiązanie jakie przychodzi do głowy to oczywiście dodać do repo 3 x plik Values i zdefiniować 3 aplikacje argo:

rozbijamy zatem w naszym repo values.yaml na 3 różne pliki 

```
├── Chart.yaml
├── templates
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── NOTES.txt
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
├── values-dev.yaml
├── values-prod.yaml
└── values-test.yaml
```

te values-X.yaml różnią się ilością replik i portami dla k8s-svc:

```
$ cat values-dev.yaml | grep -E "replicaCount|servicePort|namespace"
replicaCount: 2
namespace: dev
servicePort : 2222
$ cat values-test.yaml | grep -E "replicaCount|servicePort|namespace"
replicaCount: 3
namespace: test
servicePort : 3333
$ cat values-prod.yaml | grep -E "replicaCount|servicePort|namespace"
replicaCount: 4
namespace: prod
servicePort : 4444
```

startujemy z czystym argo i dodajemy repo + 3 x argo-app:

```
argocd@argocd-server-5fff657769-fhml5:~$ argocd repo list 
TYPE  NAME  REPO  INSECURE  OCI  LFS  CREDS  STATUS  MESSAGE  PROJECT

argocd@argocd-server-5fff657769-fhml5:~$ argocd app list 
NAME  CLUSTER  NAMESPACE  PROJECT  STATUS  HEALTH  SYNCPOLICY  CONDITIONS  REPO  PATH  TARGET

argocd@argocd-server-5fff657769-fhml5:~$ argocd repo add https://github.com/slawekgh/argo-helm
Repository 'https://github.com/slawekgh/argo-helm' added

argocd@argocd-server-5fff657769-fhml5:~$ argocd app create argo-helm-dev --repo https://github.com/slawekgh/argo-helm --path test-chart --dest-namespace dev --dest-server https://kubernetes.default.svc --auto-prune --sync-policy automated --release-name test-release --values values-dev.yaml
application 'argo-helm-dev' created

argocd@argocd-server-5fff657769-fhml5:~$ argocd app create argo-helm-test --repo https://github.com/slawekgh/argo-helm --path test-chart --dest-namespace test --dest-server https://kubernetes.default.svc --auto-prune --sync-policy automated --release-name test-release --values values-test.yaml
application 'argo-helm-test' created

argocd@argocd-server-5fff657769-fhml5:~$ argocd app create argo-helm-prod --repo https://github.com/slawekgh/argo-helm --path test-chart --dest-namespace prod --dest-server https://kubernetes.default.svc --auto-prune --sync-policy automated --release-name test-release --values values-prod.yaml
application 'argo-helm-prod' created
```

po pewnym czasie wszystko wygląda na zgodne z założeniami, argo-apps się zsynchronizowały, zaś w każdym NS jest tyle podów ile miało być i k8s-svc są na tych portach na których miały być:

```
argocd@argocd-server-5fff657769-fhml5:~$ argocd app list 
NAME                   CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                   PATH        TARGET
argocd/argo-helm-dev   https://kubernetes.default.svc  dev        default  Synced  Healthy  Auto-Prune  <none>      https://github.com/slawekgh/argo-helm  test-chart  
argocd/argo-helm-prod  https://kubernetes.default.svc  prod       default  Synced  Healthy  Auto-Prune  <none>      https://github.com/slawekgh/argo-helm  test-chart  
argocd/argo-helm-test  https://kubernetes.default.svc  test       default  Synced  Healthy  Auto-Prune  <none>      https://github.com/slawekgh/argo-helm  test-chart  

$ kk get po,svc -n dev 
NAME                                      READY   STATUS    RESTARTS   AGE
pod/test-release-deploy-7c9c7669c-dvjfc   1/1     Running   0          107s
pod/test-release-deploy-7c9c7669c-q9g9h   1/1     Running   0          107s

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/test-release   ClusterIP   10.108.8.247   <none>        2222/TCP   2m30s
$ kk get po,svc -n test
NAME                                      READY   STATUS    RESTARTS   AGE
pod/test-release-deploy-7c9c7669c-hf5nj   1/1     Running   0          2m28s
pod/test-release-deploy-7c9c7669c-rbs49   1/1     Running   0          2m28s
pod/test-release-deploy-7c9c7669c-zk948   1/1     Running   0          2m28s

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/test-release   ClusterIP   10.108.8.138   <none>        3333/TCP   2m28s
$ kk get po,svc -n prod
NAME                                      READY   STATUS    RESTARTS   AGE
pod/test-release-deploy-7c9c7669c-2vdzr   1/1     Running   0          2m26s
pod/test-release-deploy-7c9c7669c-n2kkd   1/1     Running   0          2m26s
pod/test-release-deploy-7c9c7669c-qt7bp   1/1     Running   0          2m26s
pod/test-release-deploy-7c9c7669c-zfhlb   1/1     Running   0          2m26s

NAME                   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/test-release   ClusterIP   10.108.2.95   <none>        4444/TCP   2m27s
```


rozwiązanie to mimo że proste i działa od ręki to jednak jest skrajnie niedoskonałe:
- po pierwsze zakłada że repo z helm-chart należy do nas i możemy sobie do niego wrzucać swoje pliki (a tak może nie być, może go utrzymywać zupełnie inna osoba/grupa/firma/community)
- po drugie burzy ideę BuildShipRun gdzie artefakt ma być generyczny i pozbawiony konfiguracji która to konfiguracja jest różna dla prod/test/dev oraz dogrywana podczas release na dev czy na prod
- po trzecie uniemożliwia jakąkolwiek separację danych między PROD a resztą niższych środowisk , może np. dojść do sytuacji że developer robiący jakieś swoje poc na dev wrzuci omyłkowo coś do prod itd itp 

<br>
<br>
<br>


## drugie rozwiązanie - 3 x repo z helm-chart

drugie rozwiązanie (również pozornie proste ale również pozbawione większego sensu) to skopiowanie chartu do 3 różnych repozytoriów

od tej pory trzeba będzie utrzymywać zawartość chartu w jakiejś synchronizacji co spowoduje że po chwili i tak wszystko się rozjedzie i zamieni się to w 3 osobne produkty

separacja między prod a nie-prod co prawda będzie ale BSR jest tu zupełnie nie zachowany i ryzyko różnic między PROD a środowiskami niższymi tak duże że rozwiązanie to raczej nie nadaje się do wdrożeń - a zatem nawet go nie będziemy tu modelować 


## trzecie rozwiązanie - umbrella chart

dodajemy kolejne repo (to repo będzie akurat shipem konfiguracyjnym dla środowiska DEV) które ma dependencies do centralnego repo z chartem :

https://github.com/slawekgh/helm-argo-ship-dev

w repo tym jest tylko dependency (w Chart.yaml) i values.yaml dla DEV

```
ship-repo-dev$ tree
├── Chart.yaml
├── README.md
└── values.yaml
```

w Chart.yaml umieszczamy dependency i w nim trzeba się odwołać do centralnego repo chartów helma  

głównym wyzwaniem tutaj będzie zrobienie na bazie naszego głównego repo nowego i prawidłowego repozytorium helmowego:

https://dev.to/frosnerd/using-a-private-github-repository-as-a-helm-chart-repository-5fa8
https://helm.sh/docs/topics/chart_repository/

zakładamy nowe repo:
https://github.com/slawekgh/argo-helm-chart-repository

idąc za tym że:

_chart repository is really just an HTTP server that hosts an index.yaml file together with a bunch of packaged charts in form of .tgz files._

trzeba zrobić TGZ oraz index.yaml 

TGZ robimy via ```helm package test-chart```

INDEX.YAML via ```helm repo index .   ```

powstaje: 
```
chart-repository$ tree
.
├── index.yaml
└── test-chart-0.8.tgz
```

trzeba jeszcze zmusić GitHuba żeby serwował pliki jak zwykły serwer HTTP - i tu z pomocą przychodzą ścieżki RAW jakie daje GitHuib - przykładowo:
```
https://raw.githubusercontent.com/slawekgh/argo-helm-chart-repository/main/index.yaml
```

a zatem komuś kto będzie chiał używać tego helm-repository taką ścieżkę trzeba będzie podawać: https://raw.githubusercontent.com/slawekgh/argo-helm-chart-repository/main

zaś w helm-chartach które będą się odwoływać do tego helm-repository trzeba będzie na końcu ich Chart.yaml dodefiniować:
```
dependencies:
- name: test-chart
  version: 0.8
  repository: https://raw.githubusercontent.com/slawekgh/argo-helm-chart-repository/main/
 
```

pozostaje zatem w repo-ship-dev dodać takie dependency do Chart.yaml i sprawdzić via helm dependency update czy to się poprawnie przemieli

```
ship-repo-dev$ tree
.
├── Chart.yaml
├── README.md
└── values.yaml

ship-repo-dev$ cat Chart.yaml 
apiVersion: v2
name: dev-chart
type: application
version: 1.12
appVersion: "3.6"
dependencies:
- name: test-chart
  version: 0.8
  repository: https://raw.githubusercontent.com/slawekgh/argo-helm-chart-repository/main/

```

pamiętając też o values - tutaj są nieco inaczej zdefiniowane gdyż sięgają (nadpisują) values z child-chartu - stąd musi być to tym razem w sekcji test-chart (chodzi o to wcięcie dla jasnośc):

```
ship-repo-dev$ cat values.yaml 
# Default values for test-chart
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.
test-chart:
  replicaCount: 2
  testerPodName: tester
  namespace: dev
  label: testlabel
  ala: alamakota
  servicePort : 2222
  PROTO : TCP
  targetPort : 8080
  obraz:
    image: gimboo/nginx_nonroot
    imagePolicy: Always
```


sprawdzamy czy to działa:

```
ship-repo-dev$ helm dependency update
Getting updates for unmanaged Helm repositories...
...Successfully got an update from the "https://raw.githubusercontent.com/slawekgh/argo-helm-chart-repository/main/" chart repository
Saving 1 charts
Downloading test-chart from repo https://raw.githubusercontent.com/slawekgh/argo-helm-chart-repository/main/
Deleting outdated charts

ship-repo-dev$ tree
.
├── Chart.lock
├── charts
│   └── test-chart-0.8.tgz
├── Chart.yaml
├── README.md
└── values.yaml
```

jak widać lokalnie (bez argo) się zrobiło - coś ściągnął więc jak widać działa to poprawnie 

teraz dodajemy to do argo 

```
argocd@argocd-server-5fff657769-fhml5:~$ argocd app list
NAME  CLUSTER  NAMESPACE  PROJECT  STATUS  HEALTH  SYNCPOLICY  CONDITIONS  REPO  PATH  TARGET
(czysto i pusto)
argocd@argocd-server-5fff657769-fhml5:~$ argocd app create argo-helm-dev --repo https://github.com/slawekgh/helm-argo-ship-dev --path . --dest-namespace dev --dest-server https://kubernetes.default.svc --auto-prune --sync-policy automated --release-name test-release --values values.yaml 
application 'argo-helm-dev' created

argocd@argocd-server-5fff657769-fhml5:~$ argocd app list 
NAME                  CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                            PATH  TARGET
argocd/argo-helm-dev  https://kubernetes.default.svc  dev        default  Synced  Healthy  Auto-Prune  <none>      https://github.com/slawekgh/helm-argo-ship-dev  .  
na klastrze argo wykonało założenia  
$ kk get po -n dev 
NAME                                  READY   STATUS    RESTARTS   AGE
test-release-deploy-7c9c7669c-6cbqc   1/1     Running   0          69s
test-release-deploy-7c9c7669c-vnxh9   1/1     Running   0          69s
```


analogicznie obok repo-ship-dev (https://github.com/slawekgh/helm-argo-ship-dev) należy założyć identyczne osobne repo dla test i osobne dla prod 


ich struktura będzie taka sama - mają tu być tylko dependency (w Chart.yaml) do helm-repository i values.yaml odpowiednie dla TEST i dla PROD 

```
ship-repo-dev$ tree
├── Chart.yaml
├── README.md
└── values.yaml
```

czy rozwiązanie z umbrella-chart ma sens ? 

ogólnie tak, ale:
- trzeba mu niestety dostarczyć "prawdziwe helm-repository" - czyli zamiast trzymać główny helm-chart w kodzie trzeba go za każdym razem pakować do TGZ i generować index.yaml
- dodatkowo potrzebny jest web-serwer do serwowania tych plików
- Jednym słowem idea wydaje się być dobra (jest separacja, jest BuildShipRun), niestety realizacja koncepcji jest nieco utrudniona i nieelastyczna 



## czwarte rozwiązanie - zmienne nadpisywane przez argo-aplikacje

do repo https://github.com/slawekgh/argo-helm/tree/main/test-chart dodajemy ponownie generyczny values (values-generic.yaml) który co prawda ma wpisane defaultowe wartości dla replicaCount, servicePort i namespace ale wg założeń nigdy nie powinny one być użyte :-)

```
helm-chart-repo$ cat test-chart/values-generic.yaml 
replicaCount: 1
testerPodName: tester
namespace: dev
label: testlabel
ala: alamakota
servicePort : 1111
PROTO : TCP
targetPort : 8080
obraz:
  image: gimboo/nginx_nonroot
  imagePolicy: Always
```

nie będą użyte ponieważ w tej metodzie zmienne replicaCount, servicePort oraz namespace będziemy nadpisywać 
<br>

dostępne opcje dla argo app create:
```
--helm-set stringArray                       Helm set values on the command line (can be repeated to set several values: --helm-set key1=val1 --helm-set key2=val2)
--helm-set-file stringArray                  Helm set values from respective files specified via the command line (can be repeated to set several values: --helm-set-file key1=path1 --helm-set-file key2=path2)
--helm-set-string stringArray                Helm set STRING values on the command line (can be repeated to set several values: --helm-set-string key1=val1 --helm-set-string key2=val2)
```

tutaj użyjemy --helm-set:
```
argocd@argocd-server-85f85d648b-tf2bx:~$ argocd repo list
TYPE  NAME  REPO  INSECURE  OCI  LFS  CREDS  STATUS  MESSAGE  PROJECT
argocd@argocd-server-85f85d648b-tf2bx:~$ argocd repo add https://github.com/slawekgh/argo-helm
Repository 'https://github.com/slawekgh/argo-helm' added

argocd@argocd-server-85f85d648b-tf2bx:~$ argocd app list
NAME  CLUSTER  NAMESPACE  PROJECT  STATUS  HEALTH  SYNCPOLICY  CONDITIONS  REPO  PATH  TARGET

argocd@argocd-server-85f85d648b-tf2bx:~$ argocd app create argo-helm-dev --repo https://github.com/slawekgh/argo-helm --path test-chart --dest-namespace dev --dest-server https://kubernetes.default.svc --auto-prune --sync-policy automated --release-name test-release --values values-generic.yaml --helm-set replicaCount=2 --helm-set servicePort=2222 --helm-set namespace=dev
application 'argo-helm-dev' created

argocd@argocd-server-85f85d648b-tf2bx:~$ argocd app create argo-helm-test --repo https://github.com/slawekgh/argo-helm --path test-chart --dest-namespace test --dest-server https://kubernetes.default.svc --auto-prune --sync-policy automated --release-name test-release --values values-generic.yaml --helm-set replicaCount=3 --helm-set servicePort=3333 --helm-set namespace=test
application 'argo-helm-test' created

argocd@argocd-server-85f85d648b-tf2bx:~$ argocd app create argo-helm-prod --repo https://github.com/slawekgh/argo-helm --path test-chart --dest-namespace prod --dest-server https://kubernetes.default.svc --auto-prune --sync-policy automated --release-name test-release --values values-generic.yaml --helm-set replicaCount=4 --helm-set servicePort=4444 --helm-set namespace=prod
application 'argo-helm-prod' created

argocd@argocd-server-85f85d648b-tf2bx:~$ argocd app listNAME                   CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                   PATH        TARGET
argocd/argo-helm-dev   https://kubernetes.default.svc  dev        default  Synced  Healthy  Auto-Prune  <none>      https://github.com/slawekgh/argo-helm  test-chart  
argocd/argo-helm-prod  https://kubernetes.default.svc  prod       default  Synced  Healthy  Auto-Prune  <none>      https://github.com/slawekgh/argo-helm  test-chart  
argocd/argo-helm-test  https://kubernetes.default.svc  test       default  Synced  Healthy  Auto-Prune  <none>      https://github.com/slawekgh/argo-helm  test-chart  
```

jak widać 3 argo-app się poprawnie zsynchronizowały, zaś na GKE jest również zgodnie z założeniami:


```
-----dev-------------
NAME                                  READY   STATUS    RESTARTS   AGE
test-release-deploy-7c9c7669c-8h498   1/1     Running   0          40s
test-release-deploy-7c9c7669c-b2nxt   1/1     Running   0          40s
NAME           TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
test-release   ClusterIP   10.108.5.59   <none>        2222/TCP   41s
-----test-------------
NAME                                  READY   STATUS    RESTARTS   AGE
test-release-deploy-7c9c7669c-jghr8   1/1     Running   0          18s
test-release-deploy-7c9c7669c-jlmxw   1/1     Running   0          18s
test-release-deploy-7c9c7669c-z46qx   1/1     Running   0          18s
NAME           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
test-release   ClusterIP   10.108.11.61   <none>        3333/TCP   19s
-----prod-------------
NAME                                  READY   STATUS    RESTARTS   AGE
test-release-deploy-7c9c7669c-55cg2   1/1     Running   0          5s
test-release-deploy-7c9c7669c-t8dcm   1/1     Running   0          4s
test-release-deploy-7c9c7669c-t8x5q   1/1     Running   0          4s
test-release-deploy-7c9c7669c-z8mp5   1/1     Running   0          4s
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
test-release   ClusterIP   10.108.10.248   <none>        4444/TCP   6s
```

SPEC argo-aplikacji wygląda następująco (tu na przykładzie argo-helm-dev) - zwracamy uwagę na sekcję parameters:
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  creationTimestamp: "2023-08-23T08:43:38Z"
  generation: 14
  name: argo-helm-dev
  namespace: argocd
  resourceVersion: "79461"
  uid: aa96eae1-ff2d-4699-8aaa-6f382309413b
spec:
  destination:
    namespace: dev
    server: https://kubernetes.default.svc
  project: default
  source:
    helm:
      parameters:
      - name: replicaCount
        value: "2"
      - name: servicePort
        value: "2222"
      - name: namespace
        value: dev
      releaseName: test-release
      valueFiles:
      - values-generic.yaml
    path: test-chart
    repoURL: https://github.com/slawekgh/argo-helm
  syncPolicy:
    automated:
      prune: true
```

oczywiście podobnie jak argo-app-dev również argo-app dla test i dla prod zawierają takie parametry
<br>

zalety:
- jest separacja środowisk PROD, DEV i TEST
- jest też podział na część generyczną i część konfiguracyjną (czyli BuildShipRun jest w miarę spełniony) 

wady:
- trzymanie konfiguracji środowiska w parametrach argo-application
- gdy zmiennych będą dziesiątki oczywiście raczej następujące podejście zupełnie się nie sprawdzi:
```
argocd app create argo-helm-dev --repo https://github.com/slawekgh/argo-helm --path test-chart 
--dest-namespace dev --dest-server https://kubernetes.default.svc --auto-prune --sync-policy automated 
--release-name test-release --values values-generic.yaml --helm-set replicaCount=2 --helm-set servicePort=2222 
--helm-set namespace=dev  --helm-set parametr1=xxx  --helm-set parametr2=xxx --helm-set parametr3=xxx 
--helm-set parametr4=xxx --helm-set parametr5=xxx --helm-set parametr6=xxx--helm-set parametr7=xxx  itd itd 
```
- również nie sprawdzi sie zarządzanie zmianami w konfiguracji w ten sposób:
```
    $ argocd app set argo-helm-dev -p replicaCount=8 
    $ argocd app set argo-helm-dev -p replicaCount=9 -p servicePort=9999 -p parametr1=xxx itd 
``` 

jedyna opcja tutaj to zarządzanie argo-app z kodu , czyli trzymanie kodu argo-application dla dev, dla test i dla prod w repo i wgrywanie na klaster via kubectl apply -f (oczywiście via CICD, Jenkins lub inne argo-administracyjne-główne-nadrzędne)


wtedy pamiętać musimy że argo-application-dev.yaml zawiera już konfig dla DEV 

jednym słowem konfiguracja środowiska jest trzymana w konfiguracji argo-app - jest to słabe rozwiązanie 

dodatkowo w materiałach na YT jako wadę tego podejścia wskazuje się brak możliwości wywołania lokalnego helm install/upgrade (no bo values mamy w argo a nie w repo czy pod ręką w pliku) 


## piąte rozwiązanie - 2 osobne repozytoria - jedno na helm-chart, drugie na konfig

rozwiązanie z umbrella-chart było prawie idealne gdyby nie następujące cechy:
- konieczność robienia prawdziwego helm-repository (opartego o serwujący pliki webserwer czy GCS itp) 
- produkcja TGZ 
- trzeba generować index.yaml 
<br>
<br>
rozwiązanie ze zmiennymi nadpisywanymi przez argo-aplikacje też byłoby ok gdyby nie konieczność trzymania całego konfigu aplikacji w obiektach argo (które nie do tego są)

<br>
<br>

dlatego dobrym kompromisem wydaje się być model gdzie pozostajemy przy klasycznej koncepcji zwykłego repo kodu z głównym helm-chart ale dodawania do niego konfiguracji z innego repo 

takie rozwiązanie jest ale jak na razie jest to rozwiązanie beta i weszło dopiero w ARGOCD 2.6:

https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/#helm-value-files-from-external-git-repository
https://argo-cd.readthedocs.io/en/stable/user-guide/helm/


tu kluczowy fragment:
<br>

_before v2.6 of Argo CD, Values files must be in the same git repository as the Helm chart. The files can be in a different location in which case it can be accessed using a relative path relative to the root directory of the Helm chart. As of v2.6, values files can be sourced from a separate repository than the Helm chart by taking advantage of multiple sources for Applications._


poza tym (być może z racji że to beta) mimo kilku prób nie udało mi się wykonać takiego setupu via argocd app create a jedynie definiując w YAMLu obiekty argo-Application - przynajmniej tak jest na dzień 23.08.2023 lub po prostu nie znalazłem na szybko sposobu 

celujemy w setup:

- jako repo główne (repo helm-chartu) uzyjemy https://github.com/slawekgh/argo-helm/tree/main/test-chart
- jako repo konfugiracji dla środowiska DEV użyjemy repo: https://github.com/slawekgh/helm-argo-ship-dev


oto struktura obydwu tych repo (pokazana w uproszczeniu , czyli z pominiętymi elementami które nie biorą udziału w tym setupie) :
```
helm-chart-repo$ tree
.
├── README.md
└── test-chart
    ├── Chart.yaml
    ├── templates
    │   ├── configmap.yaml
    │   ├── deployment.yaml
    │   ├── NOTES.txt
    │   ├── service.yaml
    │   └── tests
    │       └── test-connection.yaml


ship-repo-dev$ tree
.
├── README.md
└── values-dla-srodowiska-dev.yaml
```

w ARGO mamy czystą sytuację:

```
argocd@argocd-server-85f85d648b-tf2bx:~$ argocd app list
NAME  CLUSTER  NAMESPACE  PROJECT  STATUS  HEALTH  SYNCPOLICY  CONDITIONS  REPO  PATH  TARGET
argocd@argocd-server-85f85d648b-tf2bx:~$ argocd repo list
TYPE  NAME  REPO  INSECURE  OCI  LFS  CREDS  STATUS  MESSAGE  PROJECT
argocd@argocd-server-85f85d648b-tf2bx:~$
```
Trzeba dodać obiekt argo-aplikacji który pracuje na dwóch repozytoriach jednocześnie 


aby ułatwić sobie zadanie można w argo dodać z ręki prostą aplikację:
```
argocd app create argo-helm-dev --repo https://github.com/slawekgh/argo-helm --path test-chart --dest-namespace dev --dest-server https://kubernetes.default.svc --auto-prune --sync-policy automated --release-name test-release --values values-generic.yaml
```
i potem zrobić jej edycję via ```kk -n argocd edit application argo-helm-dev```

```
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  creationTimestamp: "2023-08-23T12:12:59Z"
  generation: 54
  name: argo-helm-dev
  namespace: argocd
  resourceVersion: "183619"
  uid: cb3998cf-4fe5-4fd9-bd45-50e38ac87298
spec:
  destination:
    namespace: dev
    server: https://kubernetes.default.svc
  project: default
  sources:
  - helm:
      releaseName: test-release
      valueFiles:
      - $values/values-dla-srodowiska-dev.yaml
    path: test-chart
    repoURL: https://github.com/slawekgh/argo-helm
  - ref: values
    repoURL: https://github.com/slawekgh/helm-argo-ship-dev
  syncPolicy:
    automated:
      prune: true
```


od tej pory konfigurację środowiska trzymamy wyłącznie w repo z konfigiem i zmiany (parametry) konfiguracyjne modyfikujemy wyłącznie w tym repo 

celem wprowadzenia tego samego setupu dla TEST i PROD należy stworzyć identyczne repozytoria konfiguracyjne oraz dodać kolejne 2 aplikacje argo wskazujące na te repozytoria 

podejście 5te wydaje się być najlepsze a jedyną jego wadą jest to że jest to jeszcze beta , nie jest obecnie wspierana w ARGOCD-WEBGUI oraz w cli (via argocd app create)


