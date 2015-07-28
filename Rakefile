require 'fileutils'

# VARIABLES
# These are common string variables that are used by the build tasks.

ENV['version']            ||= 'dev'
ENV['offline_repos_path'] ||= '/opt/puppetlabs/repos'
ENV['repos_name']         ||= "seteam-production-repos-#{ENV['version']}"
ENV['environment_name']   ||= "seteam-production-#{ENV['version']}"
ENV['platform_tar_flags'] ||= %x{uname} == 'Darwin' ? '--disable-copyfile' : ''

# TASKS
# These tasks result in actions or artifacts
#

task :default => [:repos_tarball, :environment_tarball]

desc "Create a tarball containing a self-contained version of all repositories needed to run r10k"
task :repos_tarball => "build/#{ENV['repos_name']}.tar.gz" do
  puts "repos_tarball task complete"
end

desc "Create a tarball containing an extracted environment (as would be built by r10k)"
task :environment_tarball => "build/#{ENV['environment_name']}.tar.gz" do
  puts "environment_tarball task complete"
end

desc "Clean up and remove the build directory"
task :clean do
  rm_rf 'build'
end

# FUNCTIONS
# Helper routines to capture logic that is required by more than one task
#

def all_files_in_git
  @all_files_in_git ||= %x{git ls-files}.split("\n")
end

def tar_transform_flags(from, to)
  case %x{uname}.chomp
  when 'Darwin'
    "-s /#{from}/#{to}/"
  else
    "--transform='s/#{from}/#{to}/'"
  end
end

def git_clone_bare(from, to)
  sh "git clone --bare --no-hardlinks '#{from}' '#{to}'"
  sh "GIT_DIR='#{to}' git repack -a"
  rm_f(File.join(to, 'objects', 'info', 'alternates'))
end

# FILES
# These define the files Rake tracks and builds
#

file 'build/repos' => ['build/environment', 'build/puppet-control'] do
  rm_rf 'build/repos'
  mkdir_p 'build/repos'

  Rake::FileList['build/environment/modules/*'].each do |mod_dir|
    mod_name   = File.basename(mod_dir)
    dotgit_dir = "#{mod_dir}/.git"
    repo_dir   = "build/repos/#{mod_name}.git"

    git_clone_bare(dotgit_dir, repo_dir)
  end

  control_dotgit_dir = 'build/puppet-control/.git'
  control_repo_dir   = 'build/repos/puppet-control.git'
  git_clone_bare(control_dotgit_dir, control_repo_dir)
end

file 'build/Puppetfile' => ['build/environment'] do
  File.open('build/Puppetfile', 'w') do |file|
    file.write("forge 'forgeapi.puppetlabs.com'\n\n")

    Rake::FileList["#{'build/environment/modules'}/*"].each do |mod_dir|
      name = File.basename(mod_dir)
      head = File.read(File.join(mod_dir, '.git', 'HEAD')).chomp
      ref  = head.match(%r{^ref: }) ? head.sub(%r{^ref: refs/heads/}, '') : head

      file.write("mod '#{name}',\n")
      file.write("  :git => '#{ENV['offline_repos_path']}/#{name}.git',\n")
      file.write("  :ref => '#{ref}'\n\n")
    end
  end
end

file 'build/puppet-control' => ['build/Puppetfile'] do
  rm_rf 'build/puppet-control'
  mkdir 'build/puppet-control'
  Dir.chdir('build/puppet-control') do
    sh "git clone ../../.git ."
    sh 'git branch -m production 2>/dev/null || git checkout -b production'
    cp '../Puppetfile', 'Puppetfile'
    sh 'git add Puppetfile'
    sh 'git commit -m "Lab environment control repo initialized"'
  end
end

file 'build/environment' => all_files_in_git do
  rm_rf 'build/environment'
  mkdir_p 'build/environment'

  # Recursively copy in all top-level files or directories
  all_files_in_git.map { |f| f.split('/').first }.uniq.each do |file|
    cp_r file, 'build/environment', :verbose => true
  end

  Dir.chdir('build/environment') do
    sh "r10k puppetfile install -v"
  end

  Dir.entries('build/environment/modules').reject{|e| e =~ /^\./}.each do |mod|
    unless File.exist?("build/environment/modules/#{mod}/.git")
      Dir.chdir("build/environment/modules/#{mod}") do
        sh 'git init .'
        sh 'git add -f *'
        sh 'git commit -m "create new repo from snapshot"'
      end
    end
  end
end

file "build/#{ENV['environment_name']}.tar.gz" => ['build/environment'] do
  environment_dir_name = File.basename('build/environment')

  Dir.chdir('build') do
    tarflags = [
      ENV['platform_tar_flags'],
      tar_transform_flags(environment_dir_name, ENV['environment_name']),
      '--exclude .git',
      '--exclude .gitignore',
      "-cvzf #{ENV['environment_name']}.tar.gz",
      environment_dir_name
    ]

    sh "tar #{tarflags.join(' ')}"
  end
end

file "build/#{ENV['repos_name']}.tar.gz" => ['build/puppet-control', 'build/repos'] do
  repos_dir_name = File.basename('build/repos')

  Dir.chdir('build') do
    tarflags = [
      ENV['platform_tar_flags'],
      tar_transform_flags(repos_dir_name, ENV['repos_name']),
      "-cvzf #{ENV['repos_name']}.tar.gz",
      repos_dir_name
    ]

    sh "tar #{tarflags.join(' ')}"
  end
end

