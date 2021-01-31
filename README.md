# はじめに

Progressive Deliveryのデモ用のマニフェストファイルを置いてあります。


# 実行環境

- Docker for Mac 3.1.0
  - Kubernetes: v1.19.3
- Istio 1.8.2
- Prometheus: 2.21.0
- Flagger: 1.6.2


istio-injectionでdefault namespaceに対してinjectionを有効化しておいてください。

```bash
$ kubectl label namespace default istio-injection=enabled
namespace/default labeled
$
```

# 説明

## 注意事項

flagger loadtester、及びサンプルのマニフェストの適用するnamespaceにご注意ください。<br>
下記のように、一部namespaceはハードコードされています。

```yaml
      - name: load-test
        url: http://flagger-loadtester/
        timeout: 2s
        type: pre-rollout
        metadata:
          cmd: "hey -z 1m -q 10 -c 2 http://rollouts-demo-canary.default:8080/"
```

また、「うまく動かないな・・・」となった場合は、下記のようにcanaryの状態、またcontrollerのログを確認し、何かエラーが出ていないかを確認する事で、
大体の問題は解決できます。

```bash
$ kubectl get canary -w -o wide
$ kubectl logs -f -n istio-system {flagger-pod-name}
```

## 1.canary

単純なCanaryリリースを行うためのマニフェストファイルです。

```bash
$ kubectl apply -f ./1.canary 
canary.flagger.app/rollouts-demo created
deployment.apps/rollouts-demo created
gateway.networking.istio.io/rollouts-demo-gateway created
$ 
```

[http://localhost](http://localhost) にアクセスして画面が表示される事を確認しましょう。
次に、Deploymentsを更新します。
```bash
$ kubectl set image deployment/rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow
deployment.apps/rollouts-demo image updated
$
```

ブラウザの黄色の表示が増えていく様子がわかります。


## 2.blue-green

開発者が新バージョンでテストしたあと、手動でv2に切り替えるリリース方法です。
正確には、A/B Testingのリリース方法となります。

```bash
$ kubectl apply -f ./2.blue-green 
canary.flagger.app/rollouts-demo configured
deployment.apps/rollouts-demo configured
$ 
```

HTTPヘッダのUser-Agentとして、 `CloudNativeDays` を含んでいる場合のみ、新バージョンに対してアクセスします。<br>
新バージョンへの切替は、下記のように /gate/openを実行する事で、切替が行われます。
`{flagger-loadtester-pod-name}` は自身の環境のPod名を確認し、それに置き換えてください。

```bash
$ kubectl exec -it -n istio-system  {flagger-loadtester-pod-name} -- /bin/sh
/home/app $ curl  --include -d '{"name": "rollouts-demo","namespace":"default"}' http://localhost:8080/gate/open
```

また、/gate/openをした場合、その状態が保持されるので、再度手動での認証を挟みたい場合は、下記のようにcloseを呼び出す必要があります。

```bash
/home/app $ curl  --include -d '{"name": "rollouts-demo","namespace":"default"}' http://localhost:8080/gate/close
```
## 3-a.maintenance

単純にメンテナンス画面にblue/greenする場合です。

```bash
$ kubectl apply -f ./3-a.maintenance 
canary.flagger.app/rollouts-demo configured
deployment.apps/rollouts-demo configured
$ 
```

## 3-b.blue-green

メンテナンス画面後にblue/greenで公開する場合です。
開発者が先に

```bash
$ kubectl apply -f ./3-b.blue-green 
canary.flagger.app/rollouts-demo configured
deployment.apps/rollouts-demo configured
$ 
```

## 4.blue-green-mirror

現在稼働中のトラフィックを新バージョンに流してテストを行う方法です。<br>
`/gate/check` により切替を一時停止するようにしています。必要に応じて `/gate/close` を呼び出して止めるようにしてください。

```bash
$ kubectl apply -f ./4.blue-green-mirror 
canary.flagger.app/rollouts-demo configured
deployment.apps/rollouts-demo unchanged
$ kubectl set image deployment/rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow
deployment.apps/rollouts-demo image updated
$
```

## 5.blue-green-rollback

手動でのロールバック判断を行えるようにする方法です。

```bash
$ kubectl apply -f ./5.blue-green-rollback 
canary.flagger.app/rollouts-demo configured
deployment.apps/rollouts-demo unchanged
$ kubectl set image deployment/rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow
deployment.apps/rollouts-demo image updated
$
```

ロールバックを行う際は、下記を実行します。
`{flagger-loadtester-pod-name}` は自身の環境のPod名を確認し、それに置き換えてください。

```bash
$ kubectl exec -it -n istio-system  {flagger-loadtester-pod-name} -- /bin/sh
/home/app $ curl  --include -d '{"name": "rollouts-demo","namespace":"default"}' http://localhost:8080/rollback/open
```