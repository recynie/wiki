---
title: MLOps模型部署与生命周期管理
date: 2026-04-13
description: 机器学习模型部署、监控、持续训练与MLOps最佳实践
tags:
  - machine-learning
  - mlops
  - deployment
  - model-serving
draft: false
permalink:
---

# MLOps模型部署与生命周期管理

MLOps是将DevOps原则应用于机器学习系统的实践，实现模型的自动化构建、测试和部署。

## ML系统特殊挑战

| 挑战 | 说明 | 解决方案 |
|------|------|----------|
| 数据漂移 | 输入数据分布随时间变化 | 监控、数据验证 |
| 模型衰减 | 性能随时间下降 | 持续训练、重新部署 |
| 可解释性 | 黑盒模型难以调试 | 模型分析、特征重要性 |
| 版本控制 | 代码+数据+模型多版本 | 实验追踪、工件管理 |

## ML工作流

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│    Data     │───►│   Train     │───►│   Evaluate  │
│  Collection │    │   Model     │    │   Model     │
└─────────────┘    └─────────────┘    └──────┬──────┘
                                             │
                                             ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Monitor    │◄───│   Deploy    │◄───│   Register  │
│  Model      │    │   Model     │    │   Model     │
└─────────────┘    └─────────────┘    └─────────────┘
```

## 模型训练与追踪

### MLflow追踪

```python
import mlflow
from mlflow.tracking import MlflowClient

# 设置追踪服务器
mlflow.set_tracking_uri("http://localhost:5000")
mlflow.set_experiment("recommendation-model")

with mlflow.start_run(run_name="v2-baseline"):
    # 记录参数
    mlflow.log_param("learning_rate", 0.01)
    mlflow.log_param("n_estimators", 100)
    mlflow.log_param("max_depth", 5)
    
    # 训练模型
    model = train_model(X_train, y_train)
    
    # 评估并记录指标
    predictions = model.predict(X_test)
    metrics = evaluate_model(y_test, predictions)
    
    for metric_name, value in metrics.items():
        mlflow.log_metric(metric_name, value)
    
    # 记录模型
    mlflow.sklearn.log_model(model, "model")
    
    # 记录附加信息
    mlflow.set_tag("version", "v2")
    mlflow.set_tag("environment", "production")
```

### 自动化训练Pipeline

```python
from prefect import task, flow
import mlflow

@task
def load_data():
    """加载训练数据"""
    df = pd.read_csv("s3://bucket/training-data.csv")
    return df

@task
def preprocess(df):
    """数据预处理"""
    X = df.drop("target", axis=1)
    y = df["target"]
    return train_test_split(X, y, test_size=0.2)

@task
def train_model(X_train, y_train, params):
    """训练模型"""
    with mlflow.start_run():
        mlflow.log_params(params)
        
        model = XGBClassifier(**params)
        model.fit(X_train, y_train)
        
        mlflow.sklearn.log_model(model, "model")
        return model

@task
def evaluate_model(model, X_test, y_test):
    """评估模型"""
    metrics = {
        "accuracy": accuracy_score(y_test, model.predict(X_test)),
        "f1": f1_score(y_test, model.predict(X_test), average="weighted")
    }
    mlflow.log_metrics(metrics)
    return metrics

@flow
def training_pipeline():
    params = {
        "learning_rate": 0.01,
        "n_estimators": 100,
        "max_depth": 5
    }
    
    df = load_data()
    X_train, X_test, y_train, y_test = preprocess(df)
    model = train_model(X_train, y_train, params)
    metrics = evaluate_model(model, X_test, y_test)
    
    # 条件注册：性能达标才注册
    if metrics["f1"] > 0.9:
        register_model(model, "production")
    
    return metrics
```

## 模型服务

### REST API服务

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import mlflow
import numpy as np

app = FastAPI()

# 加载模型
model = mlflow.sklearn.load_model("models:/recommendation-model/production")

class PredictionRequest(BaseModel):
    user_id: int
    item_features: list[float]

class PredictionResponse(BaseModel):
    prediction: float
    probability: float

@app.post("/predict", response_model=PredictionResponse)
async def predict(request: PredictionRequest):
    try:
        features = np.array(request.item_features).reshape(1, -1)
        
        prediction = model.predict(features)[0]
        probability = model.predict_proba(features)[0].max()
        
        return PredictionResponse(
            prediction=float(prediction),
            probability=float(probability)
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# 健康检查
@app.get("/health")
async def health_check():
    return {"status": "healthy", "model_loaded": model is not None}
```

### 批量推理

```python
from pyspark.ml.pipeline import PipelineModel
from pyspark.sql import SparkSession
from datetime import datetime

def batch_inference():
    spark = SparkSession.builder \
        .appName("recommendation-batch-inference") \
        .config("spark.mlflow.tracking_uri", "http://localhost:5000") \
        .getOrCreate()
    
    # 加载模型
    model = PipelineModel.load("s3://models/batch-model")
    
    # 读取待预测数据
    predictions_df = model.transform(
        spark.read.parquet("s3://data/features/2026-04-13/")
    )
    
    # 写出结果
    output_path = f"s3://predictions/{datetime.now().strftime('%Y%m%d%H%M%S')}/"
    predictions_df.write \
        .mode("overwrite") \
        .parquet(output_path)
    
    spark.stop()

# Airflow调度
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

with DAG("batch_inference", start_date=datetime(2026, 4, 1), 
         schedule_interval="0 */6 * * *") as dag:
    
    inference_task = PythonOperator(
        task_id="run_batch_inference",
        python_callable=batch_inference
    )
```

## 模型监控

### 监控指标

```python
from prometheus_client import Counter, Histogram, Gauge
import numpy as np

# 定义指标
prediction_latency = Histogram(
    'prediction_latency_seconds',
    'Time spent processing prediction',
    ['model_version']
)

prediction_errors = Counter(
    'prediction_errors_total',
    'Total prediction errors',
    ['error_type', 'model_version']
)

data_drift = Gauge(
    'feature_drift_score',
    'Feature distribution drift score',
    ['feature_name']
)

model_accuracy = Gauge(
    'model_accuracy_current',
    'Current model accuracy',
    ['model_version', 'metric']
)

def monitor_predictions(y_true, y_pred, feature_vector, model_version):
    # 延迟监控
    end = time.time()
    prediction_latency.labels(model_version=model_version).observe(end - start)
    
    # 数据漂移检测
    for i, (mean, std) in enumerate(zip(
        np.mean(feature_vector, axis=0),
        np.std(feature_vector, axis=0)
    )):
        # 与训练数据统计比较
        drift_score = abs(mean - training_means[i]) / (training_stds[i] + 1e-9)
        data_drift.labels(feature_name=f"feature_{i}").set(drift_score)
```

### 漂移检测

```python
from scipy import stats

class DataDriftDetector:
    def __init__(self, reference_data: np.ndarray, threshold: float = 0.05):
        self.reference_data = reference_data
        self.threshold = threshold
        self.reference_mean = np.mean(reference_data, axis=0)
        self.reference_std = np.std(reference_data, axis=0) + 1e-9
    
    def detect(self, current_data: np.ndarray) -> dict:
        current_mean = np.mean(current_data, axis=0)
        current_std = np.std(current_data, axis=0) + 1e-9
        
        # Kolmogorov-Smirnov检验
        ks_scores = []
        for i in range(current_data.shape[1]):
            _, p_value = stats.ks_2samp(
                self.reference_data[:, i],
                current_data[:, i]
            )
            ks_scores.append(p_value)
        
        # Population Stability Index (PSI)
        psi = self._calculate_psi(
            self.reference_mean, self.reference_std,
            current_mean, current_std
        )
        
        drift_detected = any(p < self.threshold for p in ks_scores) or psi > 0.2
        
        return {
            "drift_detected": drift_detected,
            "ks_scores": ks_scores,
            "psi": psi,
            "feature_means": current_mean.tolist()
        }
    
    def _calculate_psi(self, ref_mean, ref_std, curr_mean, curr_std):
        """Population Stability Index"""
        ref_dist = stats.norm(ref_mean, ref_std)
        curr_dist = stats.norm(curr_mean, curr_std)
        
        psi_values = []
        for bucket in np.linspace(-3, 3, 10):
            ref_prob = ref_dist.pdf(bucket).mean()
            curr_prob = curr_dist.pdf(bucket).mean()
            
            if ref_prob > 0 and curr_prob > 0:
                psi_values.append(
                    (curr_prob - ref_prob) * np.log(curr_prob / ref_prob)
                )
        
        return sum(psi_values)
```

## 持续训练（Continuous Training）

```yaml
# Kubeflow Pipeline配置
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: ml-training-pipeline
spec:
  tasks:
    - name: data-validation
      taskRef:
        name: data-validator
      params:
        - name: data_path
          value: "s3://data/training.csv"
        - name: validation_config
          value: "schema.yaml"
    
    - name: train-model
      taskRef:
        name: model-trainer
      params:
        - name: training_data
          value: "$(tasks.data-validation.results.validated_data)"
        - name: hyperparameters
          value: "params.yaml"
      runAfter:
        - data-validation
    
    - name: evaluate-model
      taskRef:
        name: model-evaluator
      params:
        - name: model_path
          value: "$(tasks.train-model.results.model_path)"
        - name: test_data
          value: "s3://data/test.csv"
        - name: threshold
          value: "0.95"
      runAfter:
        - train-model
    
    - name: register-model
      taskRef:
        name: model-registrar
      params:
        - name: model_path
          value: "$(tasks.train-model.results.model_path)"
        - name: metrics
          value: "$(tasks.evaluate-model.results.metrics)"
      runAfter:
        - evaluate-model
```

## 模型注册表

```python
import mlflow
from mlflow.tracking import MlflowClient

client = MlflowClient()

def register_production_model(run_id: str, model_name: str):
    """注册模型并设置为生产"""
    
    # 创建模型版本
    model_uri = f"runs:/{run_id}/model"
    mv = mlflow.register_model(model_uri, model_name)
    
    # 获取最新版本
    latest_version = mv.version
    
    # 验证模型性能
    metrics = client.get_run(run_id).data.metrics
    if metrics["f1_score"] < 0.9:
        print(f"Model performance {metrics['f1_score']} below threshold")
        return None
    
    # 设置阶段
    client.transition_model_version_stage(
        name=model_name,
        version=latest_version,
        stage="Production"
    )
    
    # 更新描述
    client.update_model_version(
        name=model_name,
        version=latest_version,
        description=f"Production model version {latest_version}"
    )
    
    return latest_version
```

## 参考

[^1]: Google's ML Engineering Best Practices
[^2]: MLflow Documentation
[^3]: Kubeflow Documentation

