# Test Plans

## umask and /tmp

Companies often have security requirements (as per STIG or other standards) to make `/tmp` mounted with several security options and to have the `umask` set to a restrictive value.  In this past this has caused many issues with install PE.

This test is to determine if PE can be installed and upgrade with a more restrictive umask and the tmp directory with restrictive attributes.

1. Bring up a EL6/7 server
2. Change global umask (/etc/bashrc) to 027

  ```shell
  echo '027' >> /etc/bashrc
  ```

3. Edit /etc/fstab to bind /tmp on /var/tmp (/tmp /var/tmp none rw,noexec,nosuid,nodev,bind 0 0)

  ```shell
  echo '/tmp  /var/tmp  none  rw,noexec,nosuid,nodev,bind  0  0' >> /etc/fstab
  ```

4. Reboot

  ```shell
  reboot
  ```

5. Login and check the umask for root is `027`

  ```shell
  umask
  ```

6. Install PE with Code Manager enabled (any control repo will do)
7. Ensure Code Manager is configured correctly
8. Run Code Manager, check environment is correct after run
9. Run Puppet

  ```shell
  puppet agent -t
  ```

10. Do a `mco ping` and check correct

  ```shell
  su - peadmin
  mco ping
  ```

## hiera-eyaml

Most companies use `hiera-eyaml` to protect data within Hiera.

This test aims to prove `hiera-eyaml` works on the CLI and with puppetserver.

1.  Install PE
2. Login to Master
3. Run

  ```shell
    /opt/puppetlabs/puppet/bin/gem install hiera-eyaml
    /opt/puppetlabs/bin/puppetserver gem install hiera-eyaml
  ```

4. Create a set of `eyaml` keys

  ```shell
    /opt/puppetlabs/puppet/eyaml createkeys
  ```

5. Copy keys to a directory where pe-puppet can access

  ```shell
    mkdir -p /etc/puppetlabs/puppet/ssl/eyaml
    cp keys/* /etc/puppetlabs/puppet/ssl/eyaml
    chown -R pe-puppet:pe-puppet /etc/puppetlabs/puppet/ssl/eyaml
  ```

6. Modify Hirea configuration to use eyaml

  ```shell
    mkdir /etc/puppetlabs/code/hieradata
    echo '---' >> /etc/puppetlabs/code/hieradata/common.yaml
    cat << EOF > /etc/puppetlabs/code/hiera.yaml
    :backends:
      - eyaml
    :hierarachy:
      - "nodes/%{::trusted.certname}"
      - common
    :eyaml:
      :datadir: "/etc/puppetlabs/code/hieradata"
      :extension: "eyaml"
      :pkcs7_private_key: /etc/puppetlabs/puppet/ssl/eyaml/private_key.pkcs7.pem
      :pkcs7_public_key:  /etc/puppetlabs/puppet/ssl/eyaml/public_key.pkcs7.pem
    EOF
  ```

7. Restart Puppet server

  ```shell
    service pe-puppetserver restart
  ```

8. Create some eyaml

  ```shell
    /opt/puppetlabs/puppet/bin/eyaml encrypt -s 'testing' -l mytest >> /etc/puppetlabs/code/hieradata/common.yaml
  ```

9. Determine module path and create a module and class

  ```shell
    puppet master --configprint environmentpath | xargs -I % mkdir -p %/production/modules/test/manifests
    cat << EOF > /etc/puppetlabs/code/environments/production/modules/test/manifests/init.pp
    class test {
      $data = hiera('mytest')
      notify { $data: }
    }

    EOF
  ```

10. Classify the node to use this class:

  ```shell
    CONFDIR='/etc/puppetlabs/puppet'
    CERT=$(puppet master   --confdir ${CONFDIR} --configprint hostcert)
    CACERT=$(puppet master --confdir ${CONFDIR} --configprint localcacert)
    PRVKEY=$(puppet master --confdir ${CONFDIR} --configprint hostprivkey)
    OPTIONS="--cert ${CERT} --cacert ${CACERT} --key ${PRVKEY}"
    CONSOLE=$(awk '/server: /{print $NF}' ${CONFDIR}/classifier.yaml)
    curl -X POST -H "Content-Type: application/json"  ${OPTIONS} "https://${CONSOLE}:4433/classifier-api/v1/groups"  --data '{"name":"PE Test","parent":"00000000-0000-4000-8000-000000000000","classes":{"test":{}},"rule":["and",["~","name",".*"]]}'
  ```

11. Run puppet

  ```shell
    puppet agent -t
  ```

12.  Determine if notify is correct.

## Proxies

Many companies disallow direct, or any, access to the Internet from the Puppet Infrastructure nodes.  Most of the companies use proxies to intercept the traffic.  This often leads to many issues with installation and operation of PE as new company-derived certificates need to be in place to operate with the proxies.

This test sets up a proxy for HTTP and HTTPS.  The HTTPS requires a different CA certificate so the proxy can intercept the traffic.

See the associated Vagrant environment at [Vagrant_Env](https://github.com/beergeek/proxy_env.git)

### Environment variable proxy

This test creates a proxy server plus a Puppet master.  The Puppet master will have the following as environment variables:

  ```shell
  http_proxy=10.20.1.127:3127
  https_proxy=10.20.1.127:3127
  ```

  This will set all HTTP and HTTPS traffic to be sent to the proxy, with the exception of the Puppet master's traffic.

1. Clone the Vagrant environment

  ```shell
  git clone https://github.com/beergeek/test_vagrant.git
  ```

2. Bring up the environment.  (This will great a proxy node, plus two Puppet masters so you need some memory for this test)

  ```shell
  cd proxy_env
  vagrant up
  ```

3. Once the environment is up, log into the first master.

  ```shell
  vagrant ssh master0.puppetlabs.vm
  ```

4. Attempt to run puppet

  ```shell
  puppet agent -t
  ```

5. Attempt to run Code Manager. Use [CM_FS](https://docs.puppetlabs.com/pe/latest/code_mgr_scripts.html) for details.  Trigger refresh in NC.  Determine if the number of classes is correct compared to actual classes (check `/var/log/puppetlabs/console-services/console-services.log`).  As we are behind a proxy we will need to edit the `/etc/puppetlabs/puppetserver/conf.d/code-manager.conf` file to enable `shellgit`, this is not documented for some crazy reason. `git-provider : "shellgit"`.  Once this is all completed run code manager.
6. Attempt to install `hiera-eyaml`

  ```shell
  /opt/puppetlabs/puppet/bin/gem install hiera-eyaml
  /opt/puppetlabs/bin/puppetserver gem install hiera-eyaml
  ```
7. Attempt to run Code Manager again.

## NC Refresh

In the past the refresh of environments, classes and parameters via the NC has been somewhat problematic.  Changes to this function require testing in several scenarios (via user interaction and via API usage).

## Console Features
