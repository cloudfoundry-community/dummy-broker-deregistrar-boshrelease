
# Dummy Broker Deregister errand

## Sad scenario with Pivotal Ops Manager

This BOSH release was created to help unf@#$ Pivotal Ops Manager when it wants to delete a service-broker Tile that never successfully deployed in the first place.

The scenario is that you've deployed a Tile to your Pivotal Ops Manager for a service broker. Somewhere in the deployment it fails. You decide to delete the broker tile. But you forget to disable the "Deregister Broker" errand first. So, eventually the Delete change fails:

```
Errand `broker-deregistrar' did not complete
```

Unfortunately now you cannot edit the Tile and disable the errand - its in a "being deleted" mode.

![](http://cl.ly/1D0R091c2o3w/Image%202016-02-16%20at%209.55.40%20am.png)

## Solution

So, here's a solution: upload this release - with its `broker-deregistrar` errand - and switch out the deployment manifest for one that uses this release instead and only this release. Then Ops Manager will run this release's errand and it will finish successfully. And then Ops Manager will remove the Tile. And you can move on.

## Steps

SSH into your Ops Manager VM:

```
ssh ubuntu@<opsmgr-ip>
```

Change to the directory where deployment manifests are stored:

```
cd /var/tempest/workspaces/default/deployments
```

Find your unhappy deployment manifest, make a backup, and edit the `sad-tile.yml` manifest.

```
sudo cp sad-tile.yml sad-tile.yml.backup
sudo vi sad-tile.yml
```

Replace `releases:` with just this dummy release:

```yaml
releases:
- name: dummy-broker-deregistrar
  version: latest
```

Remove all `jobs:` except the `broker-deregistrar` errand; and change its `templates:` to use the new release:

```yaml
jobs:
- name: broker-deregistrar
  templates:
  - name: broker-deregistrar
    release: dummy-broker-deregistrar
  lifecycle: errand
  instances: 1
  resource_pool: broker-deregistrar
  networks:
  - name: default
    default:
    - dns
    - gateway
```

In the terminal, target the BOSH and upload the release:



```
bosh target BOSHIP
bosh create release
bosh upload release
```

Next target your edited manifest and run the errand once:

```
bosh deployment sad-tile.yml
bosh -n deploy
bosh run errand broker-deregistrar
```

Running the errand manually confirms that the errand now successfully exits and does nothing.

Now, return to Ops Manager and press "Apply changes" again:

![](http://cl.ly/1D0R091c2o3w/Image%202016-02-16%20at%209.55.40%20am.png)

Eventually the Applying Changes verbose output will happily show:

![](http://cl.ly/3u2b0e3m0Z3Z/Image%202016-02-16%20at%209.58.51%20am.png)
