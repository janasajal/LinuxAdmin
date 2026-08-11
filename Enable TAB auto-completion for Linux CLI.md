# Linux Bash Completion – Quick Note

**Purpose:** Enable `TAB` auto-completion for Linux CLI tools such as **Helm** and **OpenShift `oc`**.

## 1. Check Bash & bash-completion

```bash
$ echo $SHELL
/bin/bash

$ rpm -q bash-completion
bash-completion-2.11-xx.el9.noarch
```

If not installed:

```bash
$ sudo dnf install -y bash-completion
```

Load Bash completion:

```bash
$ source /usr/share/bash-completion/bash_completion
```

---

## 2. Helm Bash Completion

First check the Helm location:

```bash
$ which helm
/usr/local/bin/helm
```

Install the completion script:

```bash
$ sudo sh -c '/usr/local/bin/helm completion bash > /etc/bash_completion.d/helm'
```

**Why use the absolute path?**  
`sudo` may use a restricted `PATH` that does not include `/usr/local/bin`.

Load the completion:

```bash
$ source /etc/bash_completion.d/helm
```

Verify:

```bash
$ complete -p helm
complete -o bashdefault -o default -F __start_helm helm
```

Test:

```bash
$ helm ins<TAB>
```

Result:

```bash
$ helm install
```

---

## 3. OpenShift `oc` Bash Completion

Check the `oc` location:

```bash
$ which oc
/usr/local/bin/oc
```

Install the completion script:

```bash
$ sudo sh -c '/usr/local/bin/oc completion bash > /etc/bash_completion.d/oc'
```

Load it:

```bash
$ source /etc/bash_completion.d/oc
```

Verify:

```bash
$ complete -p oc
complete -o bashdefault -o default -F __start_oc oc
```

Test:

```bash
$ oc get po<TAB>
```

Result:

```bash
$ oc get pods
```

---

## 4. Verify Completion Files

```bash
$ ls -l /etc/bash_completion.d/helm
$ ls -l /etc/bash_completion.d/oc
```

Expected:

```text
-rw-r--r--. 1 root root ... /etc/bash_completion.d/helm
-rw-r--r--. 1 root root ... /etc/bash_completion.d/oc
```

---

## 5. Quick Troubleshooting

If TAB completion is not working:

```bash
$ echo $SHELL
$ rpm -q bash-completion
$ which helm
$ which oc
$ complete -p helm
$ complete -p oc
```

Check that the completion scripts can be generated:

```bash
$ helm completion bash | head
$ oc completion bash | head
```

### Key Point

For tools installed in `/usr/local/bin`, prefer:

```bash
sudo sh -c '/usr/local/bin/helm completion bash > /etc/bash_completion.d/helm'
```

instead of:

```bash
sudo sh -c 'helm completion bash > /etc/bash_completion.d/helm'
```

because the `sudo` environment may not include `/usr/local/bin` in its `PATH`.
