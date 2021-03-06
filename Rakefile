require 'rubygems'
require 'bundler/setup'

require 'rake/clean'

distro = nil
fpm_opts = ""

if File.exist?('/etc/system-release') && File.read('/etc/redhat-release') =~ /centos|redhat|fedora|amazon/i
  distro = 'rpm'
  fpm_opts << " --rpm-user root --rpm-group root "
elsif File.exist?('/etc/os-release') && File.read('/etc/os-release') =~ /ubuntu|debian/i
  distro = 'deb'
  fpm_opts << " --deb-user root --deb-group root "
end

unless distro
  $stderr.puts "Don't know what distro I'm running on -- not sure if I can build!"
end

version = "1.8.4.1"
release = ENV['GO_PIPELINE_COUNTER'] || ENV['RELEASE'] || 1
name = "git-#{version}"

description_string = %Q{Git is a fast, scalable, distributed revision control system with an unusually rich command set that provides both high-level operations and full access to internals.}

jailed_root = File.expand_path('../jailed-root', __FILE__)

CLEAN.include("downloads")
CLEAN.include("jailed-root")
CLEAN.include("log")
CLEAN.include("pkg")
CLEAN.include("src")

task :init do
  mkdir_p "log"
  mkdir_p "pkg"
  mkdir_p "src"
  mkdir_p "downloads"
  mkdir_p "jailed-root"
end

task :download do
  cd 'downloads' do
    sh("curl --fail http://git-core.googlecode.com/files/git-#{version}.tar.gz             > git-#{version}.tar.gz 2>/dev/null")

    sh("echo '49004a8dfcbb7c0848147737d9877fd7313a42ec  git-#{version}.tar.gz'             > git-#{version}.tar.gz.shasum 2>/dev/null")

    sh("sha1sum --status --check git-#{version}.tar.gz.shasum")
  end
end

task :configure do
  cd "src" do
    sh "tar -zxf ../downloads/git-#{version}.tar.gz"
    cd "git-#{version}" do
      sh "./configure --prefix=/opt/local/git/#{version} > #{File.dirname(__FILE__)}/log/configure.#{version}.log 2>&1"
    end
  end
end

task :make do
  num_processors = %x[nproc].chomp.to_i
  num_jobs       = num_processors + 1

  cd "src/git-#{version}" do
    sh("make -j#{num_jobs} > #{File.dirname(__FILE__)}/log/make.#{version}.log 2>&1")
  end
end

task :make_install do
  rm_rf  jailed_root
  mkdir_p jailed_root
  cd "src/git-#{version}" do
    sh("make install DESTDIR=#{jailed_root} > #{File.dirname(__FILE__)}/log/make-install.#{version}.log 2>&1")
  end
end


task :dist do
  require 'erb'
  class RpmSpec
    attr_accessor :version, :release
    def initialize(version, release)
      @version      = version
      @release      = release
    end

    def get_binding
      binding
    end
  end

  ERB.new(File.read(File.expand_path('../git.spec.erb', __FILE__)), nil , '-').tap do |template|
    File.open("/tmp/git.spec", 'w') do |f|
      attrs = RpmSpec.new(version, release)
      f.puts(template.result(attrs.get_binding))
    end
    at_exit {rm_rf "/tmp/git.spec"}
  end

  mkdir_p "#{jailed_root}/usr/local/bin"

  cd "#{jailed_root}/usr/local/bin" do
    Dir["../../../opt/local/git/#{version}/bin/*"].each do |bin_file|
      ln_sf bin_file, File.basename(bin_file)
    end
  end

  if distro == 'rpm'
    output_dir = File.expand_path('../target-rpms', __FILE__)
    at_exit {rm_rf output_dir}
    cd jailed_root do
      puts "*** Building RPM..."
      rpmbuild_cmd = []
      rpmbuild_cmd << "rpmbuild /tmp/git.spec"
      rpmbuild_cmd << "--verbose"
      rpmbuild_cmd << "--buildroot #{jailed_root}"
      rpmbuild_cmd << "--define '_tmppath #{jailed_root}/../rpm_tmppath'"
      rpmbuild_cmd << "--define '_topdir #{output_dir}'"
      rpmbuild_cmd << "--define '_rpmdir #{output_dir}'"
      rpmbuild_cmd << "-bb"

      sh rpmbuild_cmd.join(" ")
    end
    sh("mv target-rpms/x86_64/*.rpm pkg/")
  else
    cd "pkg" do
      sh(%Q{
           bundle exec fpm -s dir -t #{distro} --name git-#{version} -a x86_64 --version "#{version}" -C #{jailed_root} --verbose #{fpm_opts} --maintainer snap-ci@thoughtworks.com --vendor snap-ci@thoughtworks.com --url http://snap-ci.com --description "#{description_string}" --iteration #{release} --license 'GPLv2' .
      })
    end
  end
end

desc "deploy the rpm to s3"
task :deploy do
  File.open('s3.cfg', 'w') do |f|
    f.puts "[default]"
    f.puts "access_key = #{ENV['S3_ACCESS_KEY']}"
    f.puts "secret_key = #{ENV['S3_SECRET_KEY']}"
  end

  repo_dir = if ENV['GO_SERVER_URL'] || ENV['CI']
    File.expand_path('~/.git-build-repo')
  else
    rm_rf   repo_dir
    File.expand_path('../repo', __FILE__)
  end

  mkdir_p repo_dir

  oldpwd = Dir.pwd
  cd repo_dir do
    sh("s3cmd mb --config #{oldpwd}/s3.cfg s3://#{ENV['S3_BUCKET']}")
    sh("s3cmd sync --config #{oldpwd}/s3.cfg --verbose --acl-public --delete-removed --no-preserve s3://#{ENV['S3_BUCKET']} .")
    sh("cp #{oldpwd}/pkg/*.rpm ./rpms/")

    sh("createrepo --database --update . || createrepo --database .")
    sh("s3cmd sync --config #{oldpwd}/s3.cfg --verbose --acl-public --delete-removed --no-preserve . s3://#{ENV['S3_BUCKET']}")
  end
end

desc "build git rpm"
task :default => [:clean, :init, :download, :configure, :make, :make_install, :dist]
