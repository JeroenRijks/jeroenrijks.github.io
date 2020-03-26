---
layout: post
title: "Kubectl Aliases"
subtitle: "How to be speedy"
date: 2020-03-26 00:30:00 -0400
background: '/img/posts/05.jpg'
---

When I started at Transreport, I set up Helm charts for Passenger Assist, to make our deployments easier to set up. In hindsight, it was also a really good way to get to know the company's products, because to set the Helm charts up, I needed a good understanding of the architecture, and what needs to happen for the program to run.

Working with Kubernetes requires the use of `kubectl`, a great command line tool that lets you interact with Kubernetes' API. However, using `kubectl` gets repetitive pretty quickly, especially if you're working with namespaces; I regularly run commands like these:
```
aws eks update-kubeconfig --name <passenger-assist-production-cluster-name>
kubectl get pods -n sqr-emr-staging
kubectl get pod -n pa-staging <pod-name> -o yaml
kubectl describe configmap -n sqr-emr-uat
```

To save time, and to reduce the chance of making a typo, I've set up a few aliases, allowing me to replace those commands with:
```
kswitchprodpa
kgn po
kgyn po <pod-name>
kdn cm
```


### Aliases
I (on my Ubuntu 18 OS) use aliases to speed my Kubernetes work up drastically. To avoid writing `kubectl`, I added 
```
alias k='kubectl'
```
to my `~/.bashrc` file. Every time I open a new terminal, it runs every line in my `~/.bashrc` file, which will now replace the letter `k` with `kubectl`, if I write it at the beginning of a command. Note that any already-open terminals won't respond to changes in the `~/.bashrc` file, unless you run `source ~/.bashrc`.


## Simple kubectl aliases
Here are some standard `kubectl` commands that I replaced:
```
alias k='kubectl'
alias ka='kubectl apply'
alias kl='kubectl logs'
alias ke='kubectl edit'
alias krm='kubectl delete'
alias kd='kubectl describe'
alias keit='kubectl exec -it'
alias kg='kubectl get'
alias kgsys='kubectl get -n kube-system'
alias wkg='watch kubectl get'
alias kgy='kubectl get -o yaml'
alias kwhere='kubectl config current-context'
```

## Kubernetes namespaces
Kubernetes uses namespaces to group resources together, and isolate them from other groups. `kubectl` lets you select the namespace you're interested in by adding `-n <namespace-name>` to your `kubectl` command. However, most of the time, you'll need to run several commands in the same namespace, so adding `-n  <namespace-name>` gets pretty tedious, especially when namespaces have long names. That's why I've set up these aliases:

```
alias padev='KUBE_NAMESPACE=pa-dev'
alias pastaging='KUBE_NAMESPACE=pa-staging'
### There's a whole lot more of these
```
and 
```
alias kln='kubectl logs -n $KUBE_NAMESPACE'
alias ken='kubectl edit -n $KUBE_NAMESPACE'
alias krmn='kubectl delete -n $KUBE_NAMESPACE'
alias kdn='kubectl describe -n $KUBE_NAMESPACE'
alias keitn='kubectl exec -it -n $KUBE_NAMESPACE'
alias kgn='kubectl get -n $KUBE_NAMESPACE'
alias wkgn='watch kubectl get -n $KUBE_NAMESPACE'
alias kgyn='kubectl get -o yaml -n $KUBE_NAMESPACE'
```

Now, simply adding an `n` to the end of my standard `kubectl` shortcuts stops me from accidentally writing `-n pa-aut` and `-n sqr-emr=staging` multiple times a day.

## Switching clusters

At Transreport, we use Terraform to create our EKS clusters, and to avoid naming clashes, their names have randomised strings at the ends. It's annoying to have to learn them, so instead, I have these aliases set up:
```
alias kswitchnonprodpa='aws eks update-kubeconfig --name <name-of-eks-cluster>>'
alias kswitchprodpa='aws eks update-kubeconfig --name <name-of-eks-cluster>>'
alias kswitchnonprodcommon='aws eks update-kubeconfig --name <name-of-eks-cluster>>'
alias kswitchprodcommon='aws eks update-kubeconfig --name <name-of-eks-cluster>>'
```

## Helm
Helm is essentially a Kubernetes package manager, and its command line tool, `helm`, works pretty similarly to `kubectl`, in terms of using namespaces. Use the same trick:

```
alias helmn='helm -n $KUBE_NAMESPACE`
alias helmlsn='helm ls -n $KUBE_NAMESPACE`
alias helmgvn='helm get values -n $KUBE_NAMESPACE`
alias helmtp='helm template --output-dir .`
alias helmtpn='helm template --output-dir . -n $KUBE_NAMESPACE`
alias helmsearch='helm search repo'  ### Avoid getting it wrong and typing helm repo search every time.
```

Other helm commands like `helm install` probably deserve the time taken to type the command out.
 
### Side note for Terraform users

Do yourself a favour and never write `terrafrom` again:
```
alias tf='terraform'
alias tfi='terraform init'
alias tfws='terraform workspace'
alias tfwsl='terraform workspace list'
alias tfs='terraform state'
alias tfsl='terraform state list'
alias tfp='terraform plan'
alias tfa='terraform apply'
alias tfaa='terraform apply -auto-approve'   --> Don't use this unless you're a #madlad
```

