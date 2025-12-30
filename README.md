# kubeflow_tutorial(kubeflow入门示例)
## 1. 菜单和仪表盘预览
![img.png](image/img.png)
## 2. 基本功能操作
### 2.1 创建Notebooks
- 点击Create a new book，下一步填名称，选择镜像，CPU，GPU和持久卷PV。点击Launch按钮创建
  ![img_1.png](image/img_1.png)
> 非常重要：Workspace Volume - Mount Path 目录不要更改，必须使用默认的目录，否则对notebook进行暂停，再重启会丢失所有数据！
> ![img_27.png](image/img_27.png)

- 创建成功后，点击connect连接server，开始脚本编写之旅
  ![img_5.png](image/img_5.png)
  ![img_6.png](image/img_6.png)
  首先，确认kfp版本，再安装kfp-kubernetes
```text
import kfp
print(kfp.__version__)
!pip install kfp-kubernetes
```
安装依赖包后，正式开始示例编写与演示
### 2.2 编写演示示例
#### 2.2.1 编写kubeflow Pipeline定义，创建和运行Pipeline
这里简单的使用“勾股定理运算”来演示
##### 目的
- 集群能正常创建和运行Pipeline
- Pipeline Graph中，查看Pod运行数据和日志
##### 步骤：创建Pipeline
##### step1 - pipeline定义
```text
from kfp import dsl

@dsl.component(base_image='python:3.9')
def square(x: float) -> float:
    print(f'square: {x}')
    return x ** 2

@dsl.component(base_image='python:3.9')
def add(x: float, y: float) -> float:
    print(f'add number: x={x}, y={y}')
    return x + y

@dsl.component(base_image='python:3.9')
def square_root(x: float) -> float:
    print(f'square_root: {x}')
    return x ** .5
    
@dsl.pipeline
def pythagorean(a: float, b: float) -> float:
    a_sq_task = square(x=a)
    b_sq_task = square(x=b)
    sum_task = add(x=a_sq_task.output, y=b_sq_task.output)
    root_task = square_root(x=sum_task.output)
    return root_task.output
```
##### step2 - 编译Pipeline
```text
from kfp import compiler

compiler.Compiler().compile(pythagorean, 'demo-pythagorean-pipeline.yaml')
```
##### step3 - 创建Pipeline
```text
from kfp.client import Client

client = Client(host='http://ml-pipeline.kubeflow:8888',
                namespace="kubeflow-user-example-com",
                existing_token="eyJhbGciOiJSUzI1NiIsImtpZCI6ImQteE6fSS1Rb1d6TzhRYlo5QWZlbnVSbm5mM3ZfUGx3eU1uN0Q3ZHI5c00ifQ.eyJhdWQiOlsicGlwZWxpbmVzLmt1YmVmbG93Lm9yZyJdLCJleHAiOjE3OTgzNDk0MTAsImlhdCI6MTc2NjgxMzQxMCwiaXNzIjoiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiLCJqdGkiOiJkMWUzZmVkYi1hMDY4LTQxYWYtYWU0NS00YTlhYjc1MGY0MzIiLCJrdWJlcm5ldGVzLmlvIjp7Im5hbWVzcGFjZSI6Imt1YmVmbG93Iiwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImtmcC1hcGktdXNlciIsInVpZCI6IjA3OTA1ZWQxLTU3ODYtNDI3Yy1hOGRmLWVlM2FiYTNmYjgyZCJ9fSwibmJmIjoxNzY2ODEzNDEwLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZWZsb3c6a2ZwLWFwaS11c2VyIn0.KP7fiYHAwoLBJbvph6FfyQqbzrwusJaB3C2Tds9wPgokCwZNo8CPASUlWMovQxhto2lgmkXXHiyEuhhDasrwJ034OsjB22W6PW3Pc50UiG6jXSdNX1hO9WQbKwfHUHhZPI9CwjUt4QhVTWCVDS1Q3oauQ4izjsUQHZ47LbM6VsJHIE6Xm83d3YRVdltQH3DdkoRIiD4eDsspotvnyeWcSbSUeIK148gWLR96K4kZAmtfvT-oI4Py3UoL4aLfKXYq_yEMyf6FW54N4-fSu9kVVNf3IaLUz067zqXf3mn1jBUMukLTkIyZjG80KEAHX62IwbJbjyPaOA0jOwaBRU7v3g",
                verify_ssl=False)

pipeline_info = client.upload_pipeline(
    pipeline_package_path='demo-pythagorean-pipeline.yaml',
    pipeline_name='demo-pythagorean-pipeline',
    description='This is a simple demo',
    namespace='kubeflow-user-example-com'
)
print(f"Pipeline uploaded! ID: {pipeline_info}")
```
执行成功后，在菜单栏Pipeline中可以看到相应的行
![img_9.png](image/img_9.png)
代码执行效果图：
![img_10.png](image/img_10.png)

> !!!请注意namespace和token
> namespace不填，则创建公有的Pipeline。反之，创建私有的Pipeline填写自己空间名
> token：目前让管理员通过kubectl 命令进行创建

- 获取namespace
  ![img_7.png](image/img_7.png)
  ![img_8.png](image/img_8.png)
- 创建token

新建kfp-service-account.yaml文件：
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kfp-api-user
  namespace: kubeflow
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kfp-api-user-binding
subjects:
  - kind: ServiceAccount
    name: kfp-api-user
    namespace: kubeflow
roleRef:
  kind: ClusterRole
  name: kubeflow-pipelines-edit
  apiGroup: rbac.authorization.k8s.io
```
执行命令：
```shell
kubectl -n kubeflow create token kfp-api-user \
  --audience=pipelines.kubeflow.org \
  --duration=24h
```

##### step4 运行Pipeline
```text
from kfp.client import Client

client = Client(host='http://ml-pipeline.kubeflow:8888',
                namespace="kubeflow-user-example-com",
                existing_token="ACCESS_TOKEN",
                verify_ssl=False)
run = client.create_run_from_pipeline_package(
    'demo-pythagorean-pipeline.yaml',
    arguments={
        'a': 3,
        'b': 4
    },
    enable_caching=False,
)
```
效果图：
![img_11.png](image/img_11.png)
![img_12.png](image/img_12.png)
![img_13.png](image/img_13.png)


#### 2.2.2 读取挂载卷文件数据，写入处理后的数据至挂载卷
这里使用“对整型数据进行降序排序”来演示
完整代码：请查看src/sorting/sorting-pipeline.ipynb
##### 目的
- 验证能正确在PVC卷中读取和写入数据
##### 步骤
1. 创建持久卷
     ![img_14.png](image/img_14.png)
     ![img_15.png](image/img_15.png)
     ![img_16.png](image/img_16.png)
2. 打开持久卷，在根目录新建input_numbers.csv
     ![img_17.png](image/img_17.png)![img_18.png](image/img_18.png)![img_19.png](image/img_19.png)

input_numbers.csv内容：
```text
1
3
2
4
5
10
9
7
8
6
```
3. 指定工作流Task的Pod挂载相应的持久卷
```text
# --- 定义 Pipeline 工作流 ---
@dsl.pipeline(
    name="csv-sort-and-save-to-pvc",
    description="从 PVC 读取数据，排序后写回 PVC"
)
def demo_sort_pipeline(
    raw_input_path: str = "/mnt/shared/input_numbers.csv",
    pvc_dir: str = "/mnt/shared/output",
    final_name: str = "sorted_numbers.csv"
):
    # 第一步任务
    task1 = sort_csv_component(input_file_path=raw_input_path)
    
    # 第二步任务（传入第一步的输出结果）
    task2 = save_and_print_component(
        sorted_data_input=task1.outputs['sorted_output'],
        pvc_output_directory=pvc_dir,
        file_name=final_name
    )

    # --- 为所有步骤挂载同一个 PVC ---
    # 假设你已有一个名为 'data-pvc' 的 PVC
    tasks = [task1, task2]
    for task in tasks:
        kubernetes.mount_pvc(
            task,
            pvc_name="aidatasource",
            mount_path="/mnt/shared" # 容器内访问该 PVC 的根目录
        )
```
> 提示：核心使用kubernetes指定挂载的pvc名称和路径
>
>kubernetes.mount_pvc(
>task,
>pvc_name="aidatasource",
>mount_path="/mnt/shared" # 容器内访问该 PVC 的根目录
>)
>
4. 最后编译和运行Pipeline 

效果图：
     ![img_20.png](image/img_20.png)![img_21.png](image/img_21.png)![img_22.png](image/img_22.png)
     最后会在PVC的output目录下,写入排序好的sorted_numbers.csv文件
     ![img_23.png](image/img_23.png)![img_24.png](image/img_24.png)


#### 2.2.3 为Pipeline Pod指定CPU和GPU资源 
##### 目的
- 验证正确的为Pipeline的每个步骤限定CPU和GPU资源，并且k8s正确的调度
##### 步骤 - 代码与演示
完整代码：请查看src/cpu_gpu_setting_node_selector/cgpu_setting_node_selector.ipynb
1. 在Pipeline定义每个Pod相应CPU和GPU资源 
```jupyterpython
# 在Pipeline定义资源
@dsl.pipeline(
    name="cgpu_setting_node_selector_pipeline",
    description="Full ML pipeline with Dataset and Model artifacts."
)
def cgpu_setting_node_selector_pipeline():
    # 数据加载
    data_task = load_data()
    data_task.set_cpu_limit('1').set_memory_limit('2G')

    # 模型训练
    train_task = train_model(
        input_dataset=data_task.outputs["dataset_output"]
    )
    # 为train_task配置资源
    # CPU资源
    train_task.set_cpu_limit('1').set_memory_limit('4G')
    # GPU资源
    train_task.set_accelerator_limit(2)
    train_task.add_node_selector_constraint('nvidia.com/gpu')
    kubernetes.add_node_selector(
        train_task,
        label_key='nvidia.com/gpu.product',
        label_value='NVIDIA-GeForce-RTX-3090',
    )
```
> 注意： 
> 添加kubernetes.add_node_selector方法，使用标签匹配相应的GPU，才能让容器正常起来。其他方法参考文档
> [Mastering Resource Allocation in MLOps: A Guide to Optimizing GPU Utilization in Kubeflow Pipelines](https://medium.com/@amirianfar/mastering-resource-allocation-in-mlops-a-guide-to-optimizing-gpu-utilization-in-kubeflow-pipelines-99734c8eb4a2) 
> 

2. 验证资源是否如期分配的方法
代码打印CPU和GPU信息
```jupyterpython

import os
import subprocess

# 打印 CPU/内存
with open('/sys/fs/cgroup/cpu.max', 'r') as f:
    print(f"CPU/GPU Quota Info: {f.read().strip()}")
with open('/sys/fs/cgroup/memory.max', 'r') as f:
    print(f"Memory Limit Info: {f.read().strip()}")

# 打印 GPU
try:
    res = subprocess.check_output(["nvidia-smi", "-L"]).decode("utf-8")
    print(f"GPU:\n{res}")
except:
    print("GPU limit doesn't set .")

```

你可能会用到的kubectl命令
```shell
kubectl get nodes -o custom-columns=\
,\
NVIDIA-GPU:.status.allocatable.nvid> "NAME:.metadata.name,\
> NVIDIA-GPU:.status.allocatable.nvidia\.com/gpu,\
> AMD-GPU:.status.allocatable.amd\.com/gpu,\
> INTEL-GPU:.status.allocatable.intel\.com/gpu,\
> ASCEND-NPU:.status.allocatable.huawei\.com/Ascend910"
kubectl describe node | grep nvidia.com/gpu
kubectl get nodes -L nvidia.com/gpu.product
```
![img.png](image/img_28.png)
效果图：
![img_1.png](image/img_29.png)
![img_2.png](image/img_30.png)
![img_3.png](image/img_31.png)


#### 2.2.4  minio存储训练模型
##### 目的
- 验证能正常使用minio保存和读取文件 
##### 步骤 - 代码与演示
完整代码：请查看src/minio/minio-complete.ipynb
1. 加载、构建、训练模型.
2. 将模型save本地，上传至远程存储桶
3. 读取存储桶下的模型，并发布至KServe
```shell
# Part2 将模型save本地，上传至远程存储桶
from tensorflow import keras
import os
# 保存模型，检查并创建目录 (exist_ok=True 表示如果文件夹已存在则不报错)
save_path = "models/detect-digits.keras"
os.makedirs(os.path.dirname(save_path), exist_ok=True)
keras.models.save_model(model, save_path)

# 上传到远程桶
minio_client = Minio(
    "minio-service.kubeflow:9000",
    access_key="minio",
    secret_key="minio123",
    secure=False
)
bucket_name = "mlpipeline"
local_path="models/detect-digits.keras",
remote_path="models"
minio_client.fput_object(bucket_name, remote_path, local_path)
# 成功上传文件: models/detect-digits.keras -> models/detect-digits.keras
# 最终路径 - s3://mlpipeline/models/detect-digits.keras

# 读取并发布model至KServe
!pip install kserve==0.11.0 kubernetes
from types import SimpleNamespace
deploy_model_to_kserve(
    model=SimpleNamespace(uri="s3://mlpipeline/models/detect-digits.keras"),
    # model=SimpleNamespace(uri="models/detect-digits.keras"),
    service_name="detect-digits",
    namespace="kubeflow-user-example-com"
)

```
> 参考文档：[MLOps Workflow: Recognizing Digits with Kubeflow](https://github.com/flopach/digits-recognizer-kubeflow/blob/master/readme.md)

效果图：
![img.png](image/img_32.png)

#### 2.2.5 发布Model至KServe
##### 目的
- 验证可部署Model至KServe
##### 步骤 - 代码与演示
完整代码：请查看src/kserver/kserver_pipeline.ipynb
1. 加载数据并保存为 Dataset Artifact
2. 训练模型并输出 Model Artifact
3. 部署model to kserve
```text
# 3. 部署model to kserve
@dsl.component(
    base_image="python:3.10",
    packages_to_install=["kserve==0.11.0", "kubernetes"]
)
def deploy_model_to_kserve(
    model: Input[Model],
    service_name: str,
    namespace: str
):
    from kserve import KServeClient
    from kserve import V1beta1InferenceService
    from kserve import V1beta1InferenceServiceSpec
    from kserve import V1beta1PredictorSpec
    from kserve import V1beta1SKLearnSpec
    from kubernetes import client as k8s_client

    # 手动配置 Token 和 Host
    configuration = k8s_client.Configuration()
    configuration.host = "http://kserve-controller-manager-service.kubeflow:8443" # 或者是 API Server 地址
    configuration.verify_ssl = False # 生产环境建议开启并配置证书
    configuration.api_key = {"authorization": "Bearer " + "ACCESS_TOKEN"}
    # 创建 KServe 客户端
    kserve_client = KServeClient(client_configuration=configuration)

    # 定义 InferenceService 结构
    # 注意：对于 Scikit-Learn 模型，KServe 需要存储路径包含 joblib/pickle 文件
    isvc = V1beta1InferenceService(
        api_version="serving.kserve.io/v1beta1",
        kind="InferenceService",
        metadata=k8s_client.V1ObjectMeta(
            name=service_name,
            namespace=namespace,
            annotations={'sidecar.istio.io/inject': 'false'}
        ),
        spec=V1beta1InferenceServiceSpec(
            predictor=V1beta1PredictorSpec(
                sklearn=V1beta1SKLearnSpec(
                    # model.uri 会自动转换为 s3:// 或 gs:// 路径
                    storage_uri=model.uri
                )
            )
        )
    )

    # 执行部署 (如果已存在则 patch，不存在则 create)
    try:
        kserve_client.create(isvc)
        print(f"Service {service_name} created.")
    except:
        kserve_client.patch(service_name, isvc)
        print(f"Service {service_name} updated.")

```
> 注意：<br/>
>  创建 KServe 客户端
>  kserve_client = KServeClient(client_configuration=configuration) <br/> 其中:
>  KServer Api Host:"http://kserve-controller-manager-service.kubeflow:8443" <br/>
>  Token：参照2.2.1案例-获取token

编写Pipeline定义
```text
@dsl.pipeline(
    name="kserver_pipeline",
    description="Full ML pipeline with Dataset and Model artifacts."
)
def kserver_pipeline():
    # 数据加载
    data_task = load_data()
    data_task.set_cpu_limit('1').set_memory_limit('2G')

    # 模型训练
    train_task = train_model(
        input_dataset=data_task.outputs["dataset_output"]
    )

    # GPU 资源配置
    train_task.set_cpu_limit('1')
    train_task.set_memory_limit('4G')
    train_task.set_accelerator_limit(1)
    train_task.add_node_selector_constraint('nvidia.com/gpu')
    # train_task.set_accelerator_type('NVIDIA-GeForce-RTX-3090')
    # 环境变量
    # train_task.set_env_variable(name='NVIDIA_VISIBLE_DEVICES', value='all')
    # train_task.set_env_variable(name='NVIDIA_DRIVER_CAPABILITIES', value='compute,utility')
    deploy_task = deploy_model_to_kserve(
        model=train_task.outputs["model_output"], # 这里的传递是关键
        service_name="iris-svc",
        namespace="kubeflow-user-example-com"
    )

```
> 提示几个注意点：
> - 保存模型时，指定的Artifact路径, 不要省略后缀.joblib
> > joblib.dump(clf, os.path.join(model_output.path, "model.joblib"))
> - 部署时入参，要填写model_output
>> deploy_task = deploy_model_to_kserve(
model=train_task.outputs["model_output"], # 这里的传递是关键 <br/>
service_name="iris-svc",
namespace="kubeflow-user-example-com"
)
>
效果图：
![img_2.png](image/img_2.png)
![img_3.png](image/img_3.png)
![img_4.png](image/img_4.png)
在KServe EndPoints菜单可以看到我们发布的模型
![img_26.png](image/img_26.png)

## 参考文档
- [kserve client api and sample](https://kserve.github.io/website/docs/reference/controlplane-client/controlplane-client-sdk)
- [kubeflow kfp sdk api](https://kubeflow-pipelines.readthedocs.io/en/latest/source/dsl.html?h=kubernates#kfp.dsl.PipelineTask)
- [digits-recognizer-kubeflow](https://github.com/flopach/digits-recognizer-kubeflow/blob/master/readme.md)