# How it started
It started with a [Tweet](https://twitter.com/roidelapluie/status/1400535315081801752) from Julien Pivotto (@roidelapluie): 
He created a setup that enables you to graph your Twitter followers with Prometheus and Grafana via json_exporter. This is placed on [Github](https://github.com/roidelapluie/graph-twitter-followers).

![A Grafana graph showing the number of followers](https://raw.githubusercontent.com/wiardvanrij/civo-images/main/grafana.png)

I wanted to extend this creativity by automating my Twitter banner so that it would display the Grafana panel. It also should update this automatically every n-period. This way I have a little gamification for my followers. If one would follow me, the graph should go up at the next update interval. Seems pretty neat!

# Why Civo?
Since I want to automate this, and not run it on my computer: I needed a solution foundation. So I've made a few requirements for myself:

* It runs as cloud-native
* Embraces containerization
* Ease of setup 
* Fair in pricing
* Possibilities to extend this further later on

This brought me to Kubernetes as an orchestration platform. Now you could wonder; is this not over-engineered? Then I would say yea, maybe. This whole setup is over-engineered and not the most solid solution for our use case. However, it is so much fun!

Yet again, what Civo delivers as a Kubernetes platform with k3s, makes a lot of sense. In contrast to different offerings such as EKS, AKS, and GKE, this is much simpler and easier on your wallet. Now there are solutions that remove some technical management overhead such as Google's Autopilot clusters or AWS ECS and Fargate. However, we then also lose a lot of control. 

I strongly believe Civo's k3s solution is the middle ground between a full-blown Kubernetes setup vs an abstracted Kubernetes offering. 


# The stack
So I had a pretty solid idea of how I wanted to set this up, and with which tools and projects. 

![A high-level overview of the stack](https://raw.githubusercontent.com/wiardvanrij/civo-images/main/overview.png)

Let's go over each part

## Prometheus

[Prometheus](https://prometheus.io), a Cloud Native Computing Foundation project, is a systems and service monitoring system. It collects metrics from configured targets at given intervals, evaluates rule expressions, displays the results, and can trigger alerts when specified conditions are observed.

### Prometheus Operator & kube-prometheus

The [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator) provides Kubernetes native deployment and management of Prometheus and related monitoring components. The purpose of this project is to simplify and automate the configuration of a Prometheus-based monitoring stack for Kubernetes clusters.

There is also a project called [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus). This provides example configurations for a complete cluster monitoring stack based on Prometheus and the Prometheus Operator. It includes deployment of multiple Prometheus and Alertmanager instances, metrics exporters such as the node_exporter for gathering node metrics, scrape target configuration linking Prometheus to various metrics endpoints, and example alerting rules for notification of potential issues in the cluster. 

In short; we can leverage the `kube-prometheus` project to create a solid setup without much work from our end. Especially for a hobby project, this works perfectly. I would recommend that if you are going to run this in Production; one should understand each component and not just `kubectl apply -f magic.yaml`. Please be aware of what everything does and how you can maintain & sustain it over a longer production time.

## Grafana

Lucky us, Grafana is included in the `kube-prometheus` project. However, let me quote their information for those who are yet unaware of what Grafana is:

> The open-source platform for monitoring and observability.

>Grafana allows you to query, visualize, alert on and understand your metrics no matter where they are stored. Create, explore, and share dashboards with your team and foster a data-driven culture

It means we can use Prometheus as a data source for Grafana and query our metrics. This then allows us to create beautiful graphs, visualizing our metrics. 

## json_exporter

The [json_exporter](https://github.com/prometheus-community/json_exporter) project is an exporter for Prometheus. There are many exporters available to 'extend' Prometheus. For example, there is one for Windows, Elasticsearch, MySQL, and the list goes on. Feel free to check out the [prometheus-community list](https://github.com/prometheus-community) of repositories.

We can leverage this project to parse JSON API 'output' from Twitter to metrics that Prometheus can then scrape. 

An example would be the following JSON:

```json
{
    "counter": 1234,
    "values": [
        {
            "id": "id-A",
            "count": 1,
            "some_boolean": true,
            "state": "ACTIVE"
        },
        {
            "id": "id-B",
            "count": 2,
            "some_boolean": true,
            "state": "INACTIVE"
        },
        {
            "id": "id-C",
            "count": 3,
            "some_boolean": false,
            "state": "ACTIVE"
        }
    ],
    "location": "mars"
}
```

Which, with the proper configuration, can be converted to these metrics:

```
example_global_value{environment="beta",location="planet-mars"} 1234
example_value_active{environment="beta",id="id-A"} 1
example_value_active{environment="beta",id="id-C"} 1
example_value_boolean{environment="beta",id="id-A"} 1
example_value_boolean{environment="beta",id="id-C"} 0
example_value_count{environment="beta",id="id-A"} 1
example_value_count{environment="beta",id="id-C"} 3
```

## Image renderer

Since Grafana 7.x, Grafana needs an external image renderer to render dashboard panels into images. This can be used for alerts. For example, Grafana triggers an alert, it can send an image showing the state of the graph it alerted on. We can also leverage this functionality to create images of panels on demand. 

Grafana provides a [grafana-image-renderer](https://github.com/grafana/grafana-image-renderer) 

The integration is fairly simple. One should create a deployment which can be fairly basic such as:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: image-render
  name: image-render
spec:
  replicas: 1
  selector:
    matchLabels:
      app: image-render
  strategy: {}
  template:
    metadata:
      labels:
        app: image-render
    spec:
      containers:
      - image: grafana/grafana-image-renderer:latest
        name: grafana-image-renderer
        resources: {}
        ports:
        - containerPort: 8081
```

Next, we should expose this deployment via a `service`. Either via kubectl expose or create an svc by hand.
As the final step we have to extend our Grafana deployment with two environment variables to make Grafana aware of this remote image renderer:

```yaml
- env:
    - name: GF_RENDERING_SERVER_URL
        value: http://image-render:8081/render
    - name: GF_RENDERING_CALLBACK_URL
        value: http://grafana:3000/
```

The `GF_RENDERING_SERVER_URL` should be the URL towards your image-renderer, including the port and path (`/render`). The callback URL `GF_RENDERING_CALLBACK_URL` should be the URL of your Grafana instance.


## The missing link

If we got Prometheus running and scraping, we can then visualize the data. We can also do a GET request to Grafana to create an image for us. However, we need to automate this. 
Every minute we want to:
* GET an image from Grafana
* POST this image to Twitter

Now my preferred way to program/automate things is via Go. However, I could not find an SDK or wrapper for the Twitter API that exposed the Twitter user's banner endpoint in Go. Hence I googled further and found out that there was a Python project called [tweepy](https://github.com/tweepy/tweepy) that did!

Now I'll be honest here, if I would be making something in Go, I would have made it pretty sweet. An application that ran indefinitely with a custom timer that fetches an image and checks if everything was sane. I'm new to Python and I did not want to put too much effort into it.

Therefore I've created it in a way that it would be fire & forget. It would just do a thing and be done with it. It is the following code:

```python
import tweepy
import os
import requests


consumer_key =os.environ['consumer_key']
consumer_secret =os.environ['consumer_secret']
access_token =os.environ['access_token']
access_token_secret =os.environ['access_token_secret']

auth = tweepy.OAuthHandler(consumer_key, consumer_secret)
auth.set_access_token(access_token, access_token_secret)

image_url = "http://grafana:3000/render/d-solo/wR0Yf26Gk/twitter-followers?orgId=1&panelId=2&width=1500&height=500&tz=Europe%2FAmsterdam"

img_data = requests.get(image_url).content
with open('twitter_banner.png', 'wb') as handler:
    handler.write(img_data)

api = tweepy.API(auth)
api.update_profile_banner('twitter_banner.png')
```

It's not great in multiple ways:

- It does not check if the image is fetched
- It has 0 sanity checks

However, since we are just doing fire & forget about it:

- The image is never persistent because the pod gets run every minute in a clean environment
- If the image is not present, it would fire an empty path and just throw an error and never publish anything to Twitter

For a hobby project; "it's fine". If one should use this in an actual environment, one should create a Grafana API key and call Grafana in such a way, so that credentials are not exposed internally. Adding more sanity checks and other improvements would very much be necessary.

Regardless, I've created a Docker image for it and created a very simple Kubernetes job that looks somewhat like this:
```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: push-image
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      ttlSecondsAfterFinished: 60
      template:
        spec:
        ...
```

It would create a pod with my image every minute. 

## Civo and cloud-native

The stack consists of various cloud-native components. This is great because it often removes certain aspects such as storage. For example, Grafana dashboards are created via our infra-as-code and are stored in git. They are applied as configmaps and mounted to Grafana to use. Any other configuration works the same way. Our Prometheus scrape targets are done via ServiceMonitors, which are stored on k3s. Static targets via configmaps in our PrometheusSpec;

```yaml
  additionalScrapeConfigs:
    name: additional-configs
    key: scrapeconfig.yaml
```

We still have to store data for Prometheus. It's fine to use our k3s hosts to store a little bit of data. I have limited the storage of Prometheus so I don't run into any node disk issues.

```yaml
spec:
  retention: 4h
```

The great thing is that there are cloud-native solutions to this. For example [Cortex](https://github.com/cortexproject/cortex) and [Thanos](https://github.com/thanos-io/thanos). If we pick Thanos as an example;

> Thanos is a set of components that can be composed into a highly available metric system with unlimited storage capacity, which can be added seamlessly on top of existing Prometheus deployments.

So, instead of working with (often) expensive disks, we can leverage these cloud-native components for great solutions. Object storage is cheap, easy to use, easy to manage. Either way, we can use Minio on Civo: [Guides for minio](https://www.civo.com/learn/categories/minio) or use any other object store solution. 

It would look something like this:

![Thanos object store example](https://raw.githubusercontent.com/wiardvanrij/civo-images/main/objectstore.png)