# Web IDEs

By default, Coder workspaces allow connections via:

- Web terminal
- SSH (plus any [SSH-compatible IDE](../ides.md))

It's common to also let developers to connect via web IDEs.

![Row of IDEs](../images/ide-row.png)

In Coder, web IDEs are defined as
[coder_app](https://registry.terraform.io/providers/coder/coder/latest/docs/resources/app)
resources in the template. With our generic model, any web application can
be used as a Coder application. For example:

```hcl
# Add button to open Portainer in the workspace dashboard
# Note: Portainer must be already running in the workspace
resource "coder_app" "portainer" {
  agent_id      = coder_agent.main.id
  slug          = "portainer"
  display_name  = "Portainer"
  icon          = "https://simpleicons.org/icons/portainer.svg"
  url           = "https://localhost:9443/api/status"

  healthcheck {
    url       = "https://localhost:9443/api/status"
    interval  = 6
    threshold = 10
  }
}
```

## code-server

![code-server in a workspace](../images/code-server-ide.png)

[code-server](https://github.com/coder/coder) is our supported method of running VS Code in the web browser. A simple way to install code-server in Linux/macOS workspaces is via the Coder agent in your template:

```sh
# edit your template
cd your-template/
vim main.tf
```

```hcl
resource "coder_agent" "main" {
    arch          = "amd64"
    os            = "linux"
    startup_script = <<EOF
    #!/bin/sh
    # install and start code-server
    curl -fsSL https://code-server.dev/install.sh | sh
    code-server --auth none --port 13337
    EOF
}
```

For advanced use, we recommend installing code-server in your VM snapshot or container image. Here's a Dockerfile which leverages some special [code-server features](https://coder.com/docs/code-server/):

```Dockerfile
FROM codercom/enterprise-base:ubuntu

# install a specific code-server version
RUN curl -fsSL https://code-server.dev/install.sh | sh -s -- --version=4.3.0

# pre-install versions
RUN code-server --install-extension eamodio.gitlens

# directly start code-server with the agent's startup_script (see above),
# or use a process manager like supervisord
```

You'll also need to specify a `coder_app` resource related to the agent. This is how code-server is displayed on the workspace page.

```hcl
resource "coder_app" "code-server" {
  agent_id     = coder_agent.main.id
  slug         = "code-server"
  display_name = "code-server"
  url          = "http://localhost:13337/?folder=/home/coder"
  icon         = "/icon/code.svg"

  healthcheck {
    url       = "http://localhost:13337/healthz"
    interval  = 2
    threshold = 10
  }

}
```

## JetBrains Projector

[JetBrains Projector](https://jetbrains.github.io/projector-client/mkdocs/latest/) is a JetBrains Incubator project which renders JetBrains IDEs in the web browser.

![PyCharm in Coder](../images/projector-pycharm.png)

> It is common to see latency and performance issues with Projector. We recommend using [JetBrains Gateway](https://youtrack.jetbrains.com/issues/GTW) whenever possible (also no Template edits required!)

Workspace requirements:

- JetBrains projector CLI
- At least 4 CPU cores and 4 GB RAM

- CLion
- PyCharm
- DataGrip
- GoLand
- IntelliJ IDEA Community
- IntelliJ IDEA Ultimate
- PhpStorm
- PyCharm Community
- PyCharm Professional
- Rider
- RubyMine
- WebStorm

For advanced users who want to make a custom image, you can install the
Projector CLI in the `startup_script` of the `coder_agent` resource in a Coder
template. Using the Projector CLI, you can use `projector config add` and
`projector run` to configure and start a JetBrains IDE in your workspace.

> Install the JetBrains IDE in your image and `chown` the `/opt` directory to
> the user that starts the workspace. e.g., `coder`

![IntelliJ in Coder](../images/projector-intellij.png)


**Pre-built templates:**

You can also reference/use to these pre-built templates with JetBrains
projector:

- IntelliJ
  ([Docker](https://github.com/sharkymark/v2-templates/tree/main/docker-with-intellij),
  [Kubernetes](https://github.com/sharkymark/v2-templates/tree/main/multi-projector-intellij))

- PyCharm
  ([Docker](https://github.com/sharkymark/v2-templates/tree/main/docker-with-pycharm),
  [Kubernetes](https://github.com/sharkymark/v2-templates/tree/main/multi-projector-pycharm)

> You need to have a valid `~/.kube/config` on your Coder host and a namespace
> on a Kubernetes cluster to use the Kubernetes pod template examples.

======= ![PyCharm in Coder](../images/projector-pycharm.png)

## JupyterLab

Configure your agent and `coder_app` like so to use Jupyter:

```hcl
data "coder_workspace" "me" {}

## The name of the app must always be equal to the "/apps/<name>"
## string in the base_url. This caveat is unique to Jupyter.

locals {
  jupyter_base_path = "/@${data.coder_workspace.me.owner}/${data.coder_workspace.me.name}/apps/JupyterLab/"
}

resource "coder_agent" "coder" {
  os   = "linux"
  arch = "amd64"
  dir  = "/home/coder"
  startup_script = <<-EOF
pip3 install jupyterlab
$HOME/.local/bin/jupyter lab --ServerApp.base_url=${local.jupyter_base_path} --ServerApp.token='' --ip='*'
EOF
}

resource "coder_app" "jupyter" {
  agent_id     = coder_agent.coder.id
  slug         = "jupyter"
  display_name = "JupyterLab"
  url          = "http://localhost:8888${local.jupyter_base_path}"
  icon         = "/icon/jupyter.svg"

  healthcheck {
    url       = "http://localhost:8888${local.jupyter_base_path}"
    interval  = 5
    threshold = 10
  }
}
```

![JupyterLab in Coder](../images/jupyterlab-port-forward.png)


### RStudio

Configure your agent and `coder_app` like so to use RStudio. Notice the
`subdomain=true` configuration:

```hcl
resource "coder_agent" "coder" {
  os   = "linux"
  arch = "amd64"
  dir = "/home/coder"
  startup_script = <<EOT
#!/bin/bash
# start rstudio
/usr/lib/rstudio-server/bin/rserver --server-daemonize=1 --auth-none=1 &
EOT
}

# rstudio
resource "coder_app" "rstudio" {
  agent_id      = coder_agent.coder.id
  name          = "rstudio"
  icon          = "/icon/rstudio.svg"
  url           = "http://localhost:8787"
  subdomain = true
  share     = "owner"

  healthcheck {
    url       = "http://localhost:8787/healthz"
    interval  = 3
    threshold = 10
  } 
}
```

![RStudio in Coder](../images/rstudio-port-forward.png)

### Airflow

Configure your agent and `coder_app` like so to use Airflow. Notice the
`subdomain=true` configuration:

```hcl
resource "coder_agent" "coder" {
  os   = "linux"
  arch = "amd64"
  dir = "/home/coder"
  startup_script = <<EOT
#!/bin/bash
# install and start airflow
pip3 install apache-airflow
/home/coder/.local/bin/airflow standalone &
EOT
}

resource "coder_app" "airflow" {
  agent_id      = coder_agent.coder.id
  name          = "airflow"
  icon          = "https://upload.wikimedia.org/wikipedia/commons/d/de/AirflowLogo.png"
  url           = "http://localhost:8080"
  subdomain = true
  share     = "owner"

  healthcheck {
    url       = "http://localhost:8080/healthz"
    interval  = 10
    threshold = 60
  } 
}
```

![Airflow in Coder](../images/airflow-port-forward.png)

## SSH Fallback

If you prefer to run web IDEs in localhost, you can port forward using
[SSH](../ides.md#ssh) or the Coder CLI `port-forward` sub-command. Some web IDEs
may not support URL base path adjustment so port forwarding is the only
approach.