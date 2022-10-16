# Експериемент 02 - лаб02

Целта на това упражнение е да илюстрираме важната роля на контролерите в кубернетис и да демонстрираме един забавен инструмент който намира приложение в "chaos engineering".
За целата ще създадем отделен клъстър с KIND в който ще създадем подоб

## инструкции

Ще започне като клонираме GitHub репозиторито на [Kube DOOM](https://github.com/storax/kubedoom)

```
git clone https://github.com/storax/kubedoom.git
```

README.md файла съдържа интересни пдоброности за проекта и как да се използва със Docker, Podman и K8s. В нашето упражнение ще следваме иснтрукциите за K8s.


### Създаване на K8s клъстър с KIND


```
kind create cluster --name kubedoom --config kind-config.yaml

✓ Ensuring node image (kindest/node:v1.23.0) 🖼
 ✓ Preparing nodes 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kubedoom"
You can now use your cluster with:

kubectl cluster-info --context kind-kubedoom

Thanks for using kind! 😊

```

в резултат ще бъде създаден клъстър състояще се от два нода. Порт 5900 ще бъде достъпен от "worker" нода. През него ще се свържем със VNC клиент към приложението работещо в под.


## Създаване на ресурсите необходиими за Kube DOOM

```
kubectl apply -k manifest/
```

По подразбиране Kube Doom ще е ограничен да убива подове намиращи се в "default" неймспейса.
Нека да променим това симулирайки реална ситуация където бихме желали да тестваме устойчивостта на истинско приложение:

* Нека създадем нов неймспейс: `doomedpods`
```
kubectl create ns doomedpods
```

* Сега трябва да кажем на Kube DOOM с подове от кой неймспейс да работи.
В документацията пише че това става като се редактира стойността на променливата на средата: `NAMESPACE`, как обаче да я променим в пода ?

Както би трябвало да бъде всяко приложение, пода на Kube Doom се управлява от деплоймънт контролер. Нека да го потърсим:
```
kubectl get deploy -A
kc get deploy -A
NAMESPACE            NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
kube-system          coredns                  2/2     2            2           12m
kubedoom             kubedoom                 1/1     1            1           7m45s
local-path-storage   local-path-provisioner   1/1     1            1           12m
```

тук `deploy` е съкратеното име на ресурса: `deployment`. Флага -А показва, че търсим ресурс деплоймънт във всички неймспейсове.

за да не изписваме цялото име на kubectl всеки път и да си спестим малко писане, можем да създадем alias (псевдоним):
```
alias kc=kubectl
```

нека да разгледаме настройките на деплоймънта:
```
$ kc -n kubedoom get deploy/kubedoom -o json | jq .spec.template.spec.containers

[
  {
    "env": [
      {
        "name": "NAMESPACE",
        "value": "test"
      }
    ],
```

Ако искаме да взмем само стойността на променливата можем да използваме:
```
kc -n kubedoom get deploy/kubedoom -o json | jq '.spec.template.spec.containers[].env[].value'
```

-o, съкратено от "--output" е флаг позволяващ да се модифицира получения резултат от заявката. Повече примери можете да намерите във: https://kubernetes.io/docs/reference/kubectl/cheatsheet/

В горния пример желаем да получим резултата сериализиран в JSON  за да можем да използваме инструмента `jq` за по-удобно индексиране на резултата

От резултата виждаме че подразбиращта се стойност на променливата на средата NAMESPACE е  "test". В кубернетис можем да направим това по няколко начина:

* да запазим дефиницията на ресурс като json, да я редактираме и да използваме `kubectl apply` за да я обновним в k8s
  ```
  kc -n kubedoom get deploy/kubedoom -o json > kubedoom.json
  vi kubedoom.json
  kc apply -f ./kubedoom.json
  ```

* да редактираме директно дефиницията на ресурса използвайки интерактивен редактор:
  ```
  kc -n kubedoom edit deploy/kubedoom
  ```

* да редактираме стойноста на променливата директно:
  ```
  kc -n kubedoom set env deploy/kubedoom -c kubedoom NAMESPACE=doomedpods
  ```

* До тук добре но дали нашите промени са били обновени. нека да проверим:
```
kc -n kubedoom get po
NAME                        READY   STATUS    RESTARTS   AGE
kubedoom-5bf679747c-6nj7w   0/1     Pending   0          4s
kubedoom-857458d6f-h8bqk    1/1     Running   0          5m1s
```

Деплоймънт контролера се опитва да създаде нов под с нашите нови настройки но нещо го блокира. Нека видим какво:
```
kc -n kubedoom describe po/kubedoom-5bf679747c-6nj7w
...
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  50s   default-scheduler  0/2 nodes are available: 1 node(s) didn't have free ports for the requested pod ports, 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate.
```

Описвайки пода който е блокиран в "pending state" забелязваме че той не може да бъде разпределен на никой нод понеже съществуващия под вече е заел порта a няма други хостове в клъстъра където новия под може да бъде разположен.

Знаейки какъв е проблема сега можем да го остраним просто като ръчно изтрием оригиналния под със старата конфигурация:

```
kc -n kubedoom delete po/kubedoom-857458d6f-h8bqk
```

### Свързване към Kube DOOM

За да се свържем ни е необходим VNC клиент. Подразбиращата се парола е: `idbehold`

Примери:

MAC:
* Safari:
```
vnc://127.0.0.1:5900
```

* Tiger-VNC
```
brew install tiger-vnc
```

* vnc-viewer
```
$ vncviewer viewer localhost:5900
```

Windows
* ultra-VNC - https://uvnc.com/downloads/ultravnc.html


Linux
* tigervnc, realvnc...

Съгласно документацията:
```
You should now see DOOM! Now if you want to get the job done quickly enter the cheat idspispopd and walk through the wall on your right. You should be greeted by your pods as little pink monsters. Press CTRL to fire. If the pistol is not your thing, cheat with idkfa and press 5 for a nice surprise. Pause the game with ESC.
```


### Нека да направим малко подобве(зомбита)
Нека направим два вида подовете в неймспейса: `doomedpods`
* статичен под, който не се управлява от деплоймънт
```
kc -n doomedpods run zombie01 --image nginx
pod/zombie01 created
```

Ако сега убиием зомбито с името на този под какво ще стане ?

* Няколко пода управялвани от деплоймънт
да пробваме сега да създадем под (или няколко) управлявани от деплоймънт

```
kc create deploy zombiequieen --image nginx -r 1
deployment.apps/zombiequieen created

```
Колко зоомбита се появяват сега на арента ? Колко от тях се връщат към живота ?


### заключение

В това упражнения видяхме някой практически похвати за работa с кубернетис ресурси. Видяхме защо не е препоръчително да се използват подове не управлявани от контролери - когато по някаква причина те биват изтрити или спрат, нищо не се грижи за тяхното възстановявано.

Контролерите от друга страна непрекъснато следят за желаното състояние на ресуристе управлявани от тях и влучай на отклонение вземат автоматично мерки за отстраняване на проблемите.
