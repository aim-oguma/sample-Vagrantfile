# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'net/http'

VAGRANTFILE_API_VERSION = "2"
@vb_gui = false
@vb_memory = 4096
@vb_cpus = 1
@bootstrap = "vagrant-tune"
@vm_name = ARGV[1]
@private_ip = "10.1.1.100"
@public_ip = "172.17.0.8"
@box = "hfm4/centos7"
centos_ver=7
# pubkey auth
pubkey_file = "#{ENV['HOME']}/.ssh/id_rsa.pub"
@local_pubkey = File.read(pubkey_file) if File.exist?(pubkey_file)
CORP_LAN_CHECKER = "<ifconfig.me or Your GIP Checker>(ifconfig.me みたいなGlobal IP Checker を自社で用意しています)" #
CORP_GLOBAL_IP = "<Your company's ip address>"
MYCORP_LOCAL_REPONAME = '<Your company name>'
MYCORP_LOCAL_REPO_URI = "http://<Your local repository>/centos/#{centos_ver}/os/\\$basearch/"
MYCORP_LOCAL_REPO_URI_EPEL = "http://<Your local repository>/epel/#{centos_ver}/"
MYCORP_LOCAL_REPO = "[#{MYCORP_LOCAL_REPONAME}-centos]
name=<Your company name>-CentOS-$releasever - additional
baseurl=#{MYCORP_LOCAL_REPO_URI}
enabled=1
gpgcheck=0

[#{MYCORP_LOCAL_REPONAME}-epel]
name=<Your company name>-epel-$releasever - additional
baseurl=#{MYCORP_LOCAL_REPO_URI_EPEL}
enabled=1
gpgcheck=0
" # priorities は使わない
MYCORP_LOCAL_REPOFILE='/etc/yum.repos.d/mycorp_local.repo'

recommendation="
Recommendation!!
----------------
local の/etc/hostsや ~/.ssh/config を設定しておくと良いです
複数のVagrant vm を立ち上げる, Testで何度も立ち上げるといった場合に良いです

## Examples

```[/etc/hosts]
10.1.1.100 example
```

```[~/.ssh/config]
Host 10.1.1.100 127.0.0.1
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  IdentitiesOnly yes
  LogLevel FATAL
```
"

if @vm_name.nil?
  puts "vagrant <opt> <playbook> を.yml無しで指定してください. # e.g) vagrant provision wordpress"
end

@playbook = "playbooks/#{@vm_name}.yml"
@bootstrap ||= ''

class Network
  def initialize(ipaddr)
    @ipaddr = ipaddr
  end

  def inhouse?
     CORP_GLOBAL_IP == @ipaddr
  end

  class << self
    def global_ipaddr
      http = Net::HTTP.new(CORP_LAN_CHECKER, 80)
      http.start {
        response = http.get("/")
        get_ipaddr = response.body
        return Network.new(get_ipaddr)
      }
    end
  end
end

class ReplaceText
  def initialize(
    file_name=String.new,
    pattern=String.new,
    replacement=String.new
    )
    @file_name   = file_name
    @pattern     = /#{pattern}/
    @replacement = replacement
  end

  def replace_files
    @file_name.each_line { |l| exec(l) }
  end

  private

  def exec(path)
    # 読み取り専用でファイルを開き、内容をbufferに代入
    buffer = File.open(path, "r") { |f| f.read() }

    # バックアップ用ファイルを開いて、バッファを書き込む（バックアップ作成）
    File.open("#{path}.bak" , "w") { |f| f.write(buffer) }

    # bufferの中身を変換
    buffer.gsub!(@pattern, @replacement)

    # bufferを元のファイルに書き込む
    File.open(path, "w") { |f| f.write(buffer) }
  end
end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  # 社内の場合はLocal repositoryに向けて高速化する
  if Network.global_ipaddr.inhouse?
    config.vm.provision :shell, :inline => <<-EOT
      sudo sh -c "echo '#{MYCORP_LOCAL_REPO}' > #{MYCORP_LOCAL_REPOFILE}"
      sudo yum clean all
      sudo yum repolist
    EOT
  else
    config.vm.provision :shell, :inline => <<-EOT
      if [ -f #{MYCORP_LOCAL_REPOFILE} ];then
        rm -f #{MYCORP_LOCAL_REPOFILE}
        sudo yum clean all
        sudo yum repolist
      fi
    EOT
  end

  config.vm.provider :virtualbox do |vb|
    vb.gui = @vb_gui
    vb.memory = @vb_memory
    vb.cpus = @vb_cpus
    vb.customize ["modifyvm", :id, "--nictype1", "virtio"]
    vb.customize ["modifyvm", :id, "--nictype2", "virtio"]
  end

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  system("vagrant plugin install vagrant-cachier") unless Vagrant.has_plugin?("vagrant-cachier")

  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :box
  end

  config.vm.define @vm_name do |vagrant|
    vagrant.vm.box = @box

    vagrant.vm.network :private_network, ip: @private_ip
    vagrant.ssh.forward_agent = true

    if File.exist?(@bootstrap)
      vagrant.vm.provision :shell, path: @bootstrap
    end

    def func_add_for_wp
      # 初期化用
      file_name = "#{ENV['HOME']}/.ssh/config"
      pattern = "Host wordpress\n  User vagrant"
      replacement = ''

      case yield
      when 'halt', 'destroy'
        # file_name の初期化を行う
        r = ReplaceText.new(file_name, pattern, replacement)
        r.replace_files
      else
        # TDI (Serverspec等)のためにssh wordpress用設定をする (vagrantで接続するように設定する)
        file = File.open(file_name)
        unless file.read.match(pattern)
          File.open(file_name, 'a') do |io|
            io.puts(pattern)
            io.close
          end
        end
      end
    end

    vagrant.vm.provision :shell, :inline => <<-EOT
      echo #{@local_pubkey} >> /home/vagrant/.ssh/authorized_keys
    EOT

    vagrant.vm.provision "ansible" do |ansible|
      ansible.playbook = @playbook
      ansible.limit = 'all'
      ansible.verbose = 'vvvv'
      ansible.raw_ssh_args = "-o ConnectTimeout=120s -o ControlPersist=150s"

      # Not interactive.
      # TARGET_ENV=production で "Not found roles/php/vars/{{ pjcode }}.yml. Do you use standard production parameters? [from roles/php/vars/prodcution.yml] (yes/no)"
      # と聞かれずにproduction.yml を読み込んでくれる
      # Vagrantだと対話でコケるし、testで回しまくりたいのでスルーさせる
      if ENV['TARGET_ENV'] === "production"
        ansible.extra_vars = {
          # for production test
          standard_product_auto: 'yes'
        }
      end

      # Ansible のgroup指定
      case @vm_name
      when 'wordpress'
        @group1 = "vagrant"
        @group2 = "os_CentOS"

        # wordpressの場合は、vagrant コマンドの引数を見て処理を追加する (ex: vagrant halt )
        func_add_for_wp { ARGV[0] }
      when 'vuls'  # vulsの場合(vagrant [up|provision] vuls) ならslack通知のために--extra-varsを指定する
        ansible.extra_vars = {
          slack_webhook_url: "#{ENV['SLACK_HOOKURI']}",
          channel: "#{ENV['CHANNEL'] || '#vuls'}"
        }
      when 'test'
        @group1 = "test"
      else
        warn("[Message by Ansible provision] Unknown vm_name")
        warn("[Message by Ansible provision] Unset the Ansible groups.")
        puts
      end

      # group制御部
      if @group1.nil? and @group2.nil?
        # group指定がなければ何もしない
      elsif @group2.nil?
        ansible.groups = {
          @group1 => @vm_name
        }
      else # group1とgroup2に指定がある場合
        ansible.groups = {
          @group1 => @vm_name,
          "#{@group2}:children" => [@group1]
        }
      end

      # 以下は使いたい時に
      #ansible.tags = ['init', 'php', 'wordpress', 'test']
      #ansible.skip_tags = ['common', 'nginx', 'db', 'newrelic', 'postfix']
      #ansible.vault_password_file = ENV['HOME'] + "/.ansible/vault-password"
      # vagrant だし、vaultは普段無しにしておく(secret fileにあるpwは本番用と全て別なので安全)
    end
  end
end

# ssh/config に設定が無い場合は推奨設定メッセージを出す
f = File.read(ENV['HOME']+'/.ssh/config')
(sleep 3; puts recommendation ) unless f.match("^Host.*127.0.0.1") && f.match("UserKnownHostsFile.*/dev/null")

