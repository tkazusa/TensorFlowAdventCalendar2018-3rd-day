# TensorFlowAdventCalendar2018
# 課題意識
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
 
kubectl create clusterrolebinding mlpipeline-deploy-admin --clusterrole=cluster-admin --user=taketoshi.kazusa@brainpad.cp.jp


kubectl create clusterrolebinding sa-admin --clusterrole=cluster-admin --serviceaccount=kubeflow:pipeline-runner
```




```
PIPELINE_VERSION=0.1.2
kubectl create -f https://storage.googleapis.com/ml-pipeline/release/$PIPELINE_VERSION/bootstrapper.yaml
```

Error from server (Forbidden): error when creating "https://storage.googleapis.com/ml-pipeline/release/0.1.2/bootstrapper.yaml": clusterroles.rbac.authorization.k8s.io "mlpipeline-deploy-admin" is forbidden: attempt to grant extra privileges: [
PolicyRule{Resources:["*"], APIGroups:["*"], Verbs:["*"]} PolicyRule{NonResourceURLs:["*"], Verbs:["*"]}] user=&{taketoshi.kazusa@brainpad.co.jp  [system:authenticated] map[user-assertion.cloud.google.com:[AK5xou8sPIQHUd9mRqB2uoNQ8lU/oJazV8n9Zw
3VBvnkPESAvreCnsGb7Ul2fc3OV51eZNAcpmSR6qdqrajJ9/JpW2okNHwwDovoIrnwWzARJliwi64q//mClGZTocBREpp/XmY+hNdmP7mgb0fDmy6CbkZHlnpMJLH1RiIYfxX/t8/lGYPYMuqCWGlcMXYyFXdyQ7jHoxVa9SwmxdyaGikq180wk6+4bhIdfVDdDSG/xzvYxMeMH6U=]]} ownerrules=[PolicyRule{Resourc
es:["selfsubjectaccessreviews" "selfsubjectrulesreviews"], APIGroups:["authorization.k8s.io"], Verbs:["create"]} PolicyRule{NonResourceURLs:["/api" "/api/*" "/apis" "/apis/*" "/healthz" "/swagger-2.0.0.pb-v1" "/swagger.json" "/swaggerapi" "/swa
ggerapi/*" "/version"], Verbs:["get"]}] ruleResolutionErrors=[]


```
kubectl get job
NAME                       DESIRED   SUCCESSFUL   AGE
deploy-ml-pipeline-rt8zf   1         1            14h

kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.7.240.1   <none>        443/TCP   14h
```
あれ？kubernetes本体以外のサービスたってなくね？ポートフォワード意味ある？



Kubeflow pipelines UIにローカルのブラウザからGKE上のpod内のコンテナにアクセスできるようにポートフォワードの設定をしておく。
```
export NAMESPACE=kubeflow
kubectl port-forward -n ${NAMESPACE} $(kubectl get pods -n ${NAMESPACE} --selector=service=ambassador -o jsonpath='{.items[0].metadata.name}') 8080:80
```


