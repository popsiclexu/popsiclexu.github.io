---
layout:     post
title:      "AirFlow定时调度执行Talend ETL任务"
subtitle:   "Using Airflow to Manage Talend ETL Jobs"
date:       2018-07-28 
author:     "Xu Zhenxue"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:
    - ETL
    - 任务调度
    - Airflow
---

> 滚动阅读全文  

## AirFlow调度平台简介

airflow 是一个编排、调度和监控工作流的平台，由Airbnb开源，现在在Apache Software Foundation 孵化。airflow将工作流编排为tasks组成的有向无环图（DAGs），调度器在一组workers上按照指定的依赖关系执行tasks。同时，airflow提供了丰富的命令行工具和简单易用的用户界面以便用户查看和操作，并且airflow提供了监控和报警系统

## AirFlow基础概念  
  
 Airflow主要是将工作流的相关信息定义到一个Python文件中，airflow根据文件中的定义信息执行工作流，在Airflow pipeline定义中，主要涉及两个类： <font color='red' style='background-color:#EAEDED;'>DAG</font>，<font color='red' style='background-color:#EAEDED;'>Operator</font>

<font color='red' style='background-color:#EAEDED;'>DAG</font> : 有向无环图，它将定义的任务按照依赖关系组织起来   
<font color='red' style='background-color:#EAEDED;'>Operator</font>：用来描述每个任务具体做的事，airflow内置了很多operator，如<font color='red' style='background-color:#EAEDED;'>BashOperator</font> 执行一个bash 命令，<font color='red' style='background-color:#EAEDED;'>PythonOperator</font> 调用任意的Python 函数，<font color='red' style='background-color:#EAEDED;'>EmailOperator</font> 用于发送邮件，<font color='red' style='background-color:#EAEDED;'>HTTPOperator</font> 用于发送HTTP请求， <font color='red' style='background-color:#EAEDED;'>SqlOperator</font> 用于执行SQL命令…同时，用户可以自定义Operator，这给用户提供了极大的便利性。  
通过DAG和Operator结合起来就可以构建复杂的工作流了  

## Talend简介

Talend是一个开源的ELT任务构建工具，可以通过简单拖拽的方式设计复杂的ETL任务并自动生成Java代码，设计完成后可以通过构建导出ETL任务java源码和可直接执行jar包及执行jar包的shell和bat文件。

接下来，我们通过Talend 桌面软件构建一个简单的ELT任务，然后通过airflow UI执行这个ETL任务

## Airflow调度ETL任务实例

### 1.通过Talend 设计、构建ETL任务

打开Talend设计如下简单的ETL任务，将一个CSV文件导入到数据库中：
![image](/img/etl.png)

然后，将这个ETL任务构建并导出，目录结构如下：

![image](/img/job.png)

我们使用在linux环境中执行shell文件的方式执行ETL任务。

### 2.定义Airflow pipeline
我们在airflow 指定的dags目录下新建python文件来定义airflow pipeline

代码如下:


```
import airflow
from datetime import datetime
from airflow.operators.bash_operator import BashOperator
from airflow.models import DAG
from airflow.utils.email import send_email

#shell命令执行.sh文件

job_commd = "/home/airflow/jobs/AirFlowTestJob_0.1/AirFlowTestJob/AirFlowTestJob_run.sh '/home/airflow/jobs/AirFlowTestJob_0.1/AirFlowTestJob/'"

#dag中的一些参数设置
default_args = {
    'owner': 'Airflow',
    'depends_on_past': False,
    'start_date': datetime(2018, 7, 23),    
    'email': ['VALID_EMAIL_ID'],
    'email_on_failure': True,
    'email_on_success': True,
    'retries':3,
}

dag = DAG('csv_to_data', default_args=default_args)

#定义ETL任务,多个任务时定义多个Operator
t1 = BashOperator(
    task_id='csv_to_database',
    bash_command=job_commd,
    dag=dag
)
```

### 3.调度执行ETL任务
完成以上工作，我们启动airflow webserver服务器后,便可以在airflow UI界面看到我们定义的ETl任务<font color='red' style='background-color:#EAEDED;'>csv_to_data</font>，可以进行相应操作。

![image](/img/airflow.png)

