require 'rubygems'
require 'bundler/setup'

distro = nil
fpm_opts = ""

if File.exist?('/etc/system-release') && File.read('/etc/redhat-release') =~ /centos|redhat|fedora|amazon/i
  distro = 'rpm'
  fpm_opts << " --rpm-user root --rpm-group root "
elsif File.exist?('/etc/os-release') && File.read('/etc/os-release') =~ /ubuntu|debian/i
  distro = 'deb'
  fpm_opts << " --deb-user root --deb-group root "
  fpm_opts << " --depends google-chrome-stable "
end

unless distro
  $stderr.puts "Don't know what distro I'm running on -- not sure if I can build!"
end

versions = %w(2.10 2.11 2.12 2.13 2.14)
release = Time.now.utc.strftime('%Y%m%d%H%M%S')

task :clean do
  rm_rf "pkg"
  rm_rf "downloads"
end

namespace "wrapper" do
  task :clean do
    rm_rf "jailed-root"
  end

  task :init do
    mkdir_p  "jailed-root/usr/local/bin"
    mkdir_p "pkg"
  end

  desc "build the chromedriver wrapper package"
  task :build => [:clean, :init] do

    cd "jailed-root/usr/local/bin" do
      File.open('chromedriver', 'w') do |f|
        f.write(<<-BASH)
#!/bin/bash

DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"
export LD_LIBRARY_PATH=/opt/google/chrome/lib:$LD_LIBRARY_PATH
unset RUBYOPT BUNDLE_GEMFILE RUBYLIB BUNDLE_BIN_PATH GEM_HOME GEM_PATH

# allow users to pick up a chromedriver version by specifying an environment variable
CHROMEDRIVER_VERSION=${CHROMEDRIVER_VERSION:-2.10}

exec ${DIR}/chromedriver-original-${CHROMEDRIVER_VERSION} "$@"

BASH
      end
      sh('chmod 755 chromedriver')
    end

  end
end

versions.each do |version|
  namespace version do
    package_name = "google-chrome-driver-#{version}"

    task :clean do
      rm_rf "jailed-root"
    end

    task :init do
      mkdir_p  "jailed-root/usr/local/bin"
      mkdir_p "pkg"
      mkdir_p "downloads"
    end

    desc "build chromedriver #{version} #{distro}"
    task :build => [:clean, :init] do

      cd "downloads" do
        sh("curl --silent --fail --location https://chromedriver.storage.googleapis.com/#{version}/chromedriver_linux64.zip > chromedriver_linux64.zip")
      end

      cd "jailed-root/usr/local/bin" do
        sh('unzip -q ../../../../downloads/chromedriver_linux64.zip')
        sh("mv chromedriver chromedriver-original-#{version}")
        sh("chmod 755 chromedriver-original-#{version}")
      end

      upstream_version = %x[(
        export LD_LIBRARY_PATH=/opt/google/chrome/lib:$LD_LIBRARY_PATH
        unset RUBYOPT BUNDLE_GEMFILE RUBYLIB BUNDLE_BIN_PATH GEM_HOME GEM_PATH
        timeout 30 jailed-root/usr/local/bin/chromedriver-original-#{version} --version
        )].match(%r{(#{version}\.\d+)})[1]

      raise 'could not determine version' if upstream_version.nil? || upstream_version.empty?

      cd "pkg" do
        sh(%Q{
             bundle exec fpm -s dir -t #{distro} --name #{package_name} -a x86_64 --version "#{upstream_version}" --iteration #{release} -C ../jailed-root --verbose #{fpm_opts} --depends chromedriver-wrapper --maintainer support@snap-ci.com --vendor support@snap-ci.com --url http://snap-ci.com --description "Chromedriver binary" .
        })
      end
    end
  end

  task :default => [:clean, "#{version}:build", "wrapper:build"]
end
