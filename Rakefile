# Copyright (C) 2009-2013 MongoDB Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

require "bundler"
Bundler.setup

$LOAD_PATH.unshift(File.expand_path("../lib", __FILE__))

require "rake"
require "rake/extensiontask"
require "rspec/core/rake_task"

def jruby?
  defined?(JRUBY_VERSION)
end

if jruby?
  require "rake/javaextensiontask"
  Rake::JavaExtensionTask.new do |ext|
    ext.name = "native"
    ext.ext_dir = "src"
    ext.lib_dir = "lib/mongo/auth/kerberos"
    ext.release = ENV['JAVA_RELEASE'].to_i if ENV['JAVA_RELEASE']
  end
else
  require "rake/extensiontask"
  Rake::ExtensionTask.new do |ext|
    ext.name = "mongo_kerberos_native"
    ext.ext_dir = "ext/mongo_kerberos"
    ext.lib_dir = "lib"
  end
end

desc "[INTERNAL] Loads the library's version"
task :load_version do
  require 'mongo/auth/kerberos/version'
end

desc 'Print the current version (used for releases)'
task version: :load_version do
  puts Mongo::Auth::Kerberos::VERSION
end

RSpec::Core::RakeTask.new(:rspec)

# `rake version` is used by the deployment system so get the release version
# of the product beng deployed. It must do nothing more than just print the
# product version number.
desc 'Print the current version'
task :build => [ :clean_all, *(jruby? ? :compile : nil) ] do
  output = "--output=#{ENV['GEM_FILE_NAME']}" if ENV['GEM_FILE_NAME']
  system "gem build #{output} mongo_kerberos.gemspec"
end

# `rake gem_file_name` is used by the deployment system so get the name of
# the gem file to be generated. It must do nothing more than just print the
# name of the gem file to generate.
desc 'Print the name of the gem file to generate.'
task gem_file_name: :load_version do
  base = "mongo_kerberos-#{Mongo::Auth::Kerberos::VERSION}"
  base << '-java' if jruby?
  puts "#{base}.gem"
end

# overrides the default Bundler-provided `release` task, which also
# builds the gems. Our release process assumes the gems have already
# been built (and signed via GPG), so we just need `rake release` to
# push the gems to rubygems.
desc 'Push the generated gems to RubyGems'
task :release do
  # confirm: there ought to be two gems, one for MRI, and one for Java. These
  # will have been previously generated by the 'Release' GitHub action.
  gems = Dir['*.gem']
  if gems.length != 2
    abort "Expected two gem files to be ready to release; got #{gems.length}"
  end

  if ENV['GITHUB_ACTION'].nil?
    abort <<~WARNING
      `rake release` must be invoked from the `Release` GitHub action,
      and must not be invoked locally. This ensures the gem is properly signed
      and distributed by the appropriate user.

      Note that it is the `rubygems/release-gem@v1` step in the `Release`
      action that invokes this task. Do not rename or remove this task, or the
      release-gem step will fail. Reimplement this task with caution.

      NO GEMS were pushed to RubyGems.
    WARNING
  end

  gems.each do |gem|
    system 'gem', 'push', gem
  end
end

task :clean_all => :clean do
  begin
    Dir.chdir(Pathname(__FILE__).dirname + "lib") do
      %w[ o bundle so jar ].each do |e|
         Dir.glob(File.join("**", "*.#{e}")).each do |f|
          `rm #{f}`
        end
      end
    end
  rescue Exception => e
    puts e.message
  end
end

task :spec => :compile do
  Rake::Task["rspec"].invoke
end

task :default => [ :clean_all, :spec ]

desc "Generate all documentation"
task :docs => 'docs:yard'

namespace :docs do
  desc "Generate yard documention"
  task yard: :load_version do
    out = File.join('yard-docs', Mongo::Auth::Kerberos::VERSION)
    FileUtils.rm_rf(out)
    system "yardoc -o #{out} --title mongo-ruby-kerberos-#{Mongo::Auth::Kerberos::VERSION}"
  end
end
