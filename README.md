# TensorFlowAdventCalendar2018

この記事はTensorFlow Advent Calendar 2018年 3日目の記事です。

今年のAdvent CalendarはKubeflow pipeliensについて加工と思います。

https://cloud.google.com/blog/products/ai-machine-learning/getting-started-kubeflow-pipelines


− 
# 機械学習システムの実環境へのデプロイ&サービング
- 機械学習が普及した2018年ですが、PoC(Proof of Concept)を超えて実運用まで漕ぎ着けている事例が増えてきたとはいえ、実システムに組み込んで運用する場合のハードルは依然高いように見えます。 

- High...
- TFX
- しかしそれぞれのコンポーネントがバラバラだと機械学習のワークフロー全体としては管理しづらく、再利用性もないため依然技術的負債感が拭えません。
- そこで機械学習のワークフロー全体をEndToEndで管理できるようにするためのコンポーネントがkubeflow pipelineです。
- kubeflow pipeline自体はkubeflow上にワークフローをpythonをベースにしたDSLとして記述してDeployするためのSDKや、機械学習のexperimentsやjobを管理する機能があります。
- ワークフローマネジメント自体はKubeflowのCoreComponentである、Argoが動いているらしいですが、UIが整い、やっと統一感があるpipeline管理ツールが出てきたなというところです。

- 2017年末にkubeflowが出てきてから丸一年、kubeflow自体はまだ0.4と発展途上であり、公式のexamplesもまともに動かなかったりします。このkubeflow pipelinesも例に漏れずexampleを動かすのさえ苦行ではありますが、ユーザーが増えて知見が貯まることを願ってご紹介をしようと思います。


# Kubeflow Pipelines examples
今回はkubeflowのslack(https://kubeflow.slack.com)で紹介されていたこのexample(https://github.com/amygdala/code-snippets/tree/master/ml/kubeflow-pipelines)を試してみます。kubeflow/pipelines(https://github.com/kubeflow/pipelines)の公式のrepoではありませんが、GKEを使ってkubeflow-pipelines上にTFXのコンポーネントを使って機械学習のワークフローをデプロイしていく良いsampleです。

基本的にはREADME.md(https://github.com/amygdala/code-snippets/blob/master/ml/kubeflow-pipelines/README.md)に書かれている通りに動かしますが、そのままではエラーが出る部分などあるのでWorkaroundも示しながら進めたいと思います。

# Instration and setup
まずはGCPの環境を整えます。
- GCPプロジェクトを作る
- 必要なAPIをenableにする
  - Cloud Machine Learning Engine、Kubernetes Engine、オプションでTFTやTFMAをDataflow上で動かしたり、データソースをcsvファイルからBigQueryに変えるなどする場合はそれぞれEnableする必要があります
- gcloud sdkをインストールする、もしくはcloud shellを使う
- GCSのバケットを用意しておきます。
  - https://github.com/kubeflow/pipelines/wiki/Deploy-the-Kubeflow-Pipelines-Service#deploy-kubeflow-pipelines
  - Backet名はXXXにしてあります。

## Set up a Kubernetes Engine (GKE) cluster
この通りにGKEクラスタを作成します。https://github.com/amygdala/code-snippets/blob/master/ml/kubeflow-pipelines/README.md#set-up-a-kubernetes-engine-gke-cluster

```
gcloud beta container --project <PROJECT_NAME> clusters create "kubeflow-pipelines" --zone "us-central1-a" --username "admin" --cluster-version "1.9.7-gke.11" --machine-type "custom-8-40960" --image-type "COS" --disk-type "pd-standard" --disk-size "100" --scopes "https://www.googleapis.com/auth/cloud-platform" --num-nodes "4" --enable-cloud-logging --enable-cloud-monitoring --no-enable-ip-alias --network "projects/mlops-215604/global/networks/default" --subnetwork "projects/<PROJECT_NAME>/regions/us-central1/subnetworks/default" --addons HorizontalPodAutoscaling,HttpLoadBalancing,KubernetesDashboard --enable-autoupgrade --enable-autorepair
```
PROJECT_NAMEには使っているGCPのプロジェクト名を入れて下さい。

作ったクラスタをコンテキストに割り当て、ClusterRoleリソースを作成します。
```
> gcloud container clusters get-credentials kubeflow-pipelines --zone us-central1-a --project mlops-215604
Fetching cluster endpoint and auth data.
kubeconfig entry generated for kubeflow-pipelines.
> kubectl create clusterrolebinding ml-pipeline-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)
clusterrolebinding.rbac.authorization.k8s.io "ml-pipeline-admin-binding" created
> kubectl create clusterrolebinding sa-admin --clusterrole=cluster-admin --serviceaccount=kubeflow:pipeline-runner
clusterrolebinding.rbac.authorization.k8s.io "sa-admin" created
```
## Install Kubeflow with Kubeflow Pipelines on the GKE cluster
Kubeflowのこのページ(https://www.kubeflow.org/docs/guides/pipelines/deploy-pipelines-service/)の中のDeploy Kubeflow Pipelinesに従います。

```
> PIPELINE_VERSION=0.1.2
> kubectl create -f https://storage.googleapis.com/ml-pipeline/release/$PIPELINE_VERSION/bootstrapper.yaml
clusterrole.rbac.authorization.k8s.io "mlpipeline-deploy-admin" created
clusterrolebinding.rbac.authorization.k8s.io "mlpipeline-admin-role-binding" created
job.batch "deploy-ml-pipeline-qqk9j" created
 > kubectl get job
NAME                       DESIRED   SUCCESSFUL   AGE
deploy-ml-pipeline-qqk9j   1         1            7m
> kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.7.240.1   <none>        443/TCP   18m
```

Kubeflow pipelines UIにローカルのブラウザからGKE上のpod内のコンテナにアクセスできるようにポートフォワードの設定をしておく。
```
> export NAMESPACE=kubeflow
> kubectl port-forward -n ${NAMESPACE} $(kubectl get pods -n ${NAMESPACE} --selector=service=ambassador -o jsonpath='{.items[0].metadata.name}') 8080:80

```
この状態でCloud shellから"Web Preview"するとKubeflowのとても簡素なダッシュボードに飛びます。
またそのURLの末尾に/pipelinesを追加することでKubeflow pipelinesのUIに移れます。

以上で、README.mdに記載上はKubeflow pipelinesのインストールは終わりました。しかし、これ以降のExamplesを動かすためにもう少し準備をします。

Examplesを完走するためには、下記2点が必要です。
- Jupyter notebookへ設定を追加する
- 必要なJupyter extensionをインストールする

## Jupyter notebookへ設定を追加する
GKE上にjupyternotebookのサービス(?)は立ち上がるのですが、新しいnotebookを起動できません。
このissue(https://github.com/kubeflow/pipelines/issues/179)を参考にしてFixすることができました。

まずはjupyter hubからイメージを選択し、Spawnします。
今回は、
- gcr.io/kubeflow-images-public/tensorflow-1.10.1-notebook-cpu:v0.3.1
を選択しました。

立ち上げたら
https://github.com/kubeflow/pipelines/issues/179
```
# Check for the Jupyter pod name `jupyter-<USER>` (Here user is 'admin')
kubectl get pods -n kubeflow

kubectl exec -it -n kubeflow jupyter-taketoshi-2ekazusa-40brainpad-2eco-2ejp　bash
jovyan@jupyter-admin:~$ vim .jupyter/jupyter_notebook_config.py
```
c.NotebookApp.allow_origin='*'を追記

再起動。
```
jovyan@jupyter-admin:~$ ps -auxw
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
jovyan       1  0.0  0.0   4528   820 ?        Ss   12:44   0:00 tini -- start-singleuser.sh --ip="0.0.0.0" --port=8888 --allow-root
jovyan       5  0.1  0.8 292128 62168 ?        Sl   12:44   0:01 /opt/conda/bin/python /opt/conda/bin/jupyterhub-singleuser --ip="0.0.0.0" --port=8888 --allow-root
jovyan      33  0.0  0.0  20316  4108 ?        Ss   12:52   0:00 bash
jovyan      41  0.0  0.0  36080  3300 ?        R+   12:53   0:00 ps -auxw

jovyan@jupyter-admin:~$ kill 1
jovyan@jupyter-admin:~$ command terminated with exit code 137

<USER>@CloudShell:~$ kubectl get pods  -n kubeflow |grep jupyter
jupyter-admin                                            1/1       Running   2          16m

export NAMESPACE=kubeflow
kubectl port-forward -n ${NAMESPACE} $(kubectl get pods -n ${NAMESPACE} --selector=service=ambassador -o jsonpath='{.items[0].metadata.name}') 8080:80
```
これでnotebookを立ち上げることができるようになりました。



## Juptyter notebookでTFMAを可視化するための extensionのインストール
TFMAはインタラクティブにデータをスライスし、その結果をjupyter notebook上で可視化することができますが、そのためにExtensionをインストールする必要があります。

```
# --system を付けた方が良いかも
> jupyter nbextension enable --py widgetsnbextension
> jupyter nbextension install --py --symlink tensorflow_model_analysis
エラーが出る。。。
> jupyter nbextension enable --py tensorflow_model_analysis
```
ここで、permission denyが出てしまい、結局TFMAのレンダリングがされないままでした。解決しましたら追記します！



## Running the examples
Kubeflow pipelineに機械学習のpipelineを定義していきます。
Kubeflow pipelinesのUIに入るとすでにいくつかサンプルのPipelineが定義されています。

**スクショ**



## workflow1をやってみる
ここではすでに定義されたPipelineではなくて、新しくPipelineを定義して実行してみます。まずはDSLで書かれたスクリプトをコンパイルします。手順はこちらです。
- https://github.com/amygdala/code-snippets/blob/master/ml/kubeflow-pipelines/samples/kubeflow-tf/README.md#example-workflow-1


```
> cd ~/code-snippets/ml/kubeflow-pipelines/samples/kubeflow-tf
> python3 workflow1.py
> ls
README.md  workflow1.py  workflow1.py.tar.gz  workflow2.py
```
Kubeflow pipelines UIにこの`workflow1.py.tar.gz`をアップロードするとpipelineができます。


** スクショ **

UIからExperimentsを設定し、Runさせます。
このとき`preprocess-mode`と`tfma-mode`を`local`で実行していますが、ここを'cloud'にするとDataflowで動作します。


この後、pipelineが動きます。






- https://www.kubeflow.org/docs/guides/components/jupyter/
browser previewでkubeflowにアクセス、jupyterhubへ。
- signin はGCPアカウント。
- cloudshellから~/~/code-snippets/ml/kubeflow-pipelines/components/dataflow/tfma/tfma_expers.ipynbをダウンロード
- jupyter上へ


https://github.com/tensorflow/model-analysis

```
kubectl -n ${NAMESPACE} describe pods jupyter-taketoshi-2ekazusa-40brainpad-2eco-2ejp


```

- 

- どうやってTensorflowを含む機械学習フレームワークを使ったモデルをシステムにデプロイし、継続的に使い続けるか
- Tensorflow Data Validationだけの評価でしてもしょうがないのでどの程度MLworkflowが展開しやすいのか調べる
- Kubeflow pipeline上で実現したい。KubeflowはTFXを包含。そのワークフローの中で活用している。
- KubeflowはGKE上にデプロイする。TFDVはApache Beamが必要になるから、Dataflowとか繋げられるの便利。
- On-preでやりたいよってところはCiscoのUCSでkubeflowサポートしてるし、ハイブリッドでやるならGKEとUSCで、管理ツールとしてGKE-onpreかな

# 用語の整理
- kubeflow pipelines(https://github.com/kubeflow/pipelines)
  - Kubeflow Pipelines SDK ってなんやhttps://github.com/kubeflow/pipelines/wiki
  − Documents are here -> https://github.com/kubeflow/pipelines/wiki
  - パイプラインをpythonDSLで記述できる
- TFJob CRD(Cu7stom Resource Definitions library): Kube上でTFの分散学習できるようにする
  - https://github.com/kubeflow/tf-operator/blob/master/tf_job_design_doc.md
  - 基本的なリソースはすでにkubernetesに定義されているが、それ以外のものをCustomeで定義できる。(https://thinkit.co.jp/article/13610)
  - TFjob はkubeflowで定義したCustomResourceDefinitions？。Driverとかのマウントが必要無くなる
    - TFJobSpecs:The actual specification of our TensorFlow job, defined below.
    - TFReplicaSpec: Specification for a set of TensorFlow processes, defined below.
  - これを動かすと、tf-job-operator-configというConfigMap、Deployment, "tf-job-operator"というPodができる
  - TFJob is a Kubernetes custom resource that makes it easy to run TensorFlow training jobs on Kubernetes.
  
  


## Introducting Kubeflow Pipelines
- Kubeflow Pipelines はML workflowsをE2Eでmanagementするツール
- https://github.com/amygdala/code-snippets/tree/master/ml/kubeflow-pipelines
- https://github.com/tensorflow/model-analysis/tree/master/examples/chicago_taxiこれと同じ



## TFX building blocks
- Kubeflowは TFXのコンポーネントをブロックとして使ってます。
- TensorFlow Transform, TEMA, TF serving


## TFT
- tf.transform はtraning-serving skewを解決する、それはtrainとtestで前処理の仕方が違う。チームが異なってたり、計算資源が異なると起こり得る。
- TFTのアウトプットはTF graphとして出力される
- TFT はApache Beamを使ってTFTするよ、Google Cloud DataflowはManagedのApache Beamだよ
- ExamplesはlocalのBeamを使っているけど。dataflowを使うこともできるよ
− 

## TFMA
- TFMA は様々な状況下や特徴、subsetにおけるモデルのパフォーマンスをビジュアライズしてくれる
− TFT と同様にBeamが必要。
- ExampleはTFDVを直接は使ってないけど、schemaはTFDVによって作られたものだよ

## Cloud ML Engine Online Prediction
- 今回のexampleじゃ使ってないよ
- モデルの管理ができていいよ

## Building workflows using kubeflow pipelines 

- https://medium.com/tensorflow/introducing-tensorflow-data-validation-data-understanding-validation-and-monitoring-at-scale-d38e3952c2f0
- https://github.com/tensorflow/data-validation


- https://chinagdg.org/2018/11/getting-started-with-kubeflow-pipelines/


## Tensorflow Data Validation
- 統計量のサマリを出してくれる
- それぞれのFeatureのデータの分布や統計量、TrainとTestの差を見せてくれる
- データスキーマを勝手に作ってくれる、どんな値をとるか、範囲、ボキャブラリー(?)
- 異常値を探してくれる。

##
Kubeflow piplinesでやるよ
- https://chinagdg.org/2018/11/getting-started-with-kubeflow-pipelines/



### Examples
- https://github.com/amygdala/code-snippets/tree/master/ml/kubeflow-pipelines

↓コマンドラインで
https://github.com/kubeflow/pipelines/wiki/Deploy-the-Kubeflow-Pipelines-Service#deploy-kubeflow-pipelines

## はじめに
- gcloud をインストールする
- APIsをEnableにしておく
  - Dataflow
  - BigQuery
  - Cloud ML Engine
  - Kubefnetes Engine
  
 ## Set up a Kubernetes Engine (GKE) cluster
 - GKEクラスタを立ち上げる
  - Machine type 8 vCPUs、Allow full accessto all cloud APIs
  - 詳細はスクショ
 - 'kubectl' コンテクストをセットします。
 
```
 gcloud beta container --project "mlops-215604" clusters create "kubeflow-pipelines" --zone "us-central1-a" --username "admin" --cluster-version "1.9.7-gke.11" --machine-type "custom-8-40960" --image-type "COS" --disk-type "pd-standard" --disk-size "100" --scopes "https://www.googleapis.com/auth/cloud-platform" --num-nodes "4" --enable-cloud-logging --enable-cloud-monitoring --no-enable-ip-alias --network "projects/mlops-215604/global/networks/default" --subnetwork "projects/mlops-215604/regions/us-central1/subnetworks/default" --addons HorizontalPodAutoscaling,HttpLoadBalancing,KubernetesDashboard --enable-autoupgrade --enable-autorepair
```
 
 
 ###  cloud shellの中で
```
gcloud container clusters get-credentials kubeflow-pipelines --zone us-central1-a --project mlops-215604
 
# kubectl create clusterrolebinding default-admin --clusterrole=cluster-admin --user=taketoshi.kazusa@brainpad.cp.jp
> kubectl create clusterrolebinding ml-pipeline-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)

clusterrolebinding.rbac.authorization.k8s.io "ml-pipeline-admin-binding" created

> kubectl create clusterrolebinding sa-admin --clusterrole=cluster-admin --serviceaccount=kubeflow:pipeline-runner

clusterrolebinding.rbac.authorization.k8s.io "sa-admin" created
```


```
PIPELINE_VERSION=0.1.2
kubectl create -f https://storage.googleapis.com/ml-pipeline/release/$PIPELINE_VERSION/bootstrapper.yaml
```

```
kubectl get job
NAME                       DESIRED   SUCCESSFUL   AGE
deploy-ml-pipeline-rt8zf   1         1            14h

kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.7.240.1   <none>        443/TCP   14h
```
あれ？kubernetes本体以外のサービスたってなくね？ポートフォワード意味ある？

ここ以降は下記を読みなさい。
https://github.com/amygdala/code-snippets/tree/master/ml/kubeflow-pipelines/samples/kubeflow-tf

## Install python SDK (python 3.5 above) 
https://github.com/kubeflow/pipelines/releases
```
pip3 install https://storage.googleapis.com/ml-pipeline/release/0.1.2/kfp.tar.gz --upgrade
```

## run the example pipelines
```
python3 workflow1.py
```


Kubeflow pipelines UIにローカルのブラウザからGKE上のpod内のコンテナにアクセスできるようにポートフォワードの設定をしておく。
```
export NAMESPACE=kubeflow
kubectl port-forward -n ${NAMESPACE} $(kubectl get pods -n ${NAMESPACE} --selector=service=ambassador -o jsonpath='{.items[0].metadata.name}') 8080:80
```

Kubeflowは出てくるけど、kubeflowpipelinesのUIは出てこない

https://github.com/kubeflow/pipelines/wiki/Build-a-Pipeline


Cloudshellからは”web preview”末尾に"pipeline"を付けてアクセス
https://8080-dot-3326024-dot-devshell.appspot.com/pipeline/#/pipelines

入れた！

UIからgz.tarごとアップロード。

ExperimentsとRunを定義する画面に映るので設定してみる。

project: mlops-215604
working-dir: gs://bp-kubeflow-pipelines/

結果が走る。pipelineの今どこにいるかも可視化してくれている。

kubeflow pipelines　UI -> notebook.
ID:admin
pass:admin



Jupyternotebookに入れる。



# Draft
## 手順
これに従う
- https://github.com/amygdala/code-snippets/tree/master/ml/kubeflow-pipelines

GKEクラスタ立ち上げる

```
> gcloud beta container --project "mlops-215604" clusters create "kubeflow-pipelines" --zone "us-central1-a" --username "admin" --cluster-version "1.9.7-gke.11" --machine-type "custom-8-40960" --image-type "COS" --disk-type "pd-standard" --disk-size "100" --scopes "https://www.googleapis.com/auth/cloud-platform" --num-nodes "4" --enable-cloud-logging --enable-cloud-monitoring --no-enable-ip-alias --network "projects/mlops-215604/global/networks/default" --subnetwork "projects/mlops-215604/regions/us-central1/subnetworks/default" --addons HorizontalPodAutoscaling,HttpLoadBalancing,KubernetesDashboard --enable-autoupgrade --enable-autorepair
 ```

gcloudコマンドと紐付ける
```
> gcloud container clusters get-credentials kubeflow-pipelines --zone us-central1-a --project mlops-215604
Fetching cluster endpoint and auth data.
kubeconfig entry generated for kubeflow-pipelines.
> kubectl create clusterrolebinding ml-pipeline-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)
clusterrolebinding.rbac.authorization.k8s.io "ml-pipeline-admin-binding" created
> kubectl create clusterrolebinding sa-admin --clusterrole=cluster-admin --serviceaccount=kubeflow:pipeline-runner
clusterrolebinding.rbac.authorization.k8s.io "sa-admin" created
```


https://www.kubeflow.org/docs/guides/pipelines/deploy-pipelines-service/ に従ってkubeflow pipelineをデプロイする。

```
> PIPELINE_VERSION=0.1.2
# job を作る
> kubectl create -f https://storage.googleapis.com/ml-pipeline/release/$PIPELINE_VERSION/bootstrapper.yaml
clusterrole.rbac.authorization.k8s.io "mlpipeline-deploy-admin" created
clusterrolebinding.rbac.authorization.k8s.io "mlpipeline-admin-role-binding" created
job.batch "deploy-ml-pipeline-qqk9j" created
# 
> kubectl get job
NAME                       DESIRED   SUCCESSFUL   AGE
deploy-ml-pipeline-qqk9j   1         1            7m
> kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.7.240.1   <none>        443/TCP   18m
```

Kubeflow Pipelines のpythonSDKをインストールする 
https://github.com/kubeflow/pipelines/releases
```
> pip3 install https://storage.googles.com/ml-pipeline/release/0.1.2/kfp.tar.gz --upgrade
Installing collected packages: urllib3, six, certifi, python-dateutil, PyYAML, google-resumable-media, pytz, setuptools, chardet, idna, requests, protobuf, cachetools, pyasn1, rsa, pyasn1-modules, google-auth, googleapis-common-protos, google-api-core, google-cloud-core, google-cloud-storage, pycparser, cffi, asn1crypto, cryptography, PyJWT, adal, oauthlib, requests-oauthlib, websocket-client, kubernetes, kfp
  Running setup.py install for kfp ... done
Successfully installed PyJWT-1.6.4 PyYAML-3.13 adal-1.2.0 asn1crypto-0.24.0 cachetools-3.0.0 certifi-2018.10.15 cffi-1.11.5 chardet-3.0.4 cryptography-2.4.2 google-api-core-1.5.2 google-auth-1.6.1 google-cloud-core-0.28.1 google-cloud-storage-1.13.0 google-resumable-media-0.3.1 googleapis-common-protos-1.5.5 idna-2.7 kfp-0.1 kubernetes-8.0.0 oauthlib-2.1.0 protobuf-3.6.1 pyasn1-0.4.4 pyasn1-modules-0.2.2 pycparser-2.19 python-dateutil-2.7.5 pytz-2018.7 requests-2.20.1 requests-oauthlib-1.0.0 rsa-4.0 setuptools-40.6.2 six-1.11.0 urllib3-1.24.1 websocket-client-0.54.0
```

`~/code-snippets/ml/kubeflow-pipelines/samples/kubeflow-tf`へ移動する。
README.mdにある手順に従ってpipelinesのexampleを試してみる。

```
> cd ~/code-snippets/ml/kubeflow-pipelines/samples/kubeflow-tf
> python3 workflow1.py
> ls
README.md  workflow1.py  workflow1.py.tar.gz  workflow2.py
```
Kubeflow pipelines UIにローカルのブラウザからGKE上のpod内のコンテナにアクセスできるようにポートフォワードの設定をしておく。
```
export NAMESPACE=kubeflow
kubectl port-forward -n ${NAMESPACE} $(kubectl get pods -n ${NAMESPACE} --selector=service=ambassador -o jsonpath='{.items[0].metadata.name}') 8080:80
```

### jupyterのプロセスを立ち上げ
Spawnするイメージは下記を選択。
- gcr.io/kubeflow-images-public/tensorflow-1.10.1-notebook-cpu:v0.3.1


### 立ち上げたら
https://github.com/kubeflow/pipelines/issues/179
```
# Check for the Jupyter pod name `jupyter-<USER>` (Here user is 'admin')
kubectl get pods -n kubeflow

kubectl exec -it -n kubeflow jupyter-taketoshi-2ekazusa-40brainpad-2eco-2ejp　bash
jovyan@jupyter-admin:~$ vim .jupyter/jupyter_notebook_config.py
```
c.NotebookApp.allow_origin='*'を追記

再起動。
```
jovyan@jupyter-admin:~$ ps -auxw
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
jovyan       1  0.0  0.0   4528   820 ?        Ss   12:44   0:00 tini -- start-singleuser.sh --ip="0.0.0.0" --port=8888 --allow-root
jovyan       5  0.1  0.8 292128 62168 ?        Sl   12:44   0:01 /opt/conda/bin/python /opt/conda/bin/jupyterhub-singleuser --ip="0.0.0.0" --port=8888 --allow-root
jovyan      33  0.0  0.0  20316  4108 ?        Ss   12:52   0:00 bash
jovyan      41  0.0  0.0  36080  3300 ?        R+   12:53   0:00 ps -auxw

jovyan@jupyter-admin:~$ kill 1
jovyan@jupyter-admin:~$ command terminated with exit code 137

<USER>@CloudShell:~$ kubectl get pods  -n kubeflow |grep jupyter
jupyter-admin                                            1/1       Running   2          16m

export NAMESPACE=kubeflow
kubectl port-forward -n ${NAMESPACE} $(kubectl get pods -n ${NAMESPACE} --selector=service=ambassador -o jsonpath='{.items[0].metadata.name}') 8080:80
```






### 
Cloudshellからは”web preview”末尾に"pipeline"を付けるとkubeflow pipeliensのUIへ飛ぶことができる。

https://8080-dot-3326024-dot-devshell.appspot.com/pipeline/#/pipelines




## Fix jupyternotebook

https://github.com/kubeflow/pipelines/issues/179
```
# Check for the Jupyter pod name `jupyter-<USER>` (Here user is 'admin')
kubectl get pods -n kubeflow

kubectl exec -it -n kubeflow jupyter-taketoshi-2ekazusa-40brainpad-2eco-2ejp　bash
jovyan@jupyter-admin:~$ vim .jupyter/jupyter_notebook_config.py
```
c.NotebookApp.allow_origin='*'を追記

```
> jupyter nbextension list
Known nbextensions:
  config dir: /home/jovyan/.jupyter/nbconfig
    notebook section
      tfma_widget_js/extension  enabled 
      - Validating: problems found:
        - require?  X tfma_widget_js/extension
  config dir: /opt/conda/etc/jupyter/nbconfig
    notebook section
      jupyter-js-widgets/extension  enabled 
      - Validating: OK
  config dir: /usr/local/etc/jupyter/nbconfig
    notebook section
      jupyter-js-widgets/extension  enabled 
      - Validating: OK
```





## Juptyter notebookでTFMAを可視化
- https://www.kubeflow.org/docs/guides/components/jupyter/
browser previewでkubeflowにアクセス、jupyterhubへ。
- signin はGCPアカウント。
- cloudshellから~/~/code-snippets/ml/kubeflow-pipelines/components/dataflow/tfma/tfma_expers.ipynbをダウンロード
- jupyter上へ


https://github.com/tensorflow/model-analysis

```
kubectl -n ${NAMESPACE} describe pods jupyter-taketoshi-2ekazusa-40brainpad-2eco-2ejp


```

https://github.com/kubeflow/pipelines/issues/179
https://github.com/kubeflow/kubeflow/issues/1130





gs://bp-kubeflow-pipelines//workflow-1-wnrmr/




# サンプルがいくつかある
BasicとあるものはこのRepoにものがpipelineとしてpython dslとして記述されており、compileされてkubeflow piplinesにアップロードされた状態で置いてある。

スクショ：pipeliensトップ画面スクショ

このBuild a Pipeline(https://www.kubeflow.org/docs/guides/pipelines/build-pipeline/#compile-the-samples)にもあるようにcompileされたのだろう

https://github.com/kubeflow/pipelines/tree/master/samples/basic
- Condition:  https://github.com/kubeflow/pipelines/blob/master/samples/basic/condition.py
- Exit handler: https://github.com/kubeflow/pipelines/blob/master/samples/basic/exit_handler.py
- Immediate Value: https://github.com/kubeflow/pipelines/blob/master/samples/basic/immediate_value.py
- parallel_join: https://github.com/kubeflow/pipelines/blob/master/samples/basic/parallel_join.py
- sequential.py: https://github.com/kubeflow/pipelines/blob/master/samples/basic/sequential.py
- ML - TFX: Example pipeline that does classification with model analysis based on a public tax cab BigQuery dataset.
  - https://github.com/kubeflow/pipelines/tree/master/samples/tfx
  ->TFTとTFMAが一緒に楽しめる？しかもdataflow上で？
- ML - XGboost: https://github.com/kubeflow/pipelines/tree/master/samples/xgboost-spark
  - Google DataProc clusterを作る (Hadoop, spark)
  - TFあんまり関係なさそう

## TFX　Experimentsをやってみたいがエラーが出る。。。
- Experimentsのところにvalidation-modeとpreprocess-modeがありましてDefaultはlocalになっている->CloudにするとDataflowが動く。API許可しておこう
- predict-modeとanalyse-modeは？

とりあえず、localで走らせてみた。
- 

TODO:
- https://github.com/tensorflow/model-analysis/tree/master/examples/chicago_taxi を読む
- 4つのパートがある
  - TFDV
  - TFT
  - Train
  - TFMA
  -
   
   





