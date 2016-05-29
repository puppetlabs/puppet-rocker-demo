# Building Docker Images with Puppet

The flexibility of Dockerfile means that it's long been possible to
build Docker images using Puppet. This has several advantages in some
situations:

* Puppet provides a much richer set of high-level types than Dockerfile
  where you are reliant on bash and `RUN`
* The capability to easy integrate data from different sources to affect
  the resulting images
* The ability to reuse code between containers and more traditional
  infrastructure
* A programming language capable of abstraction and with a strong suite
  of testing tools available

The approach of running a `puppet apply` command in a Dockerfile was
ably described in [this blog post](https://puppet.com/blog/building-puppet-based-applications-inside-docker)
which is now more than 2 years old.

However the approach described has several problems:

* Software only used for building the image is left in at runtime, which
  increases the surface area for an attacker
* Because of all that extra software the resulting image is much larger
  than it needs to be


## A Modern Alternative

[Rocker](https://github.com/grammarly/rocker) is an alternative Docker
build tool with some additions to the standard Dockerfile syntax. In
particular Rocker supports mounting volumes at build time. These volumes
are only available during build and do not become part of the resulting
image. With that capability we can build Docker images using Puppet
without the need to carry around Puppet at runtime.

First install
[Rocker](https://github.com/grammarly/rocker#installation).

Similar to the original example we're going to use a `Puppetfile` to
describe out dependencies:

```
forge 'https://forgeapi.puppetlabs.com'

mod 'jfryman/nginx'
mod 'puppetlabs/stdlib'
mod 'puppetlabs/concat'
mod 'puppetlabs/apt'
```

Rather than simply inline the actual Puppet code as in the original
example we're storing it in the accompanying manifests directory. The
code for our demonstration is saved as `manifests/init.pp`.

```puppet
class { 'nginx': } ->
exec { 'Disable Nginx daemon mode':
  path    => '/bin',
  command => 'echo "daemon off;" >> /etc/nginx/nginx.conf',
  unless  => 'grep "daemon off" /etc/nginx/nginx.conf',
}
```

With Rocker installed lets now look at the `Rockerfile`.

```
FROM ubuntu:16.04
MAINTAINER Gareth Rushgrove "gareth@puppet.com"

ENV PUPPET_AGENT_VERSION="1.5.0" UBUNTU_CODENAME="xenial" PATH=/opt/puppetlabs/server/bin:/opt/puppetlabs/puppet/bin:/opt/puppetlabs/bin:$PATH

LABEL com.puppet.version="0.1.0" com.puppet.dockerfile="/Dockerfile" com.puppet.puppetfile="/Puppetfile" com.puppet.manifest="/manifests.init.pp"

MOUNT /opt/puppetlabs /etc/puppetlabs /root/.gem

RUN apt-get update && \
    apt-get install -y wget=1.17.1-1ubuntu1 && \
    wget https://apt.puppetlabs.com/puppetlabs-release-pc1-"$UBUNTU_CODENAME".deb && \
    dpkg -i puppetlabs-release-pc1-"$UBUNTU_CODENAME".deb && \
    rm puppetlabs-release-pc1-"$UBUNTU_CODENAME".deb && \
    apt-get update && \
    apt-get install --no-install-recommends -y puppet-agent="$PUPPET_AGENT_VERSION"-1"$UBUNTU_CODENAME" && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN /opt/puppetlabs/puppet/bin/gem install r10k:2.2.2 --no-ri --no-rdoc

COPY Puppetfile /
COPY manifests /manifests
RUN apt-get update && \
    r10k puppetfile install --moduledir /etc/puppetlabs/code/modules && \
    puppet apply manifests/init.pp --verbose --show_diff  --summarize && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

EXPOSE 80

CMD ["nginx"]

COPY Rockerfile /Dockerfile

TAG puppet/puppet-rocker-demo
```

As mentioned, a `Rockerfile` is simply a `Dockerfile` with some new
instructions. Of particular note is the `MOUNT` operation. This
invocation mounts this directories from ephemeral containers during
build, meaning the contents are available at build time but not saved in
the resulting image.

```
MOUNT /opt/puppetlabs /etc/puppetlabs /root/.gem
```

This means we have access to the full set of Puppet and other tools for
building our image, without needing them at runtime.

Their are some other tricks in the Rockerfile which mirror current
Dockerfile best practice, for example cleaning up the package cache and
metadata to save space.

If you'd like to try this demo out you can simply run the following from
this directory:

```
rocker build
```

This should build the image from scratch, providing useful output
regarding the different layer sizes as it goes.


## Results

So, how successful is the experiment? Let's start by looking at the base
images. We've used Ubuntu for both the examples for ease of comparison,
although it's worth noting that the size of the base images has been
decreasing too. So we save around 50 MB by just updating to the latest release.

```
$ docker images | grep ubuntu
ubuntu  16.04  0b7735b9290f  7 weeks ago    123.7 MB
ubuntu  12.10  3e314f95dcac  23 months ago  172.1 MB
```

The original example left all the required build tools in the image. For
the nginx example this meant a whopping 400 MB+ image.

```
$ docker images | grep jamtur
jamtur01/puppetbase  latest  1842a4dc30c4  25 seconds ago  361.4 MB
jamtur01/nginx       latest  6c66777eb97f  9 seconds ago   408.2 MB
```

With the approach shown here we have reduced that size down to 185 MB.
That's a 55% saving on the original. The real improvement is much
larger though. The new image is only 62 MB larger than the base image compared
with the originals 236 MB.

```
$ docker images | grep rocker
puppet/puppet-rocker-demo  latest  cb051d97e72e  10 hours ago  185.5 MB
```
