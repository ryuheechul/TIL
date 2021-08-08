## Background

I use quite a useful tool called [act](https://github.com/nektos/act) often, which let us run Github Actions locally. 
Which is good since it allows me to be more confident about the workflow files before actually having to run them on Github itself.

But then a few months ago I filed this issue, https://github.com/nektos/act/issues/739. Basically I found a bug that is in my way of using an Action, [yq](https://github.com/marketplace/actions/yq-portable-yaml-processor) that I thought it should be resolved.

After about two months, since it's still not resolved yet. I became curious if I can fix it myself.
So I cloned the repo and tried to run tests and I encountered some non-trivial issue, thus [I filed an Github issue](https://github.com/nektos/act/issues/765).

The details of the issue is in the link above but as a byproduct, I learned about [binfmt_misc](https://en.wikipedia.org/wiki/Binfmt_misc) and hence sharing my experience here.


## binfmt_misc

Obviously I don't know much about it (since I just learned about it) but my current simple understanding is that it allows kernel to treat some binaries differently than usual (usually it will reject to run non-native binaries).

For example, instead of arm64 binaries not working at all on amd64 linux (which is probably most common).

You can adjust `binfmt_misc` to use qemu to run that binary.

And thanks to [this repo](https://github.com/tonistiigi/binfmt/), making that happen is very easy.

_It uses this docker image to modify host's kernel, hence `--privileged` is necessary._

One example with a linux environment (actually with Github Codespace in this case) is below.

```bash
❯ docker run --privileged --rm tonistiigi/binfmt --install arm64 # or choose whatever architecture you wish
installing: arm64 OK
{
  "supported": [
    "linux/amd64",
    "linux/arm64",
    "linux/386"
  ],
  "emulators": [
    "qemu-aarch64"
  ]
}
```

## Playing with it a bit more

Originally, I did this to be able to run different architecture docker images to be able to run tests for `nektos/act` as explained the issue above.

But I also got a hunch that this would work for regular binaries as well not just for docker images.
So I went ahead and tested via these commands below.

```bash
# let's first downlaod and prepare two binaries.
# (with a randomly chosen binary, `kubectl`)

❯ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   154  100   154    0     0   1316      0 --:--:-- --:--:-- --:--:--  1316
100 44.7M  100 44.7M    0     0  86.6M      0 --:--:-- --:--:-- --:--:--  187M

❯ mv kubectl amd-kubectl

❯ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/arm64/kubectl"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   154  100   154    0     0   1305      0 --:--:-- --:--:-- --:--:--  1305
100 41.4M  100 41.4M    0     0  67.1M      0 --:--:-- --:--:-- --:--:--  139M

❯ mv kubectl arm-kubectl

❯ chmod +x amd-kubectl arm-kubectl

# we will run the native architecture (amd64) one first 

❯ ./amd-kubectl

...

Usage:
  kubectl [flags] [options]

Use "kubectl <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands).

# obviously it worked because it's a native architecture binary now let's try arm64 one

❯ ./arm-kubectl
bash: ./arm-kubectl: cannot execute binary file: Exec format error

# doesn't work so let's try `tonistiigi/binfmt` that I mentioned above

❯ docker run --privileged --rm tonistiigi/binfmt --install arm64
installing: arm64 OK
{
  "supported": [
    "linux/amd64",
    "linux/arm64",
    "linux/386"
  ],
  "emulators": [
    "qemu-aarch64"
  ]
}

# now let's try again

❯ ./amd-kubectl

...

Usage:
  kubectl [flags] [options]

Use "kubectl <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands).

# now it works!

# let's measure some performances by the way

❯ time ./amd-kubectl 

real    0m0.034s
user    0m0.027s
sys     0m0.016s

❯ time ./arm-kubectl 

real    0m0.703s
user    0m0.737s
sys     0m0.092s

# as expected the arm one runs (a lot) slower becuase it's being emulated but it's very cool that we can do that though!
```

## How about on my laptop?

So my Github issue closed with a happy ending (at least for me).
I confirmed the tests indeed work, and learned something (cool and) new!

In my Macbook, Docker Desktop actually seems to come with multi architecture support enabled out of box.

```bash
❯ docker run --rm --privileged tonistiigi/binfmt # running without args prints current status of binfmt_misc
{
  "supported": [
    "linux/arm64",
    "linux/amd64",
    "linux/riscv64",
    "linux/ppc64le",
    "linux/s390x",
    "linux/386",
    "linux/mips64le",
    "linux/mips64",
    "linux/arm/v7",
    "linux/arm/v6"
  ],
  "emulators": [
    "qemu-arm",
    "qemu-i386",
    "qemu-mips64",
    "qemu-mips64el",
    "qemu-ppc64le",
    "qemu-riscv64",
    "qemu-s390x",
    "qemu-x86_64"
  ]
}
```

Let's verify this actually work.

I found a very appropriate image for my situation (for M1 (arm64) Macbook).
Enter [docker/whalesay](https://github.com/docker/whalesay), it's even Docker's own!

It looks like an old easter-egg-ish image that using cowsay for Hello World purposes.

And the reason why it's appropriate is that it doesn't have arm64 architecture image since it's old and not updated recently.
That makes an perfect image to test of running non-native architecture image.


```bash
# let's try
❯ docker run --rm docker/whalesay echo "We see a warning above because of we didn't specify platform."
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
We see a warning above because of we didn't specify platform.

# it works out of box! but with a warning

# specifying --platform=linux/amd64 will let the warning be away
❯ docker run --rm --platform=linux/amd64 docker/whalesay uname -a
Linux e87e4c769362 5.10.25-linuxkit #1 SMP PREEMPT Tue Mar 23 09:24:45 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux

# you can see it's printing x86_64

❯ docker run --rm --platform=linux/amd64 docker/whalesay cowsay whallo
 ________
< whallo >
 --------
    \
     \
      \
                    ##        .
              ## ## ##       ==
           ## ## ## ##      ===
       /""""""""""""""""___/ ===
  ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
       \______ o          __/
        \    \        __/
          \____\______/

# and it works fine for its original purpose as well :)
```

## More about binfmt with a focus on Docker
https://medium.com/@tonistiigi/early-look-at-docker-containers-on-risc-v-40ed43b16b09
