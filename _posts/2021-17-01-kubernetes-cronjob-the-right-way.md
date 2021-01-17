---
title:  "Kubernetes: CronJob, The Right Way"
date: 2021-01-17
categories:
  - blog
tags:
  - kubernetes
  - cronjob
---

Last week, we had an issue in production (as expected, it was a local holiday). One of the many jobs we had in our kubernetes cluster had not run for over a day. After some basic troubleshooting, we came to know that one run of the job got stuck in the **Running** state, and had not terminated. This prevented the subsequent schedules from running because of the concurrencyPolicy.

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: location-publisher
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      ...
```

As you can see from the above yaml snippet, the job is supposed to run every 5 minutes, and since we did not want multiple instances of the job to run in case once of them is delayed, we had set the _concurrencyPolicy_ to _Forbid_. However, what we did not anticipate is an instance of the job running forever and getting stuck.

To protect against this, we need two more configuration values: activeDeadlineSeconds and startingDeadlineSeconds.

**activeDeadlineSeconds** is used to tell k8s when to terminate a pod running as part of a job. You can read more about this [here](https://kubernetes.io/docs/concepts/workloads/controllers/job/#job-termination-and-cleanup). Its now clear to us that we should never create cronjbos without this setting. In our case, we set it to 300 seconds.

**startingDeadlineSeconds** is used to tell k8s when to give up trying to schedule new jobs. By default, it checks since beginning and stops once 100 missed schedules are found. Since we did not want to stop trying ever, we set this value to 3600 seconds. Since our schedules are for 5 minutes, missed schedules will never cross 100. More about this [here](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#cron-job-limitations).

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: location-publisher
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 3600
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      activeDeadlineSeconds: 300
      ...
```

One other important thing to note, if you are using any logging libraries or Azure Application Insights, is to properly flush them before exiting. This will ensure all logs gets pushed to ApplicationInsights before the job terminates.

Here's an example of how the nodejs _index.ts_ looks for the job:

```typescript
/* eslint-disable no-process-exit */
import Settings from "./appsettings.json";
import * as AppInsights from "applicationinsights";
import { v4 as uuidv4 } from "uuid";

(async () => {
    try {
        //Initialize and start Application Insights
        appInsights
            .setup(settings.applicationInsights.InstrumentationKey)
            .setSendLiveMetrics(true);

        appInsights.defaultClient.context.tags[appInsights.defaultClient.context.keys.cloudRole] =  "LOCATION-PUBLISHER";

        appInsights.start();
        AppInsights.appInsightConfig.defaultClient.context.tags["ai.operation.id"] = uuidv4();

        // DO YOUR WORK
        // ...
        // ...

        appInsights.defaultClient.flush({ callback: () => process.exit() });
    } catch (err) {
        // Log error
        appInsights.defaultClient.flush({ callback: () => process.exit(1) });
    }
})();

```

Hope that helped!
