# Run k6 using Azure Pipelines from your GitHub project

Last month Microsoft introduced the world to Azure DevOps. Azure DevOps is compromised of several services which you can all use together or just cherry-pick the one service you find the most interesting.

One of the cornerstones of their offering is Azure Pipelines.
Currently, Microsoft offers 10 free parallel jobs with unlimited build minutes for all open source projects. That makes trying out and playing with Pipelines an endeavor that costs you literally nothing.

To get started go to [https://dev.azure.com/](https://dev.azure.com/) and click _Start Free_ or login with your existing Microsoft account.
After a successful login, follow the online instructions to create a new project.

![](https://www.dropbox.com/s/h8xz15m8w9lnvdt/Screenshot%202018-11-29%2010.58.43.png)

Click on _New Pipeline_ button.

![](https://www.dropbox.com/s/ronbv2otda5z8vg/Screenshot%202018-11-29%2010.59.58.png?dl=1)

Of course, our code needs to live somewhere. Generally, when I learn new things through a project I have a philosophy not to take on 5 new things, but one at a time. It makes it easier to focus on the new and when something isn't working as expected it is way easier to find the root of the problem. Because of that, for this project, I'm choosing to host my code on good old trustworthy GitHub.

![](https://www.dropbox.com/s/rzvrq4uolbkhxwh/Screenshot%202018-11-29%2011.00.56.png?dl=1)

For it, you need to have a project on GitHub. You can create a new project for this, or use an existing one.
Click the _Authorize_ button to connect your repo to Azure Pipelines.

![](https://www.dropbox.com/s/jprli8wcv007xqz/Screenshot%202018-11-29%2011.01.53.png?dl=1)

If you are a member of multiple organizations select the organization that houses your project.

![](https://www.dropbox.com/s/x0q19y121gj2vtg/Screenshot%202018-11-29%2011.02.43.blur.png?dl=1)

Choose to install Azure Pipelines into all repositories or just into a single repo.

![](https://www.dropbox.com/s/qpga0a4qvhgw3u7/Screenshot%202018-11-29%2011.03.42.png?dl=1)

Follow the next few steps to finish giving Azure Pipelines access to your project.
After all these initial steps, the fun part can finally begin.
You can choose to use a starter template or as a true hacker head straight to your GitHub repo and start messing around there.

## Azure Pipelines config

Create a new file in the root of your repo named `azure-pipelines.yml`.

Azure provides us with ready-made pools to run our pipeline agents in. Having a choice between a Windows, MacOS and Ubuntu pool I've selected "Ubuntu 16.04". That gives us the easiest way to install k6 tool through shell commands.

We can start our example by defining the base image for our agent:

azure-pipelines.yml
```
pool:
  vmImage: 'Ubuntu 16.04'
```

Commit this new file and push the code to remote. On each push, a build is automatically triggered.

## Installing k6

For now, our build does nothing useful. Let's change that.
We are going to create our first script task within our pipeline step. On it's GitHub page, k6 provides us with clear instructions [how to install k6 on a Debian based system](https://github.com/loadimpact/k6#linux) so we can copy that verbatim as a multiline script with a descriptive `displayName` attribute.

To make sure that k6 is installed properly we can add a new script task that just outputs version of our k6 install. This task is optional and can be removed in real-life scenarios.

In the example in this article, I'm using a single-job build so I'm omitting "jobs" section.

For now, we have the following:

azure-pipelines.yml
```
pool:
  vmImage: 'Ubuntu 16.04'

steps:
- script: |
    sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 379CE192D401AB61
    echo "deb https://dl.bintray.com/loadimpact/deb stable main" | sudo tee -a /etc/apt/sources.list
    sudo apt-get update
    sudo apt-get install k6
  displayName: Install k6 tool

  - script: |
    k6 version
  displayName: Check if k6 is installed
```

Commit and push the code. Go to Pipeline's web UI and check out steps outputs. Now we are getting somewhere!

![](https://www.dropbox.com/s/k1tp0il1pvx6rti/Screenshot%202018-12-04%2009.29.32.png?dl=1)

## k6 run

We have k6 installed in our agent so it's time to run a proper load test.
k6 uses JavaScript as it's scripting language making it very flexible to write any kind of load testing scenario.

In our test, we will be testing a single webpage. We are ramping up for 10s from 1 to 15 virtual users, stay at that number for 20 seconds, and then bring the test back to 0 virtual users.

We are naming this test `local.js` since it is being run locally straight from Azure Pipelines agent.

Since we are planning to have multiple test scenarios it's good practice to create a separate directory that will house them. Create `loadtest` directory in the root of the repo and place `local.js` file within it.

loadtests/local.js
```
import { check, group, sleep } from "k6";
import http from "k6/http";

export let options = {
    stages: [
        { duration: "10s", target: 15 },
        { duration: "20s", target: 15 },
        { duration: "10s", target: 0 }
    ],
    thresholds: {
        "http_req_duration": ["p(95)<250"]
    },
    ext: {
        loadimpact: {
            name: "test.loadimpact.com"
        }
    }
};

export default function() {
    group("Front page", function() {
        let res = http.get("http://test.loadimpact.com/");
        check(res, {
            "is status 200": (r) => r.status === 200
        });
        sleep(5);
    });
}

```

In `azure-pipelines.yml` config file we add a new script task to run k6 load test directly from Azure Pipelines agent:

```
steps:
# ...
- script: |
    k6 run loadtests/local.js
  displayName: Run k6 load test within Azure Pipelines
```

Once again: commit and push.

![](https://www.dropbox.com/s/y2ccavc97ca4kyl/Screenshot%202018-12-04%2009.32.39.png?dl=1)

If you got this screen, congrats! You now know how to set up a GitHub project CI build to run on Azure Pipelines and do some sweat load testing.

## k6 cloud run

k6 can also be used as a client to Load Impact's SaaS load testing solution.

One of the advantages of using cloud execution is extremely easy testing from multiple data centers (load zones).

Our test script is almost the same as the one presented above, with a small detail changed. We have now defined to run our load test from two datacenters (Ashburn and Dublin). Not all of your customers live in a single location, so it's smart to test from multiple locations, be that from multiple locations within the US mainland, or from all over the globe.
For a list of all available datacenters from which a load test can be run see [k6 docs](https://docs.k6.io/docs/cloud-execution#section-cloud-execution-options).
Let's create a new file in our `loadtests` dir named `cloud.js`.

loadtests/cloud.js
```
import { check, group, sleep } from "k6";
import http from "k6/http";

export let options = {
    stages: [
        { duration: "10s", target: 15 },
        { duration: "20s", target: 15 },
        { duration: "10s", target: 0 }
    ],
    thresholds: {
        "http_req_duration": ["p(95)<250"]
    },
    ext: {
        loadimpact: {
            name: "test.loadimpact.com",
            distribution: {
                loadZoneLabel1: { loadZone: "amazon:us:ashburn", percent: 60 },
                loadZoneLabel2: { loadZone: "amazon:ie:dublin", percent: 40 }
              }
        }
    }
};

export default function() {
    group("Front page", function() {
        let res = http.get("http://test.loadimpact.com/");
        check(res, {
            "is status 200": (r) => r.status === 200
        });
        sleep(5);
    });
}

```

In `azure-pipelines.yml` config file add an additional step or modify your existing step that was running k6 locally.

```
steps:
# ...
- script: |
    k6 login cloud --token $(k6cloud.token)
    k6 cloud --quiet loadtests/cloud.js
  displayName: Run k6 cloud load test within Azure Pipelines
```

The important thing to notice here is the usage of token provided from Load Impact.
So don't commit your code just yet. First, we need to get the Load Impact token and set Azure Pipelines secret variable.

In order to get the token, log in into your account on [loadimpact.com](https://loadimpact.com) or create it if you don't have one already. New users can use the 30-day free trial period that supports testing from multiple load zones.

Go over to [_Integrations_ section]((https://app.loadimpact.com/integrations)) of the page and click [_Use your token_](https://app.loadimpact.com/account/token) link. Copy the provided token.

![](https://www.dropbox.com/s/pgaqmh4g6be0fzb/Screenshot%202018-11-30%2012.png?dl=1)

Now we need to add an Azure Pipelines variable that will be available within the build.

Go to Azure Pipelines web UI and from the left side menu select _Pipelines_ and then click the _Edit_ button next to the name of your project.

![](https://www.dropbox.com/s/zfgypz05izp7qgh/Screenshot%202018-11-29%2011.44.02_annotated.png?dl=1)

Once you enter the options for your project switch to _Variables_ tab within the web UI and enter a new secret variable that will be used during k6 cloud execution.

![](https://www.dropbox.com/s/r91alaprypqmvq4/Screenshot%202018-11-29%2011.50.12.png?dl=1)

Click _Add_ to add a new variable.
Under name enter `k6cloud.token` and for its value paste Load Impact's token.
Don't forget to set the variable as secret so that it's not visible as plain text in your pipelines output.
After entering the values click _Save & queue_ button above and into field _Save comment_ enter something like "adding k6cloud.token env var".

![](https://www.dropbox.com/s/2ktfhjxdvp0nppb/Screenshot%202018-11-29%2012.58.37.png?dl=1)

Now, you can push your new `loadtests/cloud.js` script alongside new Pipelines script task to trigger a new build.
You can see k6 output in Azure Pipelines web UI, but for a more in-depth view and analysis of your test go to [Load Impact's web UI](https://app.loadimpact.com).

![](https://www.dropbox.com/s/ne1kctle8b718na/Screenshot%202018-12-04%2009.41.23.png?dl=1)

## Wrap up

All code used in this article is available in a [public GitHub repo](https://github.com/loadimpact/k6-azure-pipelines-example). Feel free to use it as a starting point for setting up your own Azure Pipelines builds.

Happy testing!
