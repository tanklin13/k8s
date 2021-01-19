# 工作负载（4）-Job和Cronjob

Job：批处理任务

Cronjob：定时批处理



创建Cronjob

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  creationTimestamp: null
  generateName: reload-prometheus
  namespace: 【pg-allen-job】
  name: reload-prometheus
  labels:
    app: reload-prometheus
spec:
  concurrencyPolicy: Replace    # 并发策略。默认值Replace。
  failedJobsHistoryLimit: 2      # 保留的失败记录数。为了减少资源消耗，不建议配置过大。
  schedule: '*/3 * * * *'   # 定时任务的crontab时间配置，按 <minute> <hour> <day of month> <month> <day of week>格式填写。
  startingDeadlineSeconds: 60   # job启动超时时间。
  successfulJobsHistoryLimit: 5   # 保留的成功记录数。为了减少资源消耗，不建议配置过大。
  suspend: false      # 若job未执行完毕，是否挂起后续job。默认值false。
  jobTemplate:
    spec:
      backoffLimit: 3    # 重启次数限制，重启达到次数后，则不再重启。
      template:
        metadata:
          labels:
            app: reload-prometheus
        spec:
          containers:
          - command:        # 定时执行的命令。需要确保命令在容器内能够正常执行。
            - curl
            args:
            - -X
            - PUT
            - --connect-timeout
            - "10"
            - -m
            - "60"
            - https://xxx.xxx.com/paas/ops/reload-prometheus
            image: 172.30.10.185:15000/common/curl-alpine:3.8
            imagePullPolicy: IfNotPresent
            name: reload-prometheus
            resources:
              limits:
                cpu: 800m
                memory: 100Mi
              requests:
                cpu: 100m
                memory: 10Mi
          dnsPolicy: Default
          hostNetwork: true
          restartPolicy: Never   # 重启策略。
```



等待数分钟，即可查看状态：

```
# kubectl get cronjob -n pg-allen-job
NAME                SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
reload-prometheus   */3 * * * *   False     0        39s             7m13s

# kubectl get job -n pg-allen-job
NAME                           COMPLETIONS   DURATION   AGE
reload-prometheus-1586920500   1/1           10s        40s

# kubectl get po -n pg-allen-job
NAME                                 READY   STATUS      RESTARTS   AGE
reload-prometheus-1586920500-2qg8b   0/1     Completed   0          48s

# kubectl logs -f reload-prometheus-1586920500-2qg8b -n pg-allen-job
... ...
```



研究完毕。删除创建的cronjob：

```
# kubectl delete cronjob reload-prometheus -n pg-allen-job --cascade=true
cronjob.batch "reload-prometheus" deleted
```

删除cronjob时，若添加``--cascade=true``，则会级联删除job和pod。



扩展阅读：

* CronJob：https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
* Running Automated Tasks with a CronJob：https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/



Next：[08-Service](08-Service.md)

