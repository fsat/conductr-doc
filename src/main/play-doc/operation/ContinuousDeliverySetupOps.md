# Continuous Delivery

[Continuous Delivery](https://en.wikipedia.org/wiki/Continuous_delivery) is a practice of performing frequent, small deployments which aims to reduce time, risk, and cost associated with producing and deploying new feature. Despite the benefits, this practice has its own set of challenges, one of them often being the technical complexity associated with setting up the pipeline to enable continuous, automated deployment.

ConductR now provides Continuous Delivery feature which aims to help your team reap the associated benefits of Continous Delivery, and at the same time reducing the technical complexity normally associated with this practice.

Continuous Delivery feature is delivered through [[CLI|CLI]] based deployments, webhook based deployments, or combination of both.

The [[CLI|CLI]] provides the `conduct deploy` command which accepts bundle and optional configuration as its input. The `conduct deploy` command will post the supplied bundle and optional configuration to the Continuous Delivery bundle which will then perform the necessary deployment.

The Continuous Delivery bundle also accepts Bintray webhook as its input. The Continuous Delivery bundle accepts the notification of new version of bundles to be deployed through the use of this webhook.

Your team will be in the best position to utilise the Continuous Delivery feature by automatically deploying a successfully built bundle as part of Continuous Integration build using either `conduct deploy` command or the webhook triggered by from [bintray publishing](BintrayPublishing).


## Example pipeline

Here's an example of Continuous Delivery pipeline where bundles are propagated into production.

Let there be 2 clusters: a staging cluster and a production cluster. Both cluster has Continuous Delivery bundle installed. The Continuous Delivery service on the staging environment is accessible from the outside world.

Let's assume the development team is using `git`, and all the code that has been promoted into `master` branch can be released once it has passed the test.

A CI server has been configured to publish bundle artefacts to Bintray upon successful build of recently merged commit into the `master` branch.

The Bintray webhook has been configured to target the staging environment. This would mean each successful merged commit into the `master` branch will be available for manual test on the staging environment.

The manual tests will be performed on the staging environment, and a particular version will be selected for production deployment. As part of the version selection process, older versions which hasn't been selected will be removed using the `conduct unload` command.

Once a particular version has been selected for a production deployment, an operator can use `conduct deploy` command to promote this particular version into the production environment. Once the deployed version is confirmed to be working, the older bundle is then unloaded from the production environment.

ConductR's Continuous Delivery feature doesn't prescribe how the deployment pipeline should look like. If the Bintray webhook in the example pipeline above is pointed to production cluster instead, it would mean continuous deployment to production instead.

Another variation of this example is the omission of Bintray webhook in the process. The CI server may use `conduct deploy` command to promote the recently built bundle artefacts into staging environment instead of using the Bintray webhook.

## Setting up CD pipeline from the CI machine

Let's discuss the setup of the CD pipeline from the CI machine which will allow recently built artifact to be automatically deployed into a target ConductR cluster upon a successful build.

### Overview

A [bastion host](https://en.wikipedia.org/wiki/Bastion_host) will be required to allow CI machine to securely deploy recently built bundle or bundle configuration into a ConductR cluster. The bastion host will be configured to allow passwordless SSH access from the CI machine. Using the passwordless SSH mechanism, the CI machine will be able to remotely invoke the `conduct deploy` command on the bastion host.

The `conduct deploy` allows resolving bundle and bundle configuration from locations such as Bintray, AWS S3, HTTP URL, or Docker registry. This means the deployment pipeline can be established with bundles or bundle configurations hosted on any of these location.

Using this approach, the bundle and the bundle configuration are kept within the bastion host. This is desirable from security's stand point, especially since bundle configuration may contain credentials which may allow access to production system.

The traffic between the CI machine and the bastion host will also be secured through SSH.

### Requirements

Continuous Delivery bundle relies on bundle name and compatibility version to select the existing bundles to be replaced. Because of this, the CD pipeline only works for the bundles whose name has not been modified.

When deploying bundle and bundle configuration from S3, the full S3 URL of the bundle and bundle configuration must be supplied to the `conduct deploy` command. This also applies when deploying bundle and bundle configuration from HTTP URL. Bintray and Docker registry provides metadata which will allow resolution to the latest published version unlike S3 or HTTP URL.

When using `sbt-bintray-bundle` to publish to Bintray, ensure the bundle is released immediately upon publishing. This will allow `conduct deploy` command to obtain the latest published version from Bintray.

```
bintrayReleaseOnPublish := true
```

### Bastion host setup

The bastion host must be setup to allow SSH passwordless authentication from the CI machine. This allow secure access to the target ConductR cluster from an insecure network. Create a designated CI account on the bastion host which can be revoked at any given time - _never use the `root` user under any circumstances for the sake of security_.

Next, the bastion host must be allowed to download the bundle and bundle configuration.

If the bundle and bundle configuration is hosted on Bintray, this would mean creating `~/.bintray/.credentials` which contains the correct username and API key to download the bundle and the bundle configuration.

If the bundle and bundle configuration is hosted on AWS S3, this would mean creating `~/.aws/credentials` which contains correct credentials to download the bundle and the bundle configuration. The [S3 Resolver](DeployingBundlesOps#s3-resolver) has further details on how to setup the `~/.aws/credentials` file.

### CI machine setup

Allow access from CI machine to the Bastion host using designated CI account through passwordless SSH. Execute [ssh-copy-id](https://www.ssh.com/ssh/copy-id) command from the CI machine, copying the CI user's SSH key into the designated account on the Bastion host to allow for passwordless authentication.

Ensure the Bastion Host is registered in the list of `~/.ssh/known-hosts` of the CI machine. The simplest way to do this is to SSH from the CI machine into the Bastion host, and to accept the known host prompt if present. The process of accepting the known host prompt is only needed to be done once.

### Initial bundle setup on the ConductR cluster

Execute `conduct load` of an existing bundle version into your target cluster. Supply bundle configuration containing the environment specific configuration if required.

Once this is done, execute `conduct run` to the desired scale.

### Invoking deployment

Assuming the build has passed and the artefact has been uploaded, the deployment should be invoked in the following manner.

#### Artefacts published to Bintray

Once the artifact is published successfully to Bintray, invoke the following SSH command to trigger the deployment from the Bastion host:

```bash
ssh -t -i <bastion.pem> <ci.user>@<bastion.host> conduct deploy -y <bundle shorthand> --host <cluster.ip>
```

* `<ci.user>` is the designated CI account existing on the Bation host.
* `<bastion.pem>` is the SSH key belonging to the `<ci.user>` which has been copied to the Bastion host using `ssh-copy-id`.
* `<bastion.host>` is the public IP or host address of the bastion host.
* `<cluster.ip>` is the IP or host address of one of the ConductR core node accessible from the bastion host.
* `<bundle shorthand>` is the [shorthand expression](DeployingBundlesOps#Bundle-shorthand-expression) of the bundle recently built and published to bintray.

#### Artefacts published to Docker registry

The invocation of the deploy command is similar to the artefacts published to Bintray. Since the CLI comes with built-in [Docker resolver](DeployingBundlesOps#docker-resolver), `<bundle shorthand>` should be pointed to the newly published Docker image on a particular registry.

#### Artefacts published to S3

The CLI [S3 resolver](DeployingBundlesOps#s3-resolver) accepts full S3 url as input, as such the completed built must supply the full S3 URL of the recently published bundle as the input to the deployment process.

Once the artifact is published successfully to S3, invoke the following SSH command to trigger the deployment from the Bastion host:

```bash
ssh -t -i <bastion.pem> <ci.user>@<bastion.host> conduct deploy -y <bundle s3 url> --host <cluster.ip>
```

* `<ci.user>` is the designated CI account existing on the Bation host.
* `<bastion.pem>` is the SSH key belonging to the `<ci.user>` which has been copied to the Bastion host using `ssh-copy-id`.
* `<bastion.host>` is the public IP or host address of the bastion host.
* `<cluster.ip>` is the IP or host address of one of the ConductR core node accessible from the bastion host.
* `<bundle s3 url>` is the full S3 URL of the recently published bundle, i.e. `s3://my-bucket/path/to/my/bundle-file.zip`

## Requirements

From the perspective of Continuous Delivery setup, the operator is responsible for the installation of the Continuous Delivery service of ConductR. If webhook based deployment is required, it is expected that operations will be responsible for the  configuration of Bintray webhook which communicates with Continuous Delivery service.


## Installing Continuous Delivery service

If only `conduct deploy` based deployment is desired (i.e. no webhook), simply install and run Continuous Delivery bundle.

```
conduct load continuous-delivery
conduct run continuous-delivery --scale 2
```

Alternatively, if webhook based deployment is desired (or the combination of CLI and webhook), the Continuous Delivery bundle need to be configured with the Bintray webhook secret. Replacing `bt-secret` with the actual Bintray API key value.

```
conduct load continuous-delivery --env "BINTRAY_WEBHOOK_SECRET=bt-secret"
conduct run continuous-delivery --scale 2
```

Once the Continuous Delivery service has been started, it will be exposed via the proxy on port `9000` under the path `/deployments`. When using webhook based deployments, ensure external access is available to this port and path to allow the `/deployments` endpoint to be invoked by Bintray webook.

The Continuous Delivery service will form a cluster among its instances, and the deployer which is responsible for deployment is sharded by bundle name.

## Configuring Bintray webhook

This step is only required if you wish to invoke the Continuous Delivery bundle using Bintray webhook.

When triggering automated deployment using a webhook, Continuous Delivery bundle requires a working ConductR Cluster with ConductR HAProxy deployed. ConductR HAProxy is required to allow Bintray webhook access to the endpoint exposed Continuous Delivery bundle. ConductR HAProxy installation instructions is available on the [[Install|Install]] page.

If the bundle is going to be delivered via Bintray webhook, then Bintray webhook setup is required. In this case, Bintray credentials with publish permission and package read/write entitlement is required.


### Bintray webhook secret

The webhook secret is the Bintray API key of the credentials owning the Bintray webhook.

For the purpose of this documentation, we will be using `bt-secret` in place of the actual API key value.


### Bintray webhook setup

Execute the following command to create Bintray webhook.

```
curl -v \
     -u ${BINTRAY_USERNAME}:${BINTRAY_API_KEY} \
     -X POST \
     -H "Content-Type: application/json" \
     -d '{"url": "${CALLBACK_URL}", "method": "post"}' \
     "https://api.bintray.com/webhooks/${BINTRAY_SUBJECT}/${BINTRAY_REPO}/${BINTRAY_PACKAGE_NAME}"
```

|Term|Description|Example|
|----|-----------|-------|
|`BINTRAY_USERNAME`|The Bintray username. This can be obtained by signing into Bintray and viewing the user profile.|&nbsp;|
|`BINTRAY_API_KEY`|The API key of the user. This can be obtained by signing into Bintray and viewing the API key within the user profile.|`bt-secret` is used for our example.|
|`CALLBACK_URL`|The callback URL which will be invoked by the webhook.|&nbsp;|
|`BINTRAY_SUBJECT`|The Bintray subject who owns the `BINTRAY_REPO` in question.|`typesafe` is an example of Bintray subject which is accessible on [https://bintray.com/typesafe](https://bintray.com/typesafe).|
|`BINTRAY_REPO`|The Bintray repository where the bundle to be deployed resides.|`bundle` is an example of Bintray repo which is accessible on [https://bintray.com/typesafe/bundle](https://bintray.com/typesafe/bundle).<br>The repo owns various bundles such as `cassandra` which is accessible from [https://bintray.com/typesafe/bundle/cassandra](https://bintray.com/typesafe/bundle/cassandra).|
|`BINTRAY_PACKAGE_NAME`|The Bintray package of the bundle to be deployed.|`cassandra` is an example of Bintray package which is accessible on [https://bintray.com/typesafe/bundle/cassandra](https://bintray.com/typesafe/bundle/cassandra).|


Format the `CALLBACK_URL` as such.

```
${CD_BASE_URL}/deployments/${BINTRAY_SUBJECT}/${BINTRAY_REPO}/<subject>
```

|Term|Description|Example|
|----|-----------|-------|
|`CD_BASE_URL`|The URL where Continuous Delivery is exposed. This will normally be a load balancer which is mapped to proxy on port `9000` under the path `/deployments`.|`http://staging.acme.com`|
|`BINTRAY_SUBJECT`|Must match the `BINTRAY_SUBJECT` posted to `api.bintray.com`|&nbsp;|
|`BINTRAY_REPO`|Must match the `BINTRAY_REPO` posted to `api.bintray.com`|&nbsp;|

Once the setup is complete, the webhook can be tested using the following command. In the following example `0.0.1` is the test version number of the bundle in question.

```
curl -v \
     -u ${BINTRAY_USERNAME}:${BINTRAY_PASSWORD} \
     -X POST \
     "https://api.bintray.com/webhooks/${BINTRAY_SUBJECT}/${BINTRAY_REPO}/${BINTRAY_PACKAGE_NAME}/0.0.1"
```

Further details on the bintray webhook setup instructions can be found on the [https://bintray.com/docs/api/#_register_a_webhook](https://bintray.com/docs/api/#_register_a_webhook).

## CLI based deployment

The CLI based deployment is done using the `conduct deploy` command.

The `conduct deploy` command accepts bundle and optional configuration as its input. The treatment of the bundle and optional configuration input is the same as the `conduct load` command.

As such, the following are considered as a valid input for both bundle and the optional configuration:

* [Shorthand expression](DeployingBundlesOps#Bundle-shorthand-expression),
* Actual path to the bundle or configuration `.zip` file,
* Directory of the bundle or configuration,
* Or, the HTTP URL where bundle or configuration `.zip` file is hosted.

The bundle and optional configuration input to the `conduct deploy` command will will be processed through the `bndl` tool where appropriate. This allows operator to provide further configuration on top of the existing input. For example:

```
conduct deploy visualizer --nr-of-cpus 2.0 --roles web --roles us-east-1 --env "FOO=BAR"
```

In the example above, the following additional configurations will be applied on the `visualizer` bundle:

* Number of CPUs is increased to `2.0`.
* Roles are set to `web` and `us-east-1`.
* Additional environment variable `FOO=BAR` will be made available to the bundle process when it starts.

To see the available configuration option, run `conduct deploy --help`.

## Webhook based deployment

The webhook based deployment will occur when the Continuous Delivery bundle receives the webhook from Bintray. The Bintray webhook will be triggered upon a successful publishing of a new artefact.


## Simulating Bintray webhook

It is possible to simulate the Bintray webhook using the `conduct deploy` command by specifying `--webhook bintray` argument.

Configure the Continuous Delivery settings in the `~/.conductr/settings.conf` as such, replacing `bt-secret` with the actual Bintray API key value.

```
conductr {
  continuous-delivery {
    bintray-webhook-secret = bt-secret
  }
}
```

Invoke the command as such, where `<bundle>` is the [shorthand expression](DeployingBundlesOps#Bundle-shorthand-expression) of the bundle to be deployed.

```
conduct deploy --webhook bintray <bundle>
```

Note that `--webhook bintray` option will reject optional configuration if supplied as input. This is to match the behaviour of Bintray webhook since the webhook will invoke Continuous Delivery with a new version of bundle without configuration.

For bundles deployed in this manner, configuration from previous version of the running bundle will be applied to the version of the bundle presently being deployed if present.


## Deployment result

The result of the deployment depends on whether any compatible bundles are found running within ConductR.

A bundle is considered compatible if it has the same bundle name and `compatibilityVersion`.

The `compatibilityVersion` bundle configuration will be used to check if the versions being deployed is binary compatible with the version being replaced. ConductR expects that two different versions of a bundle having the same `compatibilityVersion` will be binary compatible.

### Input: bundle and configuration

|Compatible Bundle|Expected Result|
|------------------------|---------------|
|None|The input bundle and configuration will be deployed and scaled to `1` instance.|
|One running compatible bundle|The input bundle and configuration will be deployed, scaled in the lock-step fashion to replace the compatible bundle|
|One running compatible bundle + configuration|The input bundle and configuration will be deployed, scaled in the lock-step fashion to replace the compatible bundle. Note the configuration from the input will be used instead.|
|Multiple running compatible bundle + configuration|The Continuous Delivery bundle will consider this an ambiguous situation. As such the deployment will be cancelled and error will be raised.|

### Input: bundle only

|Compatible Bundle|Expected Result|
|------------------------|---------------|
|None|The input bundle will be deployed and scaled to `1` instance.|
|One running compatible bundle|The input bundle will be deployed, scaled in the lock-step fashion to replace the compatible bundle|
|One running compatible bundle + configuration|The input bundle will be deployed, scaled in the lock-step fashion to replace the compatible bundle. The configuration from the compatible bundle will be reapplied.|
|Multiple running compatible bundle + configuration|The input bundle will be deployed, scaled in the lock-step fashion to replace each of the compatible bundle found. The configuration from each of the compatible bundle will be reapplied.|


## Housekeeping

After a successful deployment, Continuous Delivery service *will not* unload the older bundle which has been stopped to allow the operator to quickly fallback and run the older bundle.

However this would mean the older bundle will be kept within ConductR until the older bundle is unloaded. As such, it is a good practice to unload older versions of bundles on a regular basis.
