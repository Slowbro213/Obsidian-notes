In this section I'll be giving a brief overview of the layout of the project, how it's split up, and what each part is responsible for. First, I'll give the top level of the layout.

<div style="display: flex; align-items: center; gap: 1em;">
  <img src="Pasted image 20260813205517.png" style="width: 40%;" />
  <div>

<p>
As shown in the image on the left, it may look like the project is split into three sections, given the three top-level directories. In reality it's just two:
</p>
<p>
The <code>nixos</code> directory is responsible for all the OS configurations of my nodes. Every config of every process that runs on my nodes which is not controlled by Kubernetes is controlled by these .nix files. Things like packages installed, network configs, users and their permissions, the filesystem, everything needed for Kubernetes itself to start, etc., is managed here. As such, this is probably the most crucial part of my project (not including the physical management of the nodes themselves).
</p>
<p>
The <code>clusters</code> and <code>apps</code> directories work together to declaratively manage the Kubernetes cluster through <a href="https://argo-cd.readthedocs.io/en/stable/">ArgoCD</a>. My cluster follows an 'App of Apps' pattern: I define a single, unchanging <code>Application</code> resource in ArgoCD whose only job is to sync all the other <code>Application</code> resources, which do change over time. This lets me manage my cluster purely through Git. Adding, changing, or removing YAML files in the repo rolls out changes, without ever having to touch the nodes manually. Inside <code>clusters</code> you'll find a directory called <code>gentoo</code>, a leftover name from early on, when the cluster ran on just the <code>Tux</code> node, which at the time had the <code>Gentoo</code> Linux distribution installed. And so the whole cluster ended up having that name! That directory only holds the <code>Application</code> resource definitions themselves; their actual configuration lives in the top-level <code>apps</code> directory.
</p>
<p>
The <code>scripts</code> directory holds just one bash script, which does a short probe of the health of the cluster.
</p>
  </div>
</div>

<div style="display: flex; align-items: center; gap: 1em;">
  <img src="Pasted image 20260813213743.png" style="width: 40%;" />
  <div>

<p>
As shown in the image on the left, it may look like the project is split into three sections, given the three top-level directories. In reality it's just two:
</p>
</p>
  </div>
</div>